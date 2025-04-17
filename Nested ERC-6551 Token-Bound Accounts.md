# Nested ERC-6551 Token-Bound Accounts: Implementation Guide

## Setup: ERC-6551 Registry and Account Deployment on Sonic Testnet

**Registry:** Ensure the ERC-6551 registry is deployed at the standard address `0x000000006551c19487814612e58FE06813775758` on the Sonic Testnet. This is the same singleton registry address used on Ethereum mainnet and other chains ([Smart Contract Addresses - Tokenbound Documentation](https://docs.tokenbound.org/contracts/deployments#:~:text=EVM%20Network%20Chain%20ID%20Registry,Base%20Goerli%20845310x000000006551c19487814612e58FE06813775758%20Linea%20Goerli)) ([Smart Contract Addresses - Tokenbound Documentation](https://docs.tokenbound.org/contracts/deployments#:~:text=Ethereum%2010x000000006551c19487814612e58FE06813775758%20Polygon%201370x000000006551c19487814612e58FE06813775758%20Optimism,100x000000006551c19487814612e58FE06813775758%20Base%2084530x000000006551c19487814612e58FE06813775758%20Linea%20591440x000000006551c19487814612e58FE06813775758)). All token-bound accounts (TBAs) will be deterministically derived through this registry.

**Account Implementation:** Deploy the **AccountV3** implementation contract (the ERC-6551 account logic) on Sonic. When creating TBAs, use this implementation’s address in registry calls. The AccountV3 standard supports executing transactions (`execute`), nested calls (`executeNested`), and batch calls (`executeBatch`), as well as nested ownership resolution. Each TBA is a proxy or instance of this logic tied to a specific NFT.

**Getting TBA Addresses:** To get the address of a token-bound account for a given NFT, call the registry’s `account(...)` function with the implementation address, the NFT’s chain ID, token contract address, token ID, and a salt (use the same salt that was used at creation, often `0` by default) ([A Complete Guide to ERC-6551: Token Bound Accounts | Guides | GoldRush](https://goldrush.dev/guides/a-complete-guide-to-erc-6551-token-bound-accounts/#:~:text=Image%20,It%20also%20takes%20five%20arguments)) ([A Complete Guide to ERC-6551: Token Bound Accounts | Guides | GoldRush](https://goldrush.dev/guides/a-complete-guide-to-erc-6551-token-bound-accounts/#:~:text=,Bound%20Account%20will%20be%20associated)). For example: 

```ts
const registry = new ethers.Contract(registryAddress, RegistryABI, signer);
const tbaAddress = await registry.account(
  accountImplAddress, 
  chainId, 
  nftContractAddress, 
  nftTokenId, 
  0  // salt used during createAccount
);
```

This returns the deterministic address of the TBA. (If the account wasn’t already created, you’d use `createAccount(...)` with the same parameters to deploy it.) Now you can instantiate the account contract in your ethers code:

```ts
const AccountABI = [
  "function execute(address to, uint256 value, bytes data) external returns (bytes)",
  "function executeNested(uint256 chainId, address tokenContract, uint256 tokenId, bytes data, bytes proof) external returns (bytes)",
  "function executeBatch(address[] calldata targets, uint256[] calldata values, bytes[] calldata data) external returns (bytes[])",
  "function token() external view returns (uint256 chainId, address tokenContract, uint256 tokenId)"
];
const tba = new ethers.Contract(tbaAddress, AccountABI, signer);
```

Where `signer` is an ethers Signer for an EOA that is authorized to control the TBA (more on authorization below).

## Authorization: Controlling Nested TBAs via Owners

A token-bound account will **only execute calls if the caller is authorized** – typically the owner of the NFT (or an owner up the chain of nested NFTs). In AccountV3, the `execute` functions check that the `msg.sender` is an allowed executor before performing the operation. Specifically, an account allows calls from: 

- **Its NFT’s direct owner:** The address that currently owns the NFT controlling this account (for a nested TBA, this is likely another TBA one level up).  
- **The root owner of a nested chain:** The ultimate owner at the top of the chain (e.g. an EOA that holds the top-level NFT) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)).  
- (It may also allow an authorized third-party or EntryPoint for ERC-4337, if configured ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20ERC,EntryPoint)) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=if%20%28OPAddressAliasHelper.undoL1ToL2Alias%28_msgSender%28%29%29%20%3D%3D%20address%28this%29%29%20)).)

In practice, this means **your EOA (root owner)** can directly call `execute` on any deeply nested TBA it ultimately owns, even if there are intermediate levels ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)). Likewise, an intermediate account can call `execute` on its immediate child account. The AccountV3 contract internally verifies the ownership chain. (It computes the NFT owner and even climbs up multiple nesting levels to find the root owner ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=function%20_rootTokenOwner)).) 

**Example:** If an EOA owns NFT_1, which has a TBA_1; inside TBA_1 is NFT_2 with TBA_2, and so on up to TBA_5 – the EOA is the root owner of all those TBAs. The EOA can directly invoke TBA_5’s functions because the contract will recognize the EOA as the ultimate owner in the chain of custody. The code checks the NFT’s owner and owner’s owner recursively to authorize the call ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=function%20_rootTokenOwner)).

**⚠️ Note:** Always call TBA functions using a signer that corresponds to an authorized owner. If you call with the wrong signer, the contract will reject the `execute/executeNested` with an authorization error. 

Now, let's go through each requested operation with code examples and logic:

## Sending Native Tokens from a Nested TBA to an EOA

To send native tokens (e.g. ETH on Sonic) from a token-bound account to an EOA, use the `execute()` function to perform a low-level ETH transfer. The `execute` function allows the TBA to call any address with a given ETH `value` and calldata. For a simple transfer, the calldata can be empty.

**Example:** TBA_2 (a nested account) sending 0.5 ETH to a user’s EOA:

```ts
// Assume signer is the root owner (EOA) or direct parent of TBA_2, so it's authorized
const tba2 = new ethers.Contract(tba2Address, AccountABI, signer);

// Prepare parameters: target is the recipient EOA, value in wei, data empty
const recipient = "0xYourEOAAddress";
const amountWei = ethers.parseEther("0.5"); // 0.5 ETH in wei
// Call execute to send ETH
const tx = await tba2.execute(recipient, amountWei, "0x");
await tx.wait();
console.log(`Sent 0.5 ETH from TBA_2 to EOA ${recipient}`);
```

Under the hood, the AccountV3 contract will do `recipient.call{value: 0.5 ETH}("")`. This succeeds if `msg.sender` is authorized (our signer EOA is the root owner, so it is) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)). No separate signature is needed in this direct call mode – the act of the signer submitting the transaction is the authorization. (AccountV3 also supports 4337-style delegated execution with signatures, but here we use a direct call for simplicity ([How to Create and Deploy a Token Bound Account (ERC-6551) | QuickNode Guides](https://www.quicknode.com/guides/ethereum-development/nfts/how-to-create-and-deploy-an-erc-6551-nft#:~:text=This%20critical%20interface%20allows%20TBAs,on%20behalf%20of%20the%20account)).)

If TBA_2 is one level down from the EOA, the EOA is the direct NFT owner so it’s authorized. If TBA_2 is deeper (say EOA → TBA_1 → NFT_2 → TBA_2), the EOA is still the root owner, which AccountV3 recognizes and allows ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=if%20,true)). In other words, you can directly call a nested account from the top-level EOA without first routing through intermediate accounts.

**Result:** The EOA receives the native tokens. The TBA’s balance decreases accordingly. This is effectively a “withdrawal” of ETH from the TBA to its owner.

## Withdrawing Native Tokens from Nested TBAs to the EOA

“Withdrawing” typically implies pulling assets from one or more nested accounts up to the root owner. There are two ways to approach this:

- **Direct calls per account:** As shown above, the root owner EOA can individually call each nested account’s `execute` to send funds up. For example, to withdraw ETH from a chain of TBAs (TBA_5, TBA_4, ... up to TBA_1), the EOA could call each one’s `execute` in separate transactions, sending funds ultimately to the EOA. This method is straightforward but involves multiple transactions.

- **Chained call in one transaction:** Using `executeNested`, a parent account can instruct a child account *within the same transaction*. The `executeNested` function allows an account to call an operation *in the context of one of its descendant token accounts*. This is useful for multi-level nesting, as it can traverse intermediate ownership in a single outer transaction.

### Using `executeNested` for Multi-Level Withdrawals

The `executeNested(chainId, tokenContract, tokenId, data, proof)` function (as defined in AccountV3) attempts to execute `data` on the token-bound account of the specified NFT (`chainId/tokenContract/tokenId`). If the target NFT’s account is deeper than one level, you must provide a `proof` (typically an encoded list of intermediate NFT ownership steps) so the contract can validate the path ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=while%20%28ERC6551AccountLib.isERC6551Account%28_owner%2C%20__self%2C%20erc6551Registry%29%29%20)). 

**Proof construction:** You can build the proof by gathering each nesting level’s NFT info using the `.token()` view on each intermediate account. Starting from the target NFT and moving upward:
1. Identify the NFT that the current account (caller) directly owns (if any) which ultimately leads to the target. 
2. If the target NFT is not directly owned, include the details of the intermediary NFTs in between.

For example, suppose TBA_1 owns an NFT (call it NFT_2) which has its own account TBA_2 holding some ETH. The EOA could **withdraw ETH from TBA_2 in one call via TBA_1** using `executeNested` as follows:

- **Step 1:** Build the call data for the child account (TBA_2) to send ETH to the EOA. We encode an `execute` call on the child.  
- **Step 2:** Call `TBA_1.executeNested` targeting NFT_2, with the encoded data from step 1. No proof is needed here because NFT_2 is directly owned by TBA_1.

**Code example (two-level nesting):**

```ts
const tba1 = new ethers.Contract(tba1Address, AccountABI, signer); // signer is EOA (owns NFT_1)

// We want TBA_1 to tell TBA_2 (NFT_2's account) to send 0.5 ETH to the EOA.
const targetNFT = { chainId:  <SonicChainID>, tokenContract: nft2Address, tokenId: nft2Id };

// Encode the inner call: TBA_2.execute(EOA, 0.5 ETH, "")
const innerData = tba2.interface.encodeFunctionData("execute", [
    recipientEOA, 
    ethers.parseEther("0.5"), 
    "0x"
]);

// ExecuteNested on TBA_1, targeting NFT_2's account, with no additional proof needed (direct child)
const txNested = await tba1.executeNested(
    targetNFT.chainId,
    targetNFT.tokenContract,
    targetNFT.tokenId,
    innerData,
    "0x"  // empty proof (bytes) since NFT_2 is directly owned by TBA_1
);
await txNested.wait();
```

What happens here: TBA_1 (msg.sender is EOA, authorized as owner of NFT_1) looks up the account for NFT_2 via the registry and calls `execute` on TBA_2 with the provided `innerData`. TBA_2 sees the call coming from its direct owner (TBA_1) and executes the ETH transfer to the EOA. The result is that 0.5 ETH moves from TBA_2 to the EOA, all within a single outer transaction initiated by the EOA.

For **deeper nesting (3+ levels)**, you would include a `proof` array of intermediate NFTs. For instance, if TBA_1 -> TBA_2 -> TBA_3 (with EOA as root), and you want TBA_1 to trigger an action in TBA_3, you supply the NFT details of the middle level (NFT_2) as proof. The AccountV3 contract will verify that TBA_1 owns NFT_2, and NFT_2’s account owns NFT_3, before allowing the call to propagate. Internally, it uses the provided proof to check each ownership hop via `ownerOf` and the registry ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=while%20%28ERC6551AccountLib.isERC6551Account%28_owner%2C%20__self%2C%20erc6551Registry%29%29%20)) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F)).

**General Proof Example:** If an account *A* wants to call a nested account *C* which it doesn’t directly own, but *A* owns an NFT whose account *B* owns the target NFT:
- The `proof` would contain the (chainId, contract, tokenId) of that intermediate NFT. 
- Account *A*’s `executeNested` will verify it owns that NFT, find *B* (its TBA), then verify *B* owns the target NFT before calling *C*.

In summary, to withdraw native tokens from deep TBAs in one go, you can nest `executeNested` calls or perform sequential `execute` calls. The **nested approach** packs the calls and proofs so that each intermediate account calls the next, until the deepest account executes the transfer to the top. The **direct approach** simply calls each level’s `execute` from the root owner. Both achieve moving funds up to the EOA.

## Sending ERC-20 Tokens from Nested TBAs

Sending ERC-20 tokens from a TBA is similar to sending ETH, except you call the token contract’s transfer function via `execute`. The TBA must hold the ERC-20 balance, and then it can act as the `msg.sender` to transfer those tokens out.

**Example:** TBA_3 holds some ABC tokens (an ERC-20). We want to send 100 ABC from TBA_3 to an external address (could be the owner EOA or any address):

```ts
const tba3 = new ethers.Contract(tba3Address, AccountABI, ownerSigner);

// Prepare the ERC-20 transfer call data
const erc20Abi = ["function transfer(address to, uint256 amount) external returns (bool)"];
const erc20 = new ethers.Interface(erc20Abi);
const tokenContract = "0x...ABC_TokenContractAddress";
const transferData = erc20.encodeFunctionData("transfer", [recipientAddress, BigInt(100 * 1e18)]);

// Call execute on the TBA to invoke the transfer
const tx = await tba3.execute(tokenContract, 0, transferData);
await tx.wait();
console.log("TBA_3 transferred 100 ABC tokens to", recipientAddress);
```

Here, `to = tokenContract` (the ERC-20 contract), `value = 0` (we’re not sending ETH), and `data` is the ABI-encoded `transfer(address,uint256)` call. When executed, the TBA3 contract will call the ERC-20’s `transfer` function, moving tokens from TBA3’s balance to the recipient. No prior approval is needed because the TBA itself is the token holder and is calling `transfer` as itself.

As always, ensure the call is made by an authorized signer. In this case, the EOA controlling TBA_3 (either the root owner or the direct parent’s owner) would send the transaction. AccountV3 will allow it if the signer is recognized as an owner (direct or root) of TBA_3 ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)).

**Nested scenario:** If TBA_3 is deeply nested and you prefer to trigger the transfer via a higher-level account in one transaction, you can use `executeNested` similarly to the ETH example. For instance, TBA_1 could call `executeNested` targeting NFT_3’s account, with `data` = encoded ERC-20 transfer. You would include proof of intermediate NFTs if TBA_3 isn’t directly owned by TBA_1. The effect is that TBA_3 executes the token transfer at the behest of the higher-level call.

## Withdrawing ERC-20 Tokens up to the EOA

Withdrawing ERC-20 tokens from nested accounts up to an EOA uses the same mechanisms. Essentially, the TBA needs to transfer the tokens to the EOA’s address (or whichever address acts as the “withdrawal” recipient). This can be done with a direct call or nested call:

- **Direct method:** The EOA calls each TBA’s `execute(ERC20.transfer(EOA, amount))`. For example, to withdraw all ABC tokens from TBA_3 to the EOA, call `tba3.execute(tokenContract, 0, transferData)` where `transferData` encodes `transfer(eoaAddress, fullBalance)`. The EOA will then receive the tokens.

- **Via parent (nested):** The EOA could invoke `executeNested` on a parent account to cascade the transfer. For instance, if TBA_2 owns the NFT for TBA_3, the EOA can call `tba2.executeNested(NFT_3, data=ERC20.transfer(EOA, amount), proof="0x")`. TBA_2 will instruct TBA_3 to send tokens to the EOA. This saves an extra transaction if done in one go. Just ensure to include any needed proof if the relationship isn’t direct.

**Example (direct):** withdraw ERC-20 from TBA_3 to EOA:

```ts
// EOA directly calls TBA_3 to send its tokens to EOA
const recipient = ownerEOA;  // withdraw to owner
const balance = await abcToken.balanceOf(tba3Address);
const transferData = erc20.encodeFunctionData("transfer", [recipient, balance]);
await tba3.execute(tokenContract, 0, transferData);
console.log(`Withdrew all ABC tokens from TBA_3 to EOA ${recipient}`);
```

After execution, the EOA’s address now holds those tokens, and TBA_3’s token balance is reduced.

## Executing Arbitrary Contract Interactions from a Nested TBA

One powerful feature of ERC-6551 accounts is that they can perform **any contract interaction** (just like a normal smart wallet) as long as the call is authorized by the NFT owner. You simply provide the target contract address, and the encoded function call data to `execute`. This enables scenarios such as an NFT’s TBA placing marketplace trades, interacting with DeFi protocols, etc., on behalf of the NFT.

**Example:** Let’s say TBA_1 wants to interact with a DeFi staking contract to deposit some tokens it holds. The staking contract has a function `stake(uint256 amount)`. Suppose TBA_1 already has the required tokens approved or in its balance (depending on the contract logic). We can call that function via `execute`:

```ts
const defiContract = "0x...DeFiContractAddress";
const defiAbi = ["function stake(uint256 amount)"];
const defiIface = new ethers.Interface(defiAbi);
const stakeData = defiIface.encodeFunctionData("stake", [ BigInt(50 * 1e18) ]); // stake 50 tokens

// EOA (who owns NFT_1) instructs TBA_1 to call stake(50)
const tx = await tba1.execute(defiContract, 0, stakeData);
await tx.wait();
console.log("TBA_1 called stake(50) on DeFi contract");
```

This will make TBA_1 execute the call to the DeFi contract’s `stake` function. The TBA_1 contract becomes the `msg.sender` of that call, so if the DeFi contract logic requires `msg.sender` to have tokens, it refers to TBA_1’s assets (which it presumably has).

You can similarly call DEX contracts (for swaps), NFT marketplaces, other NFT mint contracts, etc. Just encode the desired function call to the target protocol. If the call requires sending ETH (for example, purchasing an NFT or paying a fee), you can include a `value` > 0 in the `execute` call to forward ETH along with the function data.

**Batch execution:** AccountV3 also provides `executeBatch(address[] targets, uint256[] values, bytes[] data)` to perform multiple calls in one transaction (all from the same TBA) – for example, approving a token and then interacting with a protocol in one go. In ethers, you would call it like:

```ts
await tba1.executeBatch(
  [tokenContractAddress, defiContractAddress], 
  [0, 0], 
  [approveData, stakeData]
);
```

This would make TBA_1 execute both sub-calls in sequence. Ensure the arrays align in length. Batch execution is only for calls from the *same* account; it doesn’t automatically hop through nested levels. If you need to coordinate across accounts, you’d either use nested calls as discussed or have each account do its part in separate transactions.

## Sonic Testnet Considerations

Since Sonic is an EVM-compatible testnet, the above patterns remain the same. Just use Sonic’s chain ID and deployed addresses for the registry and your Account implementation. If Sonic is an Optimism-based chain or similar, AccountV3’s code already handles L2 specifics (like aliasing addresses for L1->L2 calls) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=if%20%28chainId%20%21%3D%20block.chainid%29%20)). In most cases, you won’t need special adjustments beyond deploying the contracts and using the correct chain IDs in registry calls and proofs. 

One thing to verify is the **chainId in `.token()` data** for each account. On Sonic, calling `token()` on a TBA will return an `(originChainId, originContract, originTokenId)`. This should correspond to the chain where the controlling NFT resides. If you are executing cross-chain (e.g., an account on Sonic controlling an NFT on Ethereum L1), you would need to use `executeNested` with a cross-chain proof or a trusted bridge message (AccountV3’s guardian can be configured for cross-chain executors). But if all nesting is on Sonic itself, the `chainId` will match Sonic’s ID for all levels, and on-chain `ownerOf` checks will work directly ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=if%20%28chainId%20%21%3D%20block,0)).

Finally, after implementing the above, **test each function on Sonic with up to 5 nesting levels** as you planned. Start with simple one-level calls (EOA -> TBA) then try multi-level nested calls. If a call fails, inspect the revert reason – it’s often an authorization issue (e.g. wrong signer or wrong proof). By constructing the proof from `.token()` info and using the AccountV3 interface correctly, you can execute arbitrary actions from any nested token-bound account under your control.

**Sources:**

- ERC-6551 Account V3 reference implementation and docs for allowed callers ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=%2F%2F%20Allow%20execution%20from%20owner)) ([contracts/src/AccountV3.sol at main · tokenbound/contracts · GitHub](https://github.com/tokenbound/contracts/blob/main/src/AccountV3.sol#:~:text=function%20_rootTokenOwner))  
- Tokenbound docs on registry address and usage ([Smart Contract Addresses - Tokenbound Documentation](https://docs.tokenbound.org/contracts/deployments#:~:text=EVM%20Network%20Chain%20ID%20Registry,Base%20Goerli%20845310x000000006551c19487814612e58FE06813775758%20Linea%20Goerli)) ([Smart Contract Addresses - Tokenbound Documentation](https://docs.tokenbound.org/contracts/deployments#:~:text=Ethereum%2010x000000006551c19487814612e58FE06813775758%20Polygon%201370x000000006551c19487814612e58FE06813775758%20Optimism,100x000000006551c19487814612e58FE06813775758%20Base%2084530x000000006551c19487814612e58FE06813775758%20Linea%20591440x000000006551c19487814612e58FE06813775758))  
- EIP-6551 guide (Goldsky/Goldrush) on registry `account()` parameters ([A Complete Guide to ERC-6551: Token Bound Accounts | Guides | GoldRush](https://goldrush.dev/guides/a-complete-guide-to-erc-6551-token-bound-accounts/#:~:text=Image%20,It%20also%20takes%20five%20arguments)) ([A Complete Guide to ERC-6551: Token Bound Accounts | Guides | GoldRush](https://goldrush.dev/guides/a-complete-guide-to-erc-6551-token-bound-accounts/#:~:text=,Bound%20Account%20will%20be%20associated))  
- QuickNode ERC-6551 tutorial explaining `execute` functionality ([How to Create and Deploy a Token Bound Account (ERC-6551) | QuickNode Guides](https://www.quicknode.com/guides/ethereum-development/nfts/how-to-create-and-deploy-an-erc-6551-nft#:~:text=This%20critical%20interface%20allows%20TBAs,on%20behalf%20of%20the%20account)) and the concept of arbitrary calls.
