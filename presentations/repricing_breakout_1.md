---
title: Gas repricing Breakout Room \#1 slides
---

<style>
.reveal {
  font-size: 36px;
}
</style>

# Repricing as the hidden champion

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

### :two: They unlock lattent scalling in both headliners

- ePBS will give more time in the slot for execution and data propagation. Also, we have more time for execution than data because of the PTC deadline.
- BALs will allow for paralell excution and state root calculation, but add a data cost to state operations

---

### :two: They unlock lattent scalling in both headliners

- Thus:
    - Relative cost of state growth to brust resources must increase
    - Relative cost of data to compute and state access must increase

    
![](https://notes.ethereum.org/_uploads/Skl5IuYAxx.png)

---
 

## How to reprice in Glamsterdam?
