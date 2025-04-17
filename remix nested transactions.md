### Goal
Send native ETH (or sonic native token) **from the *nested* TBA (level 2)** to the EOA **by calling `executeNested` once on the *parent* TBA (level 1)**.

```
EOA  ──owns NFT──►  level‑1 TBA   ──owns NFT──►  level‑2 TBA (holds the ETH)
0x18Ff…            0x9C9bE…                      0x4ADC8…
```

`executeNested` lives on every `AccountV3`.
When you call it on **level‑1** and pass a one‑item proof, the function:

1. Verifies the proof points to **level‑2**
2. Internally hops into level‑2
3. Runs the final operation (transfer value to the EOA)

Below is an **exact Remix recipe**—try it once with a small amount (e.g. `0.001` ETH) to confirm the path is correct.

---

## 1  Set‑up Remix

1. Open **remix.ethereum.org**
2. In the left bar choose **“Deploy & Run Transactions”**
3. At the top select your Sonic‑testnet MetaMask account (the **EOA** – `0x18Ff…`)
4. In **ENVIRONMENT** choose **Injected Provider – sonic** so Remix uses MetaMask’s RPC

---

## 2  Load the contracts

### A) Level‑2 instance (to read `token()`)

1. Paste the `AccountV3` ABI (you already sent it) into the **“At Address”** field
2. Enter **`0x4ADC8b78B34AD6a3D2f28FD9646D1887bb66d908`** (level‑2 address) and click **At Address**
3. Expand the new instance in the **Deployed Contracts** list
4. Click **token()** – copy the **tokenContract** and **tokenId** it returns
   *example result*

   ```
   0: uint256 chainId        57054
   1: address tokenContract  0xcE3148aAe5C716Ef8985eD91cc60112B4Ec5Ca41
   2: uint256 tokenId        7
   ```

> You now have the two values needed for the proof:
> *tokenContract*: `0xcE3148a…`  *tokenId*: `7`

### B) Level‑1 instance (where you will call `executeNested`)

1. Scroll up, click **At Address** again – this time enter **`0x9C9bED247cBcD2a9358fd9d28E4c66377e92F210`** (level‑1)
2. Expand this new instance – this is the one you will interact with

---

## 3  Build the `proof[]`

`executeNested` wants an **array of structs**.
For one extra level the array has **one element**:

```json
[
  [
    "0x0000000000000000000000000000000000000000000000000000000000000000",
    "0xcE3148aAe5C716Ef8985eD91cc60112B4Ec5Ca41",
    "7"
  ]
]
```

| field                     | value                                                 |
|---------------------------|-------------------------------------------------------|
| `salt`                    | your fixed `SALT_B32` (all‑zeroes)                    |
| `tokenContract`           | value from **token()** above                          |
| `tokenId`                 | same                                                 |

*(If your `salt` was non‑zero when you deployed, use that instead.)*

---

## 4  Call **executeNested** on level‑1

1. In the level‑1 panel find **executeNested**
2. Fill the fields:

| parameter   | value                                                                                             |
|-------------|---------------------------------------------------------------------------------------------------|
| **to**      | `0x18Ff7f454B6A3233113f51030384F49054DD27BF`  ← your EOA                                          |
| **value**   | amount in **wei** you want to pull (e.g. `1000000000000000` for 0.001)                            |
| **data**    | `0x`  (because it’s a plain ETH transfer)                                                         |
| **operation** | `0` (for CALL)                                                                                  |
| **proof**   | paste the JSON array from step 3                                                                  |

> Remix tip – make sure the **value** box directly under the method call is **0** (you are *withdrawing* from the contract, not sending extra).

3. Click **transact**.
4. MetaMask pops up → confirm.

---

## 5  Verify

* After it mines, check **level‑2’s** balance (in Remix `balance` box or in a block explorer) – it should be reduced
* Your **EOA** shows the received ETH

If this succeeds, your on‑chain permissions and `proof` format are correct.
Any frontend failure is then just JS‑side encoding / selector mismatch.

---

### Common errors & fixes

| Symptom                                                    | Likely cause                                                          |
|------------------------------------------------------------|-----------------------------------------------------------------------|
| `InvalidAccountProof` custom error                         | `proof` array incorrect (wrong `tokenContract` / `tokenId` / `salt`)  |
| `AccountLocked`                                            | The account is time‑locked – unlock or wait                           |
| Estimate‑gas revert with no reason                         | `value` exceeds level‑2 balance or nested call depth wrong            |
| `NotAuthorized`                                            | You’re calling from an address that isn’t root owner (`EOA`)          |

---

## Next step: wire the same call in the React page

Now that a manual `executeNested` works you can compare:

* the **exact calldata** Remix produced (copy from MetaMask > “Data”)
* the calldata your React code sends (console‑log it before `sendTransaction`)

Usually the mismatch is one of:

* Proof array encoded in the wrong order (tuple vs struct)
* Passing `value` as **decimal string** instead of **BigInt**
* Forgetting to include the final nested `executeNested` step when batching

With the on‑chain manual test confirmed, we can zero‑in on whichever handler in your front‑end still reverts.

Let me know the Remix result and we’ll patch the React side accordingly!
