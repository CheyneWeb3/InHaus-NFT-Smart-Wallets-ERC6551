# Deeply Nested ERC-6551 Accounts: Using `executeNested` and `executeBatch`

Token Bound Accounts (TBAs) per ERC-6551 allow NFTs to own assets and even other NFTs. When NFTs are nested (an NFT’s TBA owns another NFT that itself has a TBA, and so on), special handling is needed to execute transactions from the deeply nested account. The account implementation at **0x41C8...4eC** is the Tokenbound reference contract (deployed on Ethereum mainnet and testnets ([ERC 6551 Overview: What new use cases will token-bound accounts bring to Lens? | 律动BlockBeats on Binance Square](https://www.binance.com/en/square/post/1584000#:~:text=Tokenbound%20account%20implementation%20address%3A%200x41C8f39463A868d3A88af00cd0fe7102F30E44eC))). Below, we’ll walk through how to use its functions **`executeNested`** and **`executeBatch`** to perform transactions with nested TBAs, including transferring funds/assets between them. We will cover constructing the call data, building and supplying the proof of ownership chain, ABI considerations, and the effects of nesting on execution.

## Understanding Nested ERC-6551 Token-Bound Accounts

In ERC-6551, each ERC-721 token can have an account (a smart contract wallet) bound to it. A **nested TBA** scenario is when an NFT’s account owns another NFT (making that NFT *child* to the first), which may itself have an account, and so on. For example, imagine NFT **A** (top-level) has an account **Account_A**. Account_A owns another NFT **B**, which has its own account **Account_B**. Account_B then owns NFT **C** with account **Account_C**. Here Account_C is nested two levels deep (A → B → C).

One challenge with nested TBAs is that intermediate accounts might not be deployed until first use, yet we need to authenticate the call through the ownership chain. The reference implementation solves this by requiring a **proof of ownership** chain for deep calls. As noted by the Tokenbound developers, an NFT can be nested multiple levels, and if parent account contracts are not deployed, directly calling the usual `token()` method to traverse ownership would revert. The solution is to provide an explicit proof array of parent ownership info, which the account contract can verify by computing expected addresses ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=One%20edge%20case%20that%20may,to%20call%2C%20causing%20a%20revert)). The `executeNested` function implements this approach:

> “Allows the root owner of a nested token bound account to execute transactions directly against the nested account, even if intermediate accounts have not been created.” ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F,))

## Using `executeNested` for Deeply Nested Calls

**What `executeNested` Does:** This function lets an authorized owner call a **child account’s** functions (or make the child account perform an action) by supplying a proof of the ownership chain from the child up to an owner the contract recognizes. In other words, it allows a root NFT holder (or any higher-level TBA) to invoke an operation as if the lowest-level TBA itself is the caller, without needing to separately call each layer. The function signature (from the ABI) is:

```solidity
function executeNested(
    address to,
    uint256 value,
    bytes calldata data,
    uint8 operation,
    (bytes32 salt, address tokenContract, uint256 tokenId)[] calldata proof
) external payable returns (bytes memory result);
``` 

According to the implementation ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=,account%20to%20its%20parent)):

- **`to`** – The target address that the nested account will call. This can be another contract or account that **Account_C** (in our example) should interact with.
- **`value`** – Ether value (in wei) to send with the call.
- **`data`** – The encoded calldata for the operation that the nested account will execute (e.g. encoded function call to `to`).
- **`operation`** – The type of operation: `0` for a regular CALL, `1` for DELEGATECALL, `2` for CREATE, `3` for CREATE2 ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=,account%20to%20its%20parent)). (Use `0` for standard calls in most cases.)
- **`proof`** – An array of structs `ERC6551AccountInfo` each containing `{bytes32 salt, address tokenContract, uint256 tokenId}`. This is the **ownership chain proof** from the current account (“this”) up to a parent account. The proof essentially says *“Which NFT (contract and ID) does each parent account correspond to, all the way to the root.”*

**How to Build the Proof Array:** The proof should list each NFT in the chain from the nested account’s immediate parent **upwards** to the root NFT, in order. *Do not include the NFT of the account you are calling* – start with its parent. Each element in the array provides the data needed to deterministically compute the address of the next account in the chain. The account contract will iterate through this array and verify at each step that the current address indeed owns the specified NFT, then compute the next account’s address ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=ERC6551AccountInfo%20calldata%20accountInfo%3B%20for%20,tokenId)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=try%20IERC721%28tokenContract%29,)). If any link is wrong, it fails with `InvalidAccountProof`. If the chain is correct, the final computed address will be the parent of the target account, and the contract then checks that this final address is authorized (i.e. is indeed an owner or root owner) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=if%20)). In practice:

- **`salt`**: The salt used when the TBA was created via the registry. If you didn’t use a custom salt, this is typically `0x000...000`. (The standard Tokenbound Registry uses a `salt` parameter in `createAccount`; if you passed 0 or left it default, use zero.)  
- **`tokenContract`**: The address of the ERC-721 contract for the parent NFT in this step.
- **`tokenId`**: The token ID of that parent NFT.

**Proof Order Example:** Using the earlier example (Account_C nested under Account_B under Account_A): to call `executeNested` on **Account_C**, we need to prove Account_C is owned by Account_B, which is owned by Account_A, which is ultimately owned by your EOA. Suppose: 

- **Account_C** is bound to NFT **C** (`tokenContract = NFT_C_Address, tokenId = C_ID`), which is owned by Account_B’s address.
- **Account_B** is bound to NFT **B** (`tokenContract = NFT_B_Address, tokenId = B_ID`), owned by Account_A’s address.
- **Account_A** is bound to NFT **A** (`tokenContract = NFT_A_Address, tokenId = A_ID`), owned by you (your EOA address).

To allow Account_C to execute a call, you (as the root owner) will call `Account_C.executeNested(...)` and provide a proof array for **B** and **A** (from child’s parent up to root):

```js
const proof = [
  { salt: saltB, tokenContract: NFT_B_Address, tokenId: B_ID },
  { salt: saltA, tokenContract: NFT_A_Address, tokenId: A_ID }
];
```

Here `saltB` and `saltA` are the salts used when creating Account_B and Account_A (likely 0x0 if default). The order is such that: 

1. **Proof[0]** corresponds to NFT B. The contract will compute the expected address of Account_B from this info and check that the current caller owns NFT B.
2. **Proof[1]** corresponds to NFT A. It will then compute Account_A’s address and check that Account_B (the last computed) owns NFT A. 

After iterating, the contract knows Account_A is the root owner’s account. It then requires that this final account is an authorized executor (in this case, Account_A’s owner is your EOA, so it’s valid) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=try%20IERC721%28tokenContract%29,)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20owner,return%20true)). If everything checks out, **Account_C** will proceed to execute the specified call.

> **Tip:** You can use a helper contract or off-chain script to build this proof. For instance, the helper at **0x2298...3093** (which you have deployed) likely provides a function to generate the `ERC6551AccountInfo[]` array by tracing the ownership chain. Using such a helper avoids manually querying each owner. If using ethers.js with an ABI, ensure you pass the structs correctly (e.g. as an array of objects with `salt`, `tokenContract`, `tokenId` fields, or as a tuple array in the correct order).

**Calling `executeNested`:** Once you have the proof array, calling the function is straightforward. You can do this from an EOA (via a regular transaction) if you are the root owner, or from a parent account contract. In both cases, the account implementation will validate your authority. For example, using **ethers.js** from your EOA account:

```js
// Assume accountCContract is an ethers Contract instance for Account_C at 0xD99FA5...bf4 (your testnet TBA).
const toAddr    = "0x...TARGET";         // address that Account_C will call
const callData = someContractIface.encodeFunctionData("methodName", [arg1, arg2]);
// If sending ETH, callData = "0x" (empty) and set value accordingly.
const valueWei = ethers.utils.parseEther("0.1");  // example 0.1 ETH
const op       = 0;  // CALL

// Prepare proof array (saltA, NFT_A, A_ID), (saltB, NFT_B, B_ID):
const proof = [
  { salt: ethers.constants.HashZero, tokenContract: NFT_A_Address, tokenId: A_ID },
  { salt: ethers.constants.HashZero, tokenContract: NFT_B_Address, tokenId: B_ID }
];

// Execute the nested call
const tx = await accountCContract.executeNested(toAddr, valueWei, callData, op, proof);
const receipt = await tx.wait();
```

In this example, your EOA signs a transaction to `Account_C.executeNested`. The contract will verify that **Account_A** (computed from NFT A proof) is ultimately owned by your EOA (root owner), authorizing the call ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20owner,return%20true)). Then **Account_C** will perform the specified action. The `executeNested` call returns the raw `bytes` result of the inner operation (if any). You can decode this if needed (for example, if calling a function that returns a value).

**Deep Nesting:** The proof array can include as many levels as needed. For a deeply nested account, include every NFT up the chain. For instance, if one more level existed (Account_D owned by Account_C), and you wanted Account_D to act, your proof would include NFT C, B, A in order. The logic inside `executeNested` is iterative, so it will handle N-length proofs in sequence. Just ensure the order and values are correct for each step of ownership.

**Authorization Note:** The reference implementation’s internal `_isValidExecutor` will allow the call if either the immediate NFT owner or the root owner matches the caller ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20owner,return%20true)). In practice, this means if you are the **root NFT holder**, you could sometimes call a nested account directly with no proof (the contract can recursively find the root owner by calling `token()` on each account). However, **this built-in root detection can fail if intermediate accounts aren’t deployed**. The `executeNested` + proof method is the safe, explicit way to handle any depth of nesting ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=One%20edge%20case%20that%20may,to%20call%2C%20causing%20a%20revert)). It also protects against malicious NFT contracts, since the proof is verified via known registry computations rather than trusting the NFT’s `ownerOf` alone ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=An%20alternative%20approach%20that%20accounts,might%20look%20something%20like%20this)). Therefore, prefer using `executeNested` with a proof for deep calls or when in doubt.

### Example: Using `executeNested` in Practice

Suppose **Account_C (0xD99FA5...bf4)** holds some ERC-20 tokens and we want it to transfer tokens to **Account_A** (its top-level parent’s account). We (as the root owner) will call `Account_C.executeNested` to invoke the ERC-20 `transfer` on Account_C’s behalf:

1. **Prepare target call:** The `to` is the ERC-20 token contract address. The `data` is the encoded call to its `transfer(address recipient, uint256 amount)` function, with `recipient = Account_A_Address` and `amount = 100 * 1e18` (for example). `value` is 0 (we’re not sending ETH in this call), and `operation = 0` (normal call).
2. **Build proof:** Determine the chain: NFT B (owned by Account_A) and NFT A (owned by EOA). Suppose both accounts were created with salt 0. We set `proof[0] = {salt:0x0, tokenContract:NFT_B_Address, tokenId:B_ID}` and `proof[1] = {salt:0x0, tokenContract:NFT_A_Address, tokenId:A_ID}`.
3. **Call executeNested:** from our EOA or via a dApp script, call `Account_C.executeNested(tokenAddress, 0, transferData, 0, proof)`. If all is correct, Account_C will call the ERC-20’s `transfer` function, moving the tokens from Account_C to Account_A.

If something is misconfigured, you may encounter errors:
- **`InvalidAccountProof`** – This means the proof chain didn’t verify. Double-check the order of `proof` entries and that each `tokenId` indeed is owned by the address from the previous step. Also ensure you used the correct `salt` (if you explicitly used a salt when creating the account via the registry, include it; otherwise 0). Each `tokenContract` should be the ERC-721 contract of the parent NFT at that level.
- **`NotAuthorized`** – This indicates the final check failed. This can happen if the last computed address in the proof isn’t actually an owner that the contract expects. For example, if the top of your proof is not actually owned by the caller (you), or if you omitted a level and the `current` ended up being your EOA (which wouldn’t match the expected owner at that stage). Ensure the proof includes *all* NFT levels up to the one owned directly by your EOA. Also, if any account in the chain is locked (the Tokenbound implementation supports time-locking accounts), `executeNested` will revert with `AccountLocked` ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=if%20%28next.code.length%20,isLocked%28%29%29%20revert%20AccountLocked%28%29%3B)) – you must unlock it or wait for the lock to expire.

> 💡 **Using the Helper:** The helper contract at **0x2298...3093** can likely generate the correct `proof` for you. For example, it might have a function to trace an account’s ownership chain. You could call something like `TBAHelper.getProofChain(Account_C_Address)` (check the helper’s interface) which returns an array of `(salt, tokenContract, tokenId)` from Account_C’s parent up to root. You can then feed that directly into `executeNested`. This helps avoid manual mistakes in constructing the proof.

## Using `executeBatch` for Multi-Call Transactions

The Tokenbound account also provides an **`executeBatch`** function that allows batching multiple operations in a single transaction. This is useful if you want a TBA to perform several actions atomically (e.g., call multiple contracts, or call and then transfer funds, etc.). The function signature is:

```solidity
function executeBatch(Operation[] calldata operations) external payable returns (bytes[] memory results);
```

Here, `Operation` is a struct with `{ address to; uint256 value; bytes data; uint8 operation; } ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=uint256%20value%3B%20bytes%20data%3B%20uint8,operation%3B))】. Each element in the array describes one call (with the same fields as the single `execute` function). The function will validate the caller is authorized (just like `execute`) and then loop through the array, executing each operation in orde ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=if%20))】. It returns an array of results (bytes) for each sub-call, in sequence.

**Direct use of `executeBatch`:** You can call this on any token-bound account to have it perform multiple operations. For example, say **Account_A** (at address `0xD99FA5...bf4` for instance) needs to do two things: transfer 0.5 ETH to an external address, and call an NFT contract to mint a new NFT. Instead of two separate transactions, you can bundle them:

1. Define the operations:
   - Op1: `{ to: RecipientAddress, value: 0.5 ETH (in wei), data: "0x", operation: 0 }` – this will send 0.5 ETH to `RecipientAddress`.
   - Op2: `{ to: NFT_Contract_Address, value: 0, data: encodedDataForMint, operation: 0 }` – this calls the NFT contract’s `mint` (or whichever function) with the provided data.
2. Put these in an array `ops` in the correct struct format.
3. Call `Account_A.executeBatch(ops)` from an authorized caller (your EOA if you own the NFT bound to Account_A, or via ERC-4337 EntryPoint if using account abstraction). 

Using ethers.js, for example:

```js
const ops = [
  { to: recipient, value: ethers.utils.parseEther("0.5"), data: "0x", operation: 0 },
  { to: nftContract.address, value: 0, data: nftContract.interface.encodeFunctionData("mint", [Account_A_Address]), operation: 0 }
];
const tx = await accountAContract.executeBatch(ops);
await tx.wait();
```

Under the hood, the account contract will check `msg.sender` is allowed (your EOA must be the NFT owner or root owner ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=if%20)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20owner,return%20true))】, then execute each call one by one. If any call fails, the whole batch reverts (the implementation uses a low-level call and reverts on failur ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=bool%20success%3B%20,value%3A%20value%7D%28data))】). If all succeed, you get an array of return data (`results[0]` corresponds to Op1’s result, etc.). In our example, sending ETH has no return data (likely `0x`), and the mint function might return, say, a token ID which would be in `results[1]`.

**Nested usage of `executeBatch`:** You can also invoke `executeBatch` on a **nested account** using a similar proof method as above, if needed. In fact, if you want to batch operations in a child account, you have a couple of options:
- **Call it directly as root** – If you (EOA) are the root owner of that account, you might call `childAccount.executeBatch([...])` directly. The account’s `_isValidExecutor` will likely allow it because you are the root owne ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20root,return%20true))】. However, as discussed, if intermediate TBAs aren’t deployed or you want a more explicit guarantee, you can use the nested approach.
- **Call via `executeNested` from a parent** – You can treat the child’s `executeBatch` as just another function call to be invoked via `executeNested`. For example, suppose Account_B (child) needs to do a batch, and you want to trigger it through Account_A (parent). You could have Account_A call Account_B’s `executeBatch`. Concretely, use `Account_A.executeNested` with `to = Account_B_Address`, `data = AccountB.interface.encodeFunctionData("executeBatch", [ops])`, and `proof` proving Account_A owns NFT A (if Account_A itself is root, proof might be empty or not needed in this one-level case). 

   *Example:* Account_A wants Account_B to do two actions (maybe Account_B holds some assets that need moving). You can do: 
   ```js
   const ops = [ ... operations for Account_B ... ];
   const batchData = accountBContract.interface.encodeFunctionData("executeBatch", [ops]);
   await accountAContract.executeNested(Account_B_Address, 0, batchData, 0, proofForA_owns_B);
   ```
   Here `proofForA_owns_B` would contain the info for NFT B only (since Account_A is calling and is the immediate owner of NFT B). Account_A will then call into Account_B, which will verify Account_A is its owner and execute the batch internally. This results in Account_B performing all the ops. 

   Similarly, you could go down multiple levels: the `to` in an `executeNested` could be a deeply nested account’s address with the `data` being an `executeBatch` call on that account, and the proof listing the chain above it. This effectively combines the two mechanisms: you prove ownership chain and then batch multiple actions at the target.

**Batch within Batch:** You might wonder if you can nest an `executeNested` call inside a batch. Yes – since batch operations are just generic calls, one of the `Operation` entries could itself be a call to another account’s `executeNested`. For example, you could have Account_A do two things in a batch: (1) something itself, and (2) trigger Account_C (two levels down) to do something via `executeNested`. This is advanced, but the flow would be:
- Op1: `{ to: X, ... }` some call from Account_A.
- Op2: `{ to: Account_C_Address, value:0, data: AccountC.encodeFunctionData("executeNested", [target, val, innerData, 0, proof_B_to_A]), operation: 0 }`, where `proof_B_to_A` proves Account_B and Account_A links (so that Account_C will accept Account_A’s call). 

When the batch runs, Op2 will invoke Account_C’s `executeNested` with Account_A as the caller, and the proof tells Account_C that Account_B (computed) is owned by Account_A – authorizing the call. Then Account_C executes the `target` call. All this happens in one outer transaction. This illustrates how flexible these building blocks can be, though be mindful of the added gas from multiple layers of calls.

## Transferring Funds or Assets Between Nested TBAs

Transferring Ether or tokens between TBAs uses the same mechanisms described above – you just have to craft the correct call data for the transfer and target the correct account. A token-bound account is essentially a smart wallet, so to move assets **from** it you make the account call the token’s contract (or native transfer), and to move assets **to** it you send/transfer to its address. Below are a couple of common scenarios:

- **Sending ETH from one TBA to another:** You can simply call `execute`/`executeNested` with `to = recipient_TBA_address` and `value = amount` of Ether, and `data = "0x"`. For example, to have Account_B send 0.2 ETH to Account_A, you could call:  
  `Account_B.executeNested(Account_A_Address, 0.2 ETH, "0x", 0, proof_for_B)`  
  where `proof_for_B` proves you’re allowed (if you are the root owner, this might just be the parent info). The Account_B contract will transfer 0.2 ETH to Account_A. No function call data is needed since a value transfer with empty data just triggers Ethereum’s default `receive()` on Account_A (the reference implementation includes a `receive()` payable function to accept ETH). Account_A will now have an increased ETH balance. *(If Account_B is not nested but directly controlled by you, you could use `execute` or `executeBatch` instead of nested proof.)*

- **Transferring ERC-20 tokens between TBAs:** If Account_C holds some ERC-20 tokens and you want to send them to Account_B, have Account_C call the ERC-20’s `transfer(to, amount)` function with `to = Account_B_Address`. For example, encode the ERC-20 interface call and use `executeNested` on Account_C as shown earlier. This will move tokens from Account_C’s balance to Account_B’s balance. Ensure Account_C is indeed the owner of those tokens (no approval needed since it’s sending its own tokens).

- **Transferring NFTs (ERC-721/ERC-1155) between TBAs:** Suppose Account_A owns an NFT **X** (not the one that defines its existence, but another collectible) and we want to move it into Account_B. Use the NFT’s `safeTransferFrom`. Account_A would call `safeTransferFrom(Account_A_Address, Account_B_Address, X_ID)`. You can do this via `executeBatch` or `execute`. For instance, using batch for safety:  
  ```js
  const nftData = nftContract.interface.encodeFunctionData(
      "safeTransferFrom", [AccountA_Address, AccountB_Address, X_ID]
  );
  await accountAContract.executeBatch([
    { to: nftContract.address, value: 0, data: nftData, operation: 0 }
  ]);
  ```  
  This will trigger the NFT contract to transfer token X from Account_A to Account_B. Since Account_A is the owner, and it’s the caller (msg.sender) of `safeTransferFrom`, the NFT contract will accept it (the owner is calling to transfer to another address). After the call, Account_B will own NFT X. (Account_B could later manage or transfer it as needed by similar calls.)

  If the sending account is nested and you are calling from root, just apply `executeNested` with the appropriate proof. For example, to have Account_C transfer an NFT it holds to Account_A, you’d encode `safeTransferFrom(Account_C_Address, AccountA_Address, tokenId)` as data, and call `Account_C.executeNested(nftContractAddr, 0, data, 0, proof)`.

- **Transferring the bound NFTs themselves:** You typically do **not** use `execute` for transferring the NFT that defines the account’s ownership (e.g., NFT B that Account_A owns) – that is done by a normal ERC-721 transfer of NFT B (which will automatically change the ownership of Account_B). For completeness: if you transfer the parent NFT (A) to someone else, the ownership of Account_A changes (the new owner can then control it). If you transfer a child NFT (B) from Account_A to another wallet, Account_B’s owner changes (it would become owned by that other wallet’s address). In such cases, the chain of TBAs is effectively broken or altered. The contracts themselves don’t prevent you from transferring the NFTs that link the TBAs, but once ownership changes, the old owner can no longer control the account. So be cautious and ensure that if you move NFTs between owners, you update how you call the TBA (the new owner will need to use their own authority).

Overall, moving assets between TBAs is just a matter of making the source account call the appropriate token contract function with the destination account’s address. The combination of `executeNested` and `executeBatch` gives you flexibility to script complex interactions across nested accounts in one go. For example, you could batch a series of transfers from multiple child accounts all initiated by a single root call.

## Tips and Troubleshooting

- **Calldata formatting:** Always ensure you encode the `data` exactly as the target contract expects. If you’re calling a function on a target contract, use its ABI to encode the function call (as shown with `encodeFunctionData`). The `to` address should be the contract you want to interact with, not necessarily the TBA (unless the TBA itself is the target). For pure ETH transfers, use an empty `data` payload and set `value`. For contract calls, usually `value` is 0 (unless you are also sending ETH to a payable function).
- **ABI compliance:** If constructing the transaction off-chain, use the ABI of the account implementation that includes `executeNested` and `executeBatch` so that your library knows the struct types. For example, in ethers.js you might import the ABI JSON (which should have `ERC6551AccountInfo[]` for the proof and `Operation[]` for the batch). When passing structs:
  - For `Operation[]`, you can pass an array of objects like `{to:..., value:..., data:..., operation:...}` as we did. Ensure the keys match exactly or the order matches the tuple order.
  - For the `proof` array, pass an array of `{salt: ..., tokenContract: ..., tokenId: ...}` objects. Alternatively, you can construct the tuple array as `[ [salt, tokenContract, tokenId], [...] ]`. If the call fails to encode, double-check the format your library expects for nested structs.
- **Intermediate accounts not deployed:** A great feature of `executeNested` is that intermediate TBAs in the chain need not be deployed for it to work. The address computation via the registry is deterministi ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=An%20alternative%20approach%20that%20accounts,might%20look%20something%20like%20this))】. As long as the NFT ownership is in place, the proof verification will succeed. So you can control a deeply nested account without ever explicitly deploying the parent accounts (they will be computed as addresses and the ownership verified via `ownerOf`). This saves gas if you don’t need those intermediate accounts to do anything else. Keep in mind, if later you *do* want to use an intermediate account, you’ll need to deploy it via the registry’s `createAccount` function (or another `executeNested` call targeting it, which effectively deploys it on the fly in Tokenbound’s implementation). The helper contract can also help compute addresses without deploying. 
- **Locked accounts:** The Tokenbound implementation has an `isLocked()` feature (time-lock or permanent lock) on accounts. If any account in the chain is locked, `executeNested` will revert with `AccountLocked ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=if%20%28next.code.length%20,isLocked%28%29%29%20revert%20AccountLocked%28%29%3B))】. Ensure none of the TBAs in the chain are locked when performing nested ops. You can unlock if you have a guardian setup, or wait if it’s time-locked.
- **Using EntryPoint (ERC-4337):** The account implementation is EntryPoint/Account Abstraction compatible. If you’re using ERC-4337, the EntryPoint is an authorized caller by defaul ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=function%20_isValidExecutor,return%20true))】. This means you can package these calls in a UserOperation. For example, you could create a UserOp where `callData` is `account.executeBatch(...)` or `account.executeNested(...)`. Your wallet code would sign it, and the EntryPoint will call the account. The checks `_isValidExecutor(entryPoint)` will pass, and internally the account will verify your signature via ERC-1271 (since the account knows the NFT owner). Ensure your AA wallet is correctly set up to sign as the NFT owner. This is advanced, but it allows gas sponsored or multi-transaction flows without direct EOA transactions.
- **Consulting references:** For more context, see the [Tokenbound docs and example ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F,memory%29)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=,account%20to%20its%20parent))】] and a community write-up on Nested ERC-6551 Accounts (e.g. *InHaus NFT Smart Wallets – Nested ERC-6551 Accounts*). The Ethereum Magicians thread also provides insight into the design of `executeNested ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=One%20edge%20case%20that%20may,to%20call%2C%20causing%20a%20revert))】. These confirm that providing a proof array is the intended way to operate nested accounts seamlessly when direct calls would be cumbersome or impossible.

By following the above guide, you should be able to craft transactions that let deeply nested TBAs carry out operations as needed. You can call `executeNested` with the proper proof to instruct a buried account to act, use `executeBatch` to bundle multiple actions, or even combine them for complex nested interactions – all while respecting the ERC-6551 ABI and authorization model. Good luck with your implementation, and always test with a small depth and value first to ensure your proof and calldata formatting are correct before scaling up to more complex nested operations!

**Sources:**

- Tokenbound Account Implementation (v0.3.1) – function docs for `executeNested` and `executeBatch ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F,memory%29)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=,account%20to%20its%20parent))】  
- Tokenbound Account code – ownership proof verification loop in `executeNested ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=ERC6551AccountInfo%20calldata%20accountInfo%3B%20for%20,tokenId)) ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=try%20IERC721%28tokenContract%29,))】, authorization check ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=%2F%2F%20Allow%20execution%20from%20owner,return%20true))】, and batch execution logi ([
	AccountV3Upgradable | Address 0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952 | Etherscan
](https://etherscan.io/address/0x2326aa72fb2227f7c685fe9bc870ddfbe27aa952#:~:text=uint256%20length%20%3D%20operations.length%3B%20bytes,length))】.  
- *Ethereum Magicians*: ERC-6551 Nested ownership discussion by Tokenbound devs – explains using a proof array for undeployed parent account ([ERC-6551: Non-fungible Token Bound Accounts - #62 by jay - ERCs - Fellowship of Ethereum Magicians](https://ethereum-magicians.org/t/erc-6551-non-fungible-token-bound-accounts/13030/62#:~:text=One%20edge%20case%20that%20may,to%20call%2C%20causing%20a%20revert))】.
