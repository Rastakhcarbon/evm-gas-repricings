---
title: Gas repricing Breakout Room \#1 slides
---

<style>
.reveal {
  font-size: 36px;
}
</style>

# Repricings as the silent hero

## Gas repricing Breakout Room \#1

### Maria Silva

#### October 29th, 2025

[Slides](https://notes.ethereum.org/@misilva/SyU4vSyy-g)

---

:pencil2:

## Why do repricings in Glamsterdam?

---

### :one: They directly impact scalling through harmonisation

- Compute operations do no spent the same MGas/s → [Opcodes Benchmarking](https://grafana.observability.ethpandaops.io/d/feo4ronhsqv40d/opcodes-benchmarking?orgId=1&from=now-30d&to=now&timezone=browser&var-posgreSQL=benuragv7iuwwb&var-ClientName=besu&var-TestTitle=EcRecover%20precompile&var-TestTitle=EcRecoverCACHABLE&var-TestTitle=EcRecoverUNCACHABLE&var-TestTitle=EcRecoverUNCACHABLE2&refresh=auto)
- State creation does not have the cost, depending on the operation → [Bloating Technique Ranking](https://cperezz.github.io/bloatnet-website/bloating.html)
- Data from access lists are free (i.e. users only pay for state access and growth)

---

### :one: They directly impact scalling through harmonisation

**We can remove single bottlenecks with broad repricings**

<img src="https://notes.ethereum.org/_uploads/HJk0jSJk-g.png" alt="bottlenecks_diagm" height="200"/>

---

### :two: They unlock latent scaling in both headliners

- ePBS will give more time in the slot for execution and data propagation. Also, we have more time for execution than data because of the PTC deadline.
- BALs will allow for parallel execution and state root calculation, but add a data cost to state operations

---

### :two: They unlock latent scaling in both headliners

- Thus:
    - Relative cost of state growth to burst resources must increase
    - Relative cost of data to compute and state access must increase

    
![](https://notes.ethereum.org/_uploads/Skl5IuYAxx.png)

---

👷‍♀️

## How to reprice in Glamsterdam?

---

🤯

### We have 16 EIPs in the Repricing Meta EIP!

1. Broad harmonisation
   - Compute, State Access, State Growth, and Intrinsic costs
2. Pricing extension
   - Data, State Access, Transient Storage, Memory, Refunds
3. Supporting
   - Rounding errors fix, multidimensional metering

---

🤔

### How to think about all these EIPs?

- Prioritize broad harmonisations:
  - We need to harmonize compute, state access, state growth, and data.
- Add pricing extensions to address gaps and enhance scaling:
  - Refunds, Size-Based `SSTORE`, transient storage, rounding errors, etc.

---

## On the docket today (5 min each!)

- [EIP-2780](https://eips.ethereum.org/EIPS/eip-2780): Reduce intrinsic transaction gas by @benaadams
- [EIP-7973](https://eips.ethereum.org/EIPS/eip-7973): Warm Account Write Metering @misilva73
- [EIP-8032](https://eips.ethereum.org/EIPS/eip-8032): Size-Based Storage Gas Pricing by @gballet
- [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037): State Creation Gas Cost Increase by @misilva73
- [EIP-8038](https://eips.ethereum.org/EIPS/eip-8038): State-access gas cost update by @misilva73
- [EIP-8058](https://github.com/ethereum/EIPs/pull/10585): Contract Bytecode by Deduplication Discount by @CPerezz
- [EIP-2926](https://eips.ethereum.org/EIPS/eip-2926): Chunk-Based Code Merkleization by @gballet

---

# EIP-7973: Warm Account Write Metering

## Gas repricing Breakout Room \#1

---

### Motivation

- When the same account is updated multiple times in the same transaction, the state root calculation can be batched and all updates can be done in the same calculation.
- However, users still pay the full cost of `GAS_CALL_VALUE` or `GAS_STORAGE_UPDATE`.
- Making warm account writes cheaper increases throughput for transactions that update the same account multiple times.

---

### Proposal

- The first account update pays `GAS_COLD_ACCOUNT_WRITE_COST`.
- The following account updates in the same transaction pay `GAS_WARM_ACCOUNT_WRITE_COST`
- We check the warm/cold status by tracking whether one of the account fields (`nonce`, `value`, `codehash`) is changed.
- Impacts `CREATE`, `CREATE2`, `*CALL`, and `SSTORE`.

---

🐈

## Comments?

---

# EIP-8037: State Creation Gas Cost Increase

## Gas repricing Breakout Room \#1

---

### Motivation

- Without any changes, higher throughput will lead to more state growth.
- Median state growth is currently at ~74 GiB per year
- At 100M block limit, this would grow t0 ~197 GiB per year → less than 2.5 year to reach 650 GiB
- Also, state creation is not harmonized → new slots are more expensive than new code

---

### Proposal

- Sets a unit cost per new state byte, targeting an average state growth of **60 GiB per year** at a block gas limit of **300M gas units**.
- Contract deployments get a 10x cost increase; New slots get a 3x increase; New accounts get a 8.5x increase.
- To avoid limiting the maximum contract size that can be deployed, it also introduces an independent metering for code deposit costs.

---

🐈

## Comments?

---

# EIP-8038: State-access gas cost update

## Gas repricing Breakout Room \#1

---

### Motivation

- State access has been deteriorating due to the increasing state size.
- State access is a known bottleneck for higher throughput.
- BALs and ePBS will impact the relative cost of state access to data and compute.

---

### Proposal

- Broad repricing of all state access gas parameters, based on the most recent benchmark data.
- Impacts `SSTORE`, `SLOAD`, `*CALL` opcodes, `BALANCE`, `SELFDESTRUCT` and `EXT*` opcodes.
- Assumes a worse-case contract size → not fully unlocking state access scaling

---

🐈

## Comments?
