# INV-10: `eth_estimateGas` no longer returns enough gas for the transaction to do its work

**Type:** Price and Accounting

`eth_estimateGas` returns a gas limit at which the transaction does not revert. It does not check that every call
inside the transaction succeeded. When a contract catches a failed call and carries on, the estimate can be a gas at
which that call ran out of gas: the transaction is mined with status 1, and the failure is recorded, often for good.
This could happen before the fork, but only when the inner call needed about 64 times more gas than its caller
spends after it. After the fork one new storage slot can be enough. And for a call whose fixed gas limit is too small
for its new storage costs, no gas limit up to 2^24 does the work, while the estimator still returns one.

## Description

An estimator runs the transaction at different gas limits and returns the smallest one, within a tolerance, at which
it does not revert. geth (as of November 2026) works like this:

1. It runs the transaction once at a high limit: the lowest of the request's gas (or else the block gas limit), the
   node's cap for RPC calls (50,000,000 by default) and what the sender can pay at the fee cap, if one is given.
2. It takes the gas used in that run as the lower bound, and tries that amount plus about 1/63 as a first guess.
3. It bisects between the two and stops when they are within 1.5% of each other.

Every run counts as a success if the transaction itself does not revert. A call inside it that ran out of gas and
was caught (`try`/`catch`, a low-level call whose result is only logged or stored) is a success.

Close to the gas the work needs, a starved inner call leaves its caller 1/64 of the gas it forwarded. Before the
fork that was usually too little to finish, so the transaction reverted and the estimator went higher. The lowest gas
that did not revert was then also the lowest gas that did the work.

After the fork:

- **The band is wider.** Inner calls that create state need far more gas, so the 1/64 their caller keeps is often
  enough to finish and record the failure ([INV-6](inv-6.md)). There is a band of gas limits, up to hundreds of
  thousands wide, in which the transaction succeeds without doing its work, and the estimator can land in it.
- **A fixed inner gas limit cannot be estimated.** If a call gets a fixed amount of gas that its new storage costs
  exceed ([INV-4](inv-4.md)), it fails at every gas limit up to 2^24. The estimator returns a gas anyway, because the
  transaction does not revert. Only a limit above 2^24, where a reservoir pays the storage, does the work. No margin
  on the estimate helps.
- **Runs above 2^24 do not match each other.** In geth, the first run of `eth_estimateGas` is usually above 2^24 and
  gets a reservoir, so a capped call can succeed in that run and fail at the gas the estimator returns. `eth_call`,
  `debug_traceCall` and `eth_simulateV1` (without validation) give all the gas to execution instead, with no
  reservoir, so the same call fails there even at gas limits where the real transaction would succeed. Below 2^24 all
  of them match the real transaction.

Estimators do not all land in the same place:

- geth's starts from the gas used and lands near the top of the band. The work needs more gas than it uses, because
  each level of calls keeps back 1/64 of what it forwards, and geth searches only a little above the gas used. So the
  miss grows with each call level. For the example below, on Amsterdam, it was up to 0.3%
  with the target called directly and up to 1.7% through a proxy.
- An estimator that searches the whole range for the lowest gas that does not revert lands at the bottom of the band.
  For the example below that was up to 5.8 times too little, and more for bigger inner calls.

Bridges, governance and multisig executors, oracle callbacks, staking contracts and account-abstraction bundlers
often catch failed calls. An honest sender who uses the estimate, or the estimate plus a small margin, can get a mined
transaction whose work has failed.

## Vulnerable example

A bridge inbox that anyone can relay to, and the relayer that sends the messages.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract MessageInbox {
    mapping(bytes32 => bool) public delivered;

    event Delivered(bytes32 indexed id, bool success);

    // Anyone can relay a message once it is proven.
    function relay(address target, bytes calldata data, uint256 nonce) external returns (bool ok) {
        bytes32 id = keccak256(abi.encode(target, data, nonce));
        // ...verify the message against the source chain...
        require(!delivered[id], "already delivered");
        delivered[id] = true;                         // the message cannot be relayed again

        (ok, ) = target.call(data);                   // all gas forwarded; a failure is final
        emit Delivered(id, ok);
    }
}
```

```ts
import { Contract, JsonRpcProvider, Wallet } from "ethers";

const provider = new JsonRpcProvider(RPC_URL);
const relayer = new Wallet(RELAYER_KEY, provider);
const inbox = new Contract(INBOX, [
  "function relay(address target, bytes data, uint256 nonce) returns (bool)",
  "event Delivered(bytes32 indexed id, bool success)",
], relayer);

async function relayMessage(target: string, data: string, nonce: bigint) {
  // Assumption: the estimate is enough gas for the message to be delivered.
  const gasLimit = await inbox.relay.estimateGas(target, data, nonce);
  const tx = await inbox.relay(target, data, nonce, { gasLimit });
  await tx.wait();                                // status 1: the message counts as delivered
}
```

Run in the execution specs (EELS) with a port of geth's estimator, for a message whose target sits behind a proxy
and creates two new storage slots:

| | Osaka | Amsterdam |
|---|---|---|
| Gas that does not revert but loses the message | 100,993 to 104,151 | 183,624 to 375,874 |
| Lowest gas that delivers | 104,152 | 375,875 |
| geth's estimate | 105,054: delivered | 374,021: **lost for good** |

The relayer's transaction succeeds, `Delivered(id, false)` is emitted and the message can never be relayed again.
Over 18 such messages (targets creating 0 to 8 new slots), geth's estimate lost 8 on Osaka and 14 on Amsterdam. With
the target called directly instead of through a proxy, it lost none on Osaka and 8 on Amsterdam.

With a fixed callback gas no estimate below 2^24 can be right. For the oracle in [INV-4](inv-4.md), whose callback of
100,000 gas writes two new slots, geth's first run at 50,000,000 gas delivers the callback. Its estimate, 241,726,
does not, and neither did any of 286 gas limits sampled from there up to 2^24. The lowest gas that delivers is
16,897,627, just above 2^24.

## Fixed example

The relayer checks the outcome at the gas it sends with, not just that the transaction does not revert.

```ts
import { isError, type TransactionReceipt } from "ethers";

const TX_GAS_CAP = 1n << 24n;                     // above this a reservoir exists, and eth_call does not model it

// The lowest gas, to within 1%, at which relay() delivers the message. Assumes that more gas never turns delivery
// into failure, which is not true of code that branches on gasleft().
async function deliveringGas(target: string, data: string, nonce: bigint): Promise<bigint> {
  // eth_call at a given gas limit. Below 2^24 it matches the transaction.
  const delivers = (gasLimit: bigint): Promise<boolean> =>
    inbox.relay.staticCall(target, data, nonce, { gasLimit }).catch((e) => {
      if (isError(e, "CALL_EXCEPTION")) return false;            // reverted or ran out of gas at this limit
      throw e;                                                  // a network error: do not guess
    });

  // Not delivered at 2^24: more gas below 2^24 does not help. The target fails for another reason, or it has a
  // fixed gas limit that only a reservoir can pay for. Handle it under your failure policy, not here.
  if (!(await delivers(TX_GAS_CAP))) throw new Error("not delivered at 2^24 gas");

  // The transaction does not revert at the estimate. If it does not deliver there, double, then bisect.
  let hi = await inbox.relay.estimateGas(target, data, nonce);
  if (hi > TX_GAS_CAP) hi = TX_GAS_CAP;
  let lo = hi;
  while (!(await delivers(hi))) {
    lo = hi;
    hi = hi * 2n < TX_GAS_CAP ? hi * 2n : TX_GAS_CAP;
  }
  while (hi - lo > hi / 100n) {
    const mid = (lo + hi) / 2n;
    if (await delivers(mid)) hi = mid;
    else lo = mid;
  }
  return hi;
}

async function relayMessage(target: string, data: string, nonce: bigint) {
  // The margin covers state that changes before inclusion, not the estimator. Stay at or below 2^24: that is
  // what was simulated.
  const withMargin = (await deliveringGas(target, data, nonce)) * 11n / 10n;
  const gasLimit = withMargin < TX_GAS_CAP ? withMargin : TX_GAS_CAP;
  const tx = await inbox.relay(target, data, nonce, { gasLimit });
  const receipt: TransactionReceipt | null = await tx.wait();

  // Still check what happened: the state can change between the simulation and inclusion.
  const delivered = receipt!.logs
    .filter((log) => log.address.toLowerCase() === INBOX.toLowerCase())
    .map((log) => inbox.interface.parseLog(log))
    .find((event) => event?.name === "Delivered");
  if (!delivered?.args.success) throw new Error(`message not delivered in ${tx.hash}`);
}
```

In EELS this search returned a gas that delivers for all 72 messages tested (Osaka and Amsterdam, direct and behind
a proxy, 0 to 8 new slots), at most 1.5% above the lowest one. On the oracle of [INV-4](inv-4.md) it stops at the
check at 2^24 instead of returning a number.

This protects the relayer, not the inbox. Anyone else can still relay with too little gas and lose the message. The
contract needs its own fix.

If the function does not return its outcome, simulate with `eth_simulateV1` and read the event, or trace the call
(`debug_traceCall`) and check the inner call for an error.

## Common pitfalls

- **"The estimate succeeded, so the transaction works."** It shows only that the transaction does not revert. Check
  the outcome: a return value, an event, a state change.
- **A fixed margin on the estimate.** A margin that covers geth's miss on one contract does not cover another, does
  not cover an estimator that lands at the bottom of the band, and does not help at all when a call has a fixed gas
  limit.
- **Estimates made before the fork.** Cached or stored estimates, and gas values computed from them, are too low after
  the fork. Re-estimate, and check the outcome at the new value.
- **"It worked at the cap."** Above 2^24, simulations do not match the transaction or each other: geth's
  `eth_estimateGas` gives the run a reservoir, while `eth_call`, `debug_traceCall` and `eth_simulateV1` without
  validation give all the gas to execution. An `eth_call` without a gas field runs at the node's cap, above 2^24.
  Simulate at the exact gas you will send, below 2^24.
- **Your own search loops.** Code that finds a gas limit by trying `eth_call` at smaller and smaller values, on-chain
  or off-chain, has the same problem if it only checks for a revert.

## Other things to consider

- **Which code is affected.** Any transaction to a contract that catches a failed call: relayers, keepers,
  executors of governance and multisig transactions, bundlers, bridge finalizers, wallets sending to such contracts.
  Contracts that revert when an inner call fails are not affected.
- **Fix the contract where you can.** Check the other invariant descriptions.
- **Monitor receipts, not just status.** Read the events that report the outcome of the inner work, and alert or
  retry when it failed.
