# INV-9: 21,000 gas no longer guarantees a plain ETH transfer

**Type:** Price and Accounting

Sending ETH to an address that does not exist yet now costs 204,600 gas, not 21,000. A transfer sent with 21,000,
or with any other limit below 204,600, is not rejected. It is mined with status 0. The sender pays the fee and uses
up the nonce, and no ETH moves. Software that treats "mined" as "paid", or that sends with a fixed 21,000 (or 21,000
plus a margin), gets the wrong result.

## Description

Before Glamsterdam, a plain ETH transfer to an EOA without a 7702 delegation cost exactly 21,000 gas, whether or not
the recipient existed. Once mined, it could not fail. A lot of off-chain software relies on that:

- wallets, exchanges and payment systems send ETH with a hard-coded `gasLimit: 21000`;
- deposit watchers credit any mined transaction to a deposit address, without reading the receipt;
- transactions are signed once, offline or long in advance, with 21,000 gas.

EIP-2780 (scheduled for Glamsterdam with EIP-8037) splits the intrinsic cost of a transaction into parts. The cost
does not change for a transfer to an existing EOA. It does change for other recipients:

| Plain ETH transfer (no calldata) to | Osaka | Amsterdam |
|---|---|---|
| An existing EOA (it has a balance or a nonce) | 21,000 | 21,000 |
| An address that does not exist (EIP-161 empty: no balance, nonce or code) | 21,000 | 21,000 + **183,600** state gas = 204,600 |
| An EIP-7702 delegated account | 21,000 + its code | 24,000 + its code (3,000 of it to load the delegate, if cold) |
| Zero value, to another EOA or a new address | 21,000 | 15,000 |

The 183,600 for the new account is not part of intrinsic gas. It is a **runtime** charge, made after the transaction
has been accepted as valid. A transaction that covers the intrinsic 21,000 but not the runtime charge is therefore
valid. It can be included in a block, where it runs out of gas:

- the receipt has status 0 and `gasUsed` equal to the full gas limit;
- the sender pays the fee and its nonce goes up;
- no ETH moves, the recipient is not created, and there is no EIP-7708 transfer log;
- the transaction still carries its `to` and `value` fields in the block.

Before, a transaction with too little gas for what it had to pay before execution was invalid and never mined.
Now "mined" no longer means "ETH transferred", even for a plain transfer to an address with no code. 

An address does not exist if it has a zero balance, a zero nonce and no code. In practice that is an address that
has never received ETH and never sent a transaction. Many real recipients are in that state, and some always are:

- freshly generated deposit, invoice and payment addresses;
- exchange deposit addresses that have only received tokens, when they get "gas money" to sweep the tokens;
- counterfactual smart accounts (ERC-4337) funded before they are deployed;
- stealth addresses (ERC-5564), which are new by design;
- a user's address on a chain they have just bridged to ("refuel" or gas-drop services), faucets and airdrops.

## Vulnerable example

An exchange backend (TypeScript, ethers v6). Every user has their own deposit address, which is new until the first
deposit.

```ts
import { JsonRpcProvider, Wallet } from "ethers";

const provider = new JsonRpcProvider(RPC_URL);
const hotWallet = new Wallet(HOT_WALLET_KEY, provider);

// Credit ETH deposits.
async function scanBlock(blockNumber: number) {
  const block = await provider.getBlock(blockNumber, true);
  for (const tx of block!.prefetchedTransactions) {
    const user = userOfDepositAddress(tx.to);
    // Assumption: a plain transfer to one of our EOAs cannot fail once mined.
    if (user && tx.value > 0n) await credit(user, tx.value, tx.hash);
  }
}

// Pay out ETH withdrawals.
async function withdraw(to: string, amount: bigint) {
  // Assumption: 21,000 is always enough to send ETH to an address without code.
  const tx = await hotWallet.sendTransaction({ to, value: amount, gasLimit: 21_000n });
  await markPaid(to, amount, tx.hash);
}
```

Two things go wrong after the fork:

- **Fake deposits.** An attacker sends 100 ETH to their own new deposit address with a gas limit of 21,000. The
  transaction is mined with status 0, so the ETH never leaves the attacker's account, but `scanBlock` credits
  100 ETH. The attacker pays only the fee for 21,000 gas and can repeat this, because the address still does not
  exist afterwards. They need 100 ETH in their account for the transaction to be valid, but they keep it. They then
  withdraw the credited ETH, which comes out of other users' funds.
- **Lost withdrawals.** A user withdraws to a new address. The transfer is mined with status 0, the user receives
  nothing and the withdrawal is marked as paid.

## Fixed example

```ts
import { Transaction } from "ethers";

const MAX_TRANSFER_GAS = 204_600n;              // a transfer that creates an account; anything more runs code

// Credit ETH deposits: only what was really transferred.
async function scanBlock(blockNumber: number) {
  const block = await provider.getBlock(blockNumber, true);
  if (!block) throw new Error(`block ${blockNumber} not available yet`);
  for (const tx of block.prefetchedTransactions) {
    const user = userOfDepositAddress(tx.to);
    if (!user || tx.value === 0n) continue;

    const receipt = await provider.getTransactionReceipt(tx.hash);
    // No receipt yet, or one from another block after a reorg: scan the block again later. Never skip.
    if (!receipt || receipt.blockHash !== block.hash) throw new Error(`rescan block ${blockNumber}`);
    if (receipt.status !== 1) continue;         // mined is not the same as transferred
    await credit(user, tx.value, tx.hash);      // must be idempotent per transaction hash
  }
}

// Pay out ETH withdrawals.
async function withdraw(withdrawalId: string, to: string, amount: bigint) {
  // The recipient's code decides what an estimate returns. Pay only plain transfers here; send addresses
  // with code (contracts, 7702 delegations) to a separate path with its own limits.
  if ((await provider.getCode(to)) !== "0x") throw new Error(`${to} has code`);

  // 21,000 to an existing EOA, 204,600 to a new address. No margin, and a cap: if the address gets code
  // before inclusion, its code can never use more than this.
  const gasLimit = await provider.estimateGas({ from: hotWallet.address, to, value: amount });
  if (gasLimit > MAX_TRANSFER_GAS) throw new Error(`unexpected gas ${gasLimit} for ${to}`);

  // Record the signed transaction before broadcasting it. After a crash, rebroadcast it or read its
  // receipt; never sign a new one for the same withdrawal while this one can still be mined.
  const signed = await hotWallet.signTransaction(
    await hotWallet.populateTransaction({ to, value: amount, gasLimit }),
  );
  await markSent(withdrawalId, signed, Transaction.from(signed).hash!);
  const tx = await provider.broadcastTransaction(signed);

  const receipt = await tx.wait();              // ethers v6 throws on status 0; other libraries may not
  if (receipt?.status !== 1) throw new Error(`withdrawal ${tx.hash} failed`);
  await markPaid(withdrawalId, tx.hash);
}
```

Reading the receipt status fixes top-level transfers. To catch every ETH transfer, including those made by
contracts, credit deposits from the EIP-7708 transfer logs instead, or from traces. A failed transfer emits no log.
An EIP-7708 log has the same topic and layout as an ERC-20 `Transfer` log, and any contract can emit one. Credit ETH
only for logs whose emitter is `0xfffffffffffffffffffffffffffffffffffffffe`; any other emitter is a token, not ETH.

## Common pitfalls

- **"Mined means paid."** Deposit watchers, payment processors, invoicing and checkout systems that do not read the
  receipt status. Before, this was safe for plain transfers to EOAs without a delegation, such as an exchange's own
  deposit addresses. Now anyone can make it credit ETH that never arrived.
- **"No code means 21,000 gas."** Certain wallets skip estimation when the recipient has no code. A new address has
  no code either. The check must be "exists and has no code", and it is simpler to always estimate.
- **Hard-coded 21,000 in senders:** hot wallets, withdrawal and payout batches, faucets, airdrops, gas-money
  top-ups for sweeping token deposit addresses, refuel services, keepers that fund new addresses. Also limits
  derived from it: "21,000 plus a margin", fixed 30,000 or 50,000 defaults, and "21,000 plus calldata" for
  transfers that carry a memo. All of them are below 204,600.
- **"Send max".** Computing the value as `balance - 21,000 × maxFee` leaves room for only 21,000 gas, which is too
  little for a new recipient. With a gas limit of 204,600 the value must leave the larger fee in the account, or
  the transaction is invalid.
- **Fee quotes.** UIs and APIs that show "21,000 × gas price" under-quote a transfer to a new address almost tenfold.
- **Proofs of payment from transaction inclusion.** Contracts or services that accept a proof that a transaction
  with some `to` and `value` is in a block (a transaction-trie proof) without proving its receipt accept failed
  transfers. Prove the receipt status, or the EIP-7708 log together with its emitter.
- **Classifying by gas.** `gasUsed == 21000` no longer means "a successful plain transfer". A failed transfer to a
  new address uses its whole gas limit, so exactly 21,000 if it was sent with 21,000. A successful zero-value one
  uses 15,000.

## Pre-signed transactions

This section is about plain ETH transfers. For signed values in general, see "Pre-signed transactions and other
signed gas values" in the [README](../README.md#pre-signed-transactions-and-other-signed-gas-values).

A signed transaction cannot change its gas limit. Check every transaction that is signed now and broadcast later:

- **Signed before the fork and broadcast after it:** queued or scheduled payouts, transactions held in a custody
  approval flow, transactions in a private mempool or a relayer's queue.
- **Signed offline or by a signer that cannot see the chain:** air-gapped and hardware wallets, cold storage. These
  cannot tell whether the recipient exists. If they default to 21,000, they fail for every new address.
- **Signed for later use:** inheritance and "dead man's switch" transactions, emergency sweeps and recovery
  bundles. These often pay to a new, unused address on purpose.

Each of these, sent with less than 204,600 gas to an address that does not exist, is mined and fails. It uses up
its nonce, so it cannot be resent. If later transactions depend on it, for example because the recipient was meant
to spend the ETH, they fail as well. Re-sign before the fork with a gas limit for a new recipient, or make sure the
recipient exists first (1 wei is enough, but sending it needs 204,600 gas).

## Gas for a new account can run code

The fix for a new recipient is a higher gas limit, but that gas is not reserved for creating the account. If the
recipient has code by the time the transaction is included, there is no account to create, and the gas goes to the
recipient's code instead. With a limit of 204,600, that code gets up to 183,600 gas (180,600 behind a cold 7702
delegation). With 21,000 it got none: any code that does more than `STOP` ran out of gas at once.

The recipient can usually decide this. Between the time the limit is chosen (or the transaction signed) and the
time it is included:

- an EOA owner can add a 7702 delegation to any contract;
- a counterfactual smart account or another CREATE2 address can be deployed;
- a recipient who planned this can give their address any code they like.

The code then runs paid by the sender, with `msg.sender` and `tx.origin` set to the sender's wallet, before the
sender can react:

- it can spend the sender's gas on something useful to the recipient, such as tokens whose only cost to mint is
  gas, or new storage slots
- it can call other contracts that authorize by `tx.origin`, as the sender's wallet. Therefore, the sender's
  wallet must not control any `tx.origin`-based authorizations. This is achieved if the sender's wallet holds
  only ETH.

## Other things to consider

- **Always estimate, or compute by recipient.** 21,000 if the recipient exists and has no code; 204,600 if it does
  not exist; 24,000 plus its code if it is delegated with EIP-7702. A non-existent address can only start to exist,
  so an estimate for a new address stays high enough. An EOA can still add a 7702 delegation between estimate and
  inclusion, so keep a margin.
- **21,000 is no longer the minimum either.** A zero-value transaction to another account costs 15,000 and a
  transfer to yourself 12,000. Code that adds a fixed 21,000 as the transaction base cost, for example in gas
  refunds, now pays the wrong amount (see INV-5).
- **Related:** INV-4 (fixed gas amounts, including ETH sent by contracts to new addresses, which costs 183,600 more)
  and INV-8 (the same mechanism: a runtime charge turns an underfunded transaction into a mined failure instead of
  an invalid transaction).
