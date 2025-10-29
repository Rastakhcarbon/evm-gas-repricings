# EVM resources and gas costs 

#### Maria Silva, September 2025

:::info
:information_source: **TLDR**

With the current slot and EVM design:
- **Compute** opcode can be cheaper, while some precompiles must be more expensive. [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904) addresses these mispricings. This is a repricing EIP.
- **Eth transfers** can be cheaper. [EIP-2780](https://eips.ethereum.org/EIPS/eip-2780) addresses this mispricing. It also harmonises account creation costs with `*CALL` operations.  This is a repricing EIP.
- **LOG receipts** are a bottleneck to 85M. [EIP-7975](https://eips.ethereum.org/EIPS/eip-7975) addresses this bottleneck. This is a P2P protocol improvement.
- **Calldata** is a bottleneck for 105M. [EIP-7976](https://eips.ethereum.org/EIPS/eip-7976) and [EIP-7981](https://eips.ethereum.org/EIPS/eip-7981) increase data costs. [EIP-7934](https://eips.ethereum.org/EIPS/eip-7934) could also be used to directly address the bottleneck by creating a limit on the block size. 
- **State access** (mostly writes) on large contracts is a bottleneck to 100M. [EIP-8032](https://github.com/ethereum/EIPs/pull/10451) and [EIP-8038](https://eips.ethereum.org/EIPS/eip-8037) solve this bottleneck. EIP-8032 is both a new feature and repricing EIP, while EIP-8038 is a pure repricing. We are also working on a harmonisation EIP for all state access operations.
- **State size** will become a concern in less than 2.5 years, under the current schedule for gas limit increases. The plan is to propose an increase in state creation costs by ~10x. This can be done with [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037). [EIP-2926](https://eips.ethereum.org/EIPS/eip-2926) and [EIP-8032](https://github.com/ethereum/EIPs/pull/10451) also touch on state creation pricing. 
- **History size** is not a concern in the short or medium term. The current limits imposed by the data resource are already limiting the level of history growth we can observe.

With ePBS and BAL's:
- TBD
:::

Ethereum’s slot-based structure introduces a strict temporal constraint: all attestations must be processed, aggregated, and propagated within a single 12-second slot. This makes time a fundamental resource. To maintain network health, validators must execute blocks, validate them, and gossip attestations quickly enough to avoid missed slots and penalties.

On the other hand, [EIP-7870](https://eips.ethereum.org/EIPS/eip-7870) sets clear guidelines for the minimum hardware requirements for validators, block builders, and full nodes. This standard not only limits the amount of disk and memory available to nodes, but also restricts the computational and bandwidth capacity available to process blocks and propagate data, respectively.

Each type of resource contributes differently to these bottlenecks:
- **Execution time** affects how quickly validators can process blocks and produce attestations.
- **Data size** influences how long it takes to propagate blocks and blobs across the peer-to-peer network.
- **Memory size** determines how efficiently nodes can handle concurrent workloads without triggering garbage collection or cache eviction. It also affects the minimum hardware requirements that proposers and attesters must meet to fulfill their responsibilities.
- **State growth** impacts the performance of state access (and thus execution time). It also increases the storage requirements for proposers and attesters.
- **History growth** impacts the storage requirements for nodes, especially for archival and full nodes. 

Thinking about these resources is key to understand where to focus on for scaling Ethereum. Concretely, we should understand which resources are currently bottlenecking our scalling and how severe is the mismatch between each resource pair. Also, how will the relative cost of different resources change once ePBS and BALs are introduced in Glamsterdam? 

## Bottlenecks overview

There are two ways of thinking about bottlenecks. First, we have worst-case blocks. These are one-time blocks specifically designed to stress a particular resource. They usually involve filling a block as much as possible with specific operations. This is a good benchmark for burst resources, such as compute and state access. 

However, this benchmark is not realistic for persistent usage resources such as state growth. If someone started to grow a state by filling many blocks to capacity, the base fees would increase, and the attack would become increasingly expensive to continue. Instead, we need to consider more realistic usage patterns for state growth. For instance, filling a block up to the target for a consistent amount of time, or even considering the historical state growth rates.

In the following subsections, we will consider both types of bottleneck reasoning methods.

### Burst resources

There is an ongoing effort to build and measure these worse-case blocks. The tests are in [EEST](https://github.com/ethereum/execution-specs/tree/forks/osaka/tests/eest/benchmark) and the benchmarks are run using Nethermind's [gas-benchmarks tool](https://github.com/NethermindEth/gas-benchmarks/tree/main). This [grafana dashboard](https://grafana.observability.ethpandaops.io/d/feo4ronhsqv40d/opcodes-benchmarking?orgId=1&from=now-7d&to=now&timezone=browser&var-posgreSQL=benuragv7iuwwb&var-ClientName=$__all&var-TestTitle=$__all&refresh=auto) has the latest results. This analysis uncovered the [following bottlenecks](https://github.com/NethermindEth/eth-perf-research/blob/main/README.md#gas-limit-blockers) for scalling until 105M gas units:

- **Modexp precompile**. [EIP-7883](https://eips.ethereum.org/EIPS/eip-7883) triples the cost of Modexp and introduces an aggressive scale for inputs exceeding 32 bytes. This EIP is introduced in Fusaka.
- **Receipts sizes.** [EIP-7975](https://eips.ethereum.org/EIPS/eip-7975) adds a facility for paginating block receipt, thus solving the 10MiB message size limit commonly applied in clients. This is a protocol-level improvement and does not require a hard fork.
- **Block sizes in calldata heavy blocks** - [EIP-7934](https://eips.ethereum.org/EIPS/eip-7934) introduces a protocol-level cap on the maximum RLP-encoded block size to 10 MiB, thus limiting this bottleneck directly. 
- **Point evaluation Precompile.** [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904) proposes a overhaul of gas prices, focussing computational complexity. Even though precompiles are not yet included, we plan to add precompiles to this EIP and propose it to Glamsterdam.
- **State access for heavy contracts.** Analysis is still underway. However, we aim to propose [EIP-8032](https://github.com/ethereum/EIPs/pull/10451) for Glamsterdam.
- **Block size of calldata heavy transactions**. There are two EIPs proposed for Glamsterdam that further increase data costs, namely, [EIP-7976](https://eips.ethereum.org/EIPS/eip-7976) and [EIP-7981](https://eips.ethereum.org/EIPS/eip-7981). [EIP-7934](https://eips.ethereum.org/EIPS/eip-7934) could also be used to durectly address the bottleneck by creating the limit on the block size. 

Finally, from the same exercise, we know that most compute opcodes and ETH transfers are relatively underpriced. The following swarmplot shows the million gas per second achieved by the various test blocks on the worst client. Each point is a test block generated by [EEST](https://github.com/ethereum/execution-specs/tree/forks/osaka/tests/eest/benchmark). 

![](https://notes.ethereum.org/_uploads/S1Ldi6Phxg.png)

As we can observe, there is a large variation between the MGas/s achieved by each test, with some cases being 2-10x faster. This means we could make these operations 2-10x cheaper. [EIP-2780](https://eips.ethereum.org/EIPS/eip-2780) and [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904) should address these mispricings.

#### A quick note on state access

State access is the area at the moment with the most heterogenous set of EIPs. Because of this, we will go over the parameters and the EIPs can could change them.

The costs for state access are controlled by the following parameters:

```python
GAS_STORAGE_UPDATE = Uint(5000)
GAS_STORAGE_CLEAR_REFUND = Uint(4800)

GAS_COLD_SLOAD = Uint(2100)
GAS_COLD_ACCOUNT_ACCESS = Uint(2600)
GAS_WARM_ACCESS = Uint(100)
```

`GAS_STORAGE_UPDATE` and `GAS_STORAGE_CLEAR_REFUND` are only used on `SSTORE`. This opcode is being repriced in [EIP-8032](https://github.com/ethereum/EIPs/pull/10451), so there would be the actual place to price these parameters. However, given that this EIP requires a trie change, we are also planning to propose a simple reprice-only EIP to update `SSTORE`.

We can fit the reprice of `GAS_COLD_ACCOUNT_ACCESS` in [EIP-2780](https://eips.ethereum.org/EIPS/eip-2780) as we plan to make a full standardisation of account access costs across ETH transfers and CALLs.


`GAS_WARM_ACCESS` is being repriced in [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904). In this context, it would also make sense to price `GAS_COLD_SLOAD` in the same EIP.


### Persistent resources

#### State

As of May 2025, the current database size in a Geth node dedicated to state is ~340 GiB. After the increase in gas limit from 30M to 36M gas units, the median size of new state created each day doubled, from ~102 MiB to ~205 MiB. All else staying equal, we expect further increases in state creation as new gas limits are set.

![](https://notes.ethereum.org/_uploads/HJl-oefEnxx.png)

The bloatnet initiative uncovered critical bottlenecks in state access patterns at a state size of 650GB. They observed a direct correlation between state size and degradation in sync time. Specifically, at this point, nodes will experience longer initial sync times, longer state I/O times, and increased memory requirements.

At a 60M gas limit (and a proportional increase in new state per day of 1.7x), we would see a daily state growth of ~349 MiB and a yearly state growth of ~124 GiB. Similarly, at a 100M gas limit, the state would grow at a rate of ~ 553 MiB per day and 197 GiB per year. This level of state growth would give us less than 2.5 years until the size of the state database exceeds the 650 GiB threshold.

Besides introducing code-chunking in an MPT context, [EIP-2926](https://eips.ethereum.org/EIPS/eip-2926) proposes to increase the costs of each new contract byte from 200 to 500 gas units. However, this increase may be too low. If we were to target an **average state growth of 60 GiB per year** at a block gas target of 50 million units (i.e. a block limit of 100M), these paramater should be increased to 2,040. In addition, it does not reprice other state creation operations, such as new slots and new accounts. 

This [recent draft](https://notes.ethereum.org/n_rUJi-yTaq4P_CYRjCjiQ) makes another proposal to address state growth more holistically. It also provide the rational for setting the cost of each new contract byte to 2,040. We plan to propose this as an EIP for Glamsterdam. But there is still some work to do.

##### History

As of May 2025, the current database size in a Geth node dedicated to history is ~901 GiB. Compared with state, history experienced a smaller increase in new DB space created each day when the gas limit was changed from 30M to 36M gas units. Specifically, the median size of new history created each day increased from ~420 MiB to ~532 MiB. This is a 26% increase, which is more in line with the 20% increase in the gas limit.

![](https://notes.ethereum.org/_uploads/S1p7J0wngl.png)

If we were to assume a proportional increase under the 100M gas units limit, we would observe an additional ~1.4 GiB per day of history, which results in ~527 GiB of new history per day. This [article from Paradigm](https://www.paradigm.xyz/2024/03/how-to-raise-the-gas-limit-2) reports that if history size grew past 1.8 TiB, many nodes would be forced to upgrade their storage. With ~527 GiB added per year, we would only reach this threshold in 26 years. Also, it would take 13 years to reach 1 TiB of history. 

Given that the work on history expiry is progressing well and that it does not require a hard-fork, we can confortable raise the gas limit without history growth becoming a concern anytime soon.

## Relative costs after Glamsterdam

:::warning
:construction_worker: Work in progress

Key changes to consider:
- ePBS:
    - Time for payload propagation plus execution increases from 4s to 6s (1.5x).
    - Header propagation and execution is done at separate times from payload propagation and execution. This part is not likely a bottleneck
    - No changes on memory not persistent resources
- BALs:
    - Execution can be paralelized, thus giving a 4x-3x speedup in execution time. *Is this speedup correct?*
    - State transition can be done independently from execution. *What is the speedup?*
:::


## Setting gas costs

To set the gas costs os each EVM operation (which includes opcodes, precompiles, and transaction intrinsic operations), we propose a simple linear transformation between the resource usage of the operation and a desired max limit. This is the same logic used to set the costs for the first EVM gas model. In short, the process is as follows.

1. Measure resource utilization of each EVM operation. The outcome is a mapping of each operation to a vector of resource utilization metrics $U^\text{op}=(u_1^\text{op}, ..., u_6^\text{op})$, with the following structure (some metrics will be zero):
    - Compute: CPU time in seconds.
    - State Access: Time spent on I/O or count of disk I/O operations when accessing the trie (e.g., SLOAD/SSTORE).
    - Data: Block size in MB.
    - Memory: Short-term memory allocation during execution, i.e., RAM usage.
    - State Growth: Delta in disk size of the state trie before and after execution.
    - History Growth: Delta in disk space used for historical receipts, logs, etc.
3. Define resource limits in the same units used to measure the resource utilization of each EVM operation. For example, CPU time in seconds for computation. The outcome is a vector of limits $L = (l_1, ..., l_6)$.
4. Pick the desired gas limit (in number of gas units) for the repricing, $G$.
5. Compute the gas cost vector of each operation as $C^\text{op} = (c_1^\text{op}, ..., c_6^\text{op})$, where $c_i^\text{op} = \frac{G \cdot u_6^\text{op}}{l_i}$.
6. Compute the total gas cost of each operation as $c^\text{op} = \sum_{i=1, ..., 6} c_i^\text{op}$


#### How to measure resource utilization?

Similar to the methodology in the [Gas Cost Estimator](https://github.com/imapp-pl/gas-cost-estimator/blob/master/docs/report_stage_ii.md) from Jacek Glen and Lukasz Glen, we generate synthetic blocks that isolate and stress individual EVM operations (opcodes, precompiles, and ETH transfers). These blocks will follow the [EEST](https://github.com/ethereum/execution-spec-tests?tab=readme-ov-file) framework.

To benchmark a single operation, different blocks are created by varying the number of single operations executed and by changing the inputs to the operation (if it takes inputs). For state operations, we also vary the state size, leveraging the work done by the [Bloatnet initiative](https://cperezz.github.io/bloatnet-website/index.html). We also run the same blocks for all the client implementations.

To collect the resource utilization metrics, we use [Nethermind's Performance Benchmarking tool](https://github.com/NethermindEth/eth-perf-research/tree/main). Once the data is collected, we estimate the resource utilization vectors using [linear regression](https://en.wikipedia.org/wiki/Linear_regression). The dependent variable for each regression model is the resource utilization metric $u_i^\text{op}$. In contrast, the independent variables are the number of operations executed, the client implementation, and the inputs to the dynamic cost (if they exist).


#### What about dynamic costs?

Some operations have a dynamic component to their costs. For example, the `EXP` opcode has a gas cost that grows linearly with the exponent's byte size. For these operations, their utilization vectors $U^\text{op}$ must also be dynamic. This means that we need to benchmark the operation with a range of inputs to assess how the resource costs vary with different inputs.

#### What about differences between client implementations?

Ultimately, we require a single cost model that is independent of the client's implementation. Although the decision may be different in a specific situation, the general rule is to take the worst client result that cannot be optimised. This means that the worst-performing client on a given operation and resource combination will define the cost of that operation for the entire network. This is the safer and most conservative choice.

#### A note on Data

We do not require dedicated benchmarks for measuring raw data size. Call data and blobs already have clearly defined metering in MiB, and transaction payload sizes can be derived from historical blocks. Thus, we can collect this data if the lift is not significant, but it is definitely a nice-to-have.

The remaining task is translating data size into expected propagation time. This can be estimated using standard bandwidth assumptions and refined with empirical propagation data gathered from ongoing work around the 6-second slot proposal.

PeerDAS introduces a nuance: not all blobs are downloaded by each attester. As a result, effective bandwidth usage per node decreases linearly with blob redundancy. This should be reflected in any future metering adjustments related to blob data.

### What to use as an anchor? 

Whenever we decide to do a major change in gas proces, a key challenge arrises - the choice of the anchor. An anchor is a metric that allows us to convert resource utilization into a standardized gas cost. 

For compute and state access operations (i.e. execution time), this anchor will induce a specific million gas per second rate. For instance, if we anchor on `EcRecover` at the current price (3000 gas units), this induces a rate of 175Mgas/s. For state growth and history growth, the anchor induces a average DB growth per million gas. This was the rationale we used to set the costs for [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037). Finally, for data, prices have a direct translation into block size in kb by gas unit.


#### Execution time

Given that the current gas model uses integer gas units, there is a trade-off in the choice of an anchor - with higher Mgas/s, we get more expressive prices as the rounding errors are lower. However, this comes at the cost of all operations becoming more expensive, which means that we will have to further increase the gas limit to process the same amount of transactions. This also means we need to further raise the cost of other resources, such as state and data, to make the relative costs between resources more balanced. In other words, we need to rebase the concept of gas.

The following plot shows the average rounding errors of compute operations for different anchors, assuming the estimated execution times from [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904) empirical analysis.

![](https://notes.ethereum.org/_uploads/Hy0JfNq6ge.png)

As we can see, with an anchor of 50Mgas/s (which would result in the gas limit of 100Mgas), we would get an average rounding error of 59.3%. This means that we could be 59.3% cheaper on average if we didn't have to round to integer gas units. Only after a 400Mgas/s anchor do we get to reasonably low rounding errors.

This means we either propose to convert the gas model to a fractional model (as in [EIP-2045: Particle gas costs for EVM opcodes](https://eips.ethereum.org/EIPS/eip-2045)) or need to rebase it by making everything more expensive. As an example, if we assume that the fastest operation costs 5 gas (which is `JUMPDEST`), this would indicate an anchor of ~3000Mgas/s, representing a 100x increase over the current 60Mgas block limit. Assuming that ePBS and BALs provide a further scaling of compute and I/O of 3x-6x, we are looking at a rebase of 300x to 600x in the current gas limit. Of course, we should scale down the base fee at the fork boundary to account for this rebase.

**Cons of fractional gas**

- New gas counter adds complexity to the EVM runtime
- Harder to implement

**Cons of a 600x rebase**

- Higher likelyhood of breaking hardcoded values -> more analysis on this is needed.
- Changing the meaning of gas will likely have an impact on UX, at least in the short-term, as users expectations of gas limits will need to change. Not sure how much of a problem this is, as wallets should help with this.
- Not sure how this will impact the Scale L1 narrative. So far, we have been using the gas limit as a measure of scalability, which will not longer make sense after the rebase.

### Testing gas price changes

From Pari:

>- Set gas cost values
>- STEEL allows these values to be configured in a fork
>- EEST generates tests and uploads on a branch or with tag
>- gas-benchmark points to custom EEST fixtures
>- Client makes changes in codebase and push to a repricing branch
>- Results show up on custom dashboard on grafana

## Concerns about changing gas costs

- What can break?
- How hard is it to change gas costs?
- When is backwards compatibility a concern?

:::info
:information_source: Check Security considerations from https://eips.ethereum.org/EIPS/eip-2929

Also check [this discussion](https://ethereum-magicians.org/t/eip-2045-particle-gas-costs/3311/13?u=misilva73)
:::

## Repricings in Glamsterdam

There are currently 16 Repricing EIPs proposed for Glamsterdam. In this section, we discuss how they support L1 scaling, bundling options and how we think about prioritization.

### Overview

There are three types of EIPs within the repricing scope:

1. **Broad harmonisation**. These EIPs reprice a class of operations with the goal of harmonizing them and removing single bottlenecks.
2. **Pricing extension**. These EIPs make targeted changes to a specific opcode or component of the gas model, usually coupled with a new mechanism.
3. **Supporting**. These EIPs are not directly doing a repricing, but instead introduce a change that support other repricing EIPs or enhance the scalability potential of repricings.

### Bundle options

#### Minimal Repricing

The Minimal Repricing includes a broad repricing of compute and state operations, thus fixing the majority of mispriced opcodes, aligning the cost of ETH transfers with the relevant compute and state operations, and updating data costs. Specifically, it:

- Harmonises the costs of compute operations with [EIP-7904](https://eips.ethereum.org/EIPS/eip-7904), thus removing single bottlenecks and allowing a higher throughput of compute operations.
- Updates the costs of state access operations with [EIP-8038](https://eips.ethereum.org/EIPS/eip-8038), thus removing state access as a scaling bottleneck.
- Increases the cost of state creation operations with [EIP-8037](https://eips.ethereum.org/EIPS/eip-8037), thus allowing for higher transaction throughput without accelerating state growth.
- Aligns the cost of ETH transfers with the rest of compute and state operations with [EIP-2780](./eip-2780), thus increasing the throughput of ETH transfers and aligning their cost with similar operations.
- Charges access lists for their data footprint with [EIP-7981](https://eips.ethereum.org/EIPS/eip-7981), thus lowering the worst-case block size achieved through call data.
- Increases call data cost for data-heavy transactions with [EIP-7976](https://eips.ethereum.org/EIPS/eip-7976), thus lowering the worst-case block size achieved through call data.

This is the minimum viable bundle that allows us to remove single bottlenecks while leveraging the gains obtained from ePBS and BALs.

On the one hand, we are doing a broad repricing of key burst resources (compute, state access, and data) that will take into account the relative performance of operations within individual resources and the changes in available execution time and data propagation delivered by ePBS and BALs.

On the other hand, we are increasing the cost of state growth to ensure its growth rate does not exceed unfeasible levels, given the increased transaction throughput. State growth is the most pressing of the persistent resources to address, as it affects hardware requirements and state access (and thus execution time). Additionally, there is no clear path yet to a long-term solution for state growth, which makes mitigating it even more critical.

#### Extended Repricing

The Extended Repricing includes all changes from the Minimal Repricing plus some pricing extensions that mitigate worst-case blocks or enhanced throughput on the average case. The additions include:

- Excluding refunds from block accounting with [EIP-7778](https://eips.ethereum.org/EIPS/eip-7778), thus mitigating the worst-case block from state access operations. This should be a simple change that tackles a concrete bottleneck.
- Scaling the cost of `SSTORE` with the contract's storage size with [EIP-8032](https://eips.ethereum.org/EIPS/eip-8032), thus allowing smaller contracts to write to their storage more cheaply, while solving the state access bottleneck of large contracts. Compared with [EIP-8038](https://eips.ethereum.org/EIPS/eip-8038), this change improves the throughput of `SSTORE` operations on all contracts smaller than ~8 GB of data by making them cheaper. Even though this EIP includes a change to the trie, given the fact that state access is a key bottleneck, we expect the gains from this change to compensate the added complexity. However, more benchmarking is needed to gather final numbers.
- Decreasing the costs of `TLOAD` and `TSTORE` with [EIP-7971](https://eips.ethereum.org/EIPS/eip-7971), thus improving the usability of transient storage. Pushing more users to leverage transient storage will ease the burden on state access.
- Solving rounding errors from efficient compute operations with [EIP-8053](https://eips.ethereum.org/EIPS/eip-8053) or [EIP-8059](https://eips.ethereum.org/EIPS/eip-8059). On average, transactions are expected to incur 4% of rounding errors on compute operations.

#### Further additions

The following EIPs tackle some mispriced operations. However, they also introduce new mechanisms that need to be further considered.

- Add multi-block state access discounts with [EIP-8057](https://eips.ethereum.org/EIPS/eip-8057).
- Reduce the cost for deploying duplicated contracts with [EIP-8058](https://eips.ethereum.org/EIPS/eip-8058).
- Change the memory expansion cost from quadratic to linear with [EIP-7923](https://eips.ethereum.org/EIPS/eip-7923).
- Add code chunking and update code deployment cost with [EIP-2926](https://eips.ethereum.org/EIPS/eip-2926).
- Add Multidimensional Gas Metering with [EIP-8011](https://eips.ethereum.org/EIPS/eip-8011).
- Add warm account writes with [EIP-7973](https://eips.ethereum.org/EIPS/eip-7973).

### ToDo's

#### Compute (EIP-7904)

- [x] Propose EIP-7904 for Glamsterdam
- [ ] Redo and verify emprical analysis
    - Missing opcodes: `BLOCKHASH`, `PREVRANDAO`, `BASEFEE`, `BLOBHASH`, `BLOBBASEFEE`, `SELFDESTRUCT`
- [ ] Investigate pure compute costs for LOG, RETURN, REVERT, and *COPY family.
- [ ] Investigate the costs of precompiles
- [ ] Investigate execution costs for contract deployment -> impacts init_code_cost and hash_cost
- [ ] Investigate transient storage opcodes ([EIP-7609](https://eips.ethereum.org/EIPS/eip-7609) and [EIP-7973](https://github.com/ethereum/EIPs/pull/9894/files) may be relevant)
- [ ] Align on anchor -> EcRecover, 175Mgas/s, 300Mgas for 2 seconds ?
- [ ] Update parameters and EIP (new prices, anchor explanation, precompiles, improve wording)
- [ ] Run impact analysis -> replay 1 year of blocks with new pricing and analyse revert rates
- [ ] Run EEST benchmarks with new pricing

#### State access

- [x] Propose EIP-8038 for Glamsterdam
- [ ] Run replay benchmark ([more info here](https://notes.ethereum.org/HWoV1k0hTF6Jqw4LitBPqQ?view#Bechmarking-SSTORE-for-depth-based-pricing)) to benchmark state access operations
- [ ] Run statefull EEST benchmarks
- [ ] Collect benchmark data and estimate execution time for all state access operations:
    - SSTORE, SLOAD, EXT* family, *CALL family, BALANCE, CREATE/CREATE2
- [ ] Estimate parameters for state access EIPs and update EIPs (EIP-8032 + EIP-8038)
- [ ] Run impact analysis -> replay 1 year of blocks with new pricing and analyse revert rates
- [ ] Run EEST benchmarks with new pricing

#### State growth

- [x] Fix problem with 16.7M transaction limit for EIP-xxxx
- [x] Add duplicated bytecode discount
- [x] Propose EIP-8037 for Glamsterdam
- [ ] Benchmark new contract wih duplicated code vs. new account
- [ ] Benchmark code duplication logic in contract deplyments -> is the execution overhead being covered by `hash_cost` and `storage_access_cost`?
    - This includes benchmarking `GAS_KECCAK256_WORD`
- [ ] Align costs of EIP-2926 with EIP-8037
- [ ] Benchmark added compute costs for EIP-2926
- [ ] Run impact analysis -> replay 1 year of blocks with new pricing and analyse revert rates ([more info here](https://notes.ethereum.org/HWoV1k0hTF6Jqw4LitBPqQ?view#Uncovering-issues-with-state-creation-gas-increases))
- [ ] Run EEST benchmarks with new pricing


#### Intrinsic costs (EIP-2780)

- [ ] Add cost harmonization for all account creation operations
- [ ] Improve rational section
- [ ] Align costs with state access EIPs
- [ ] Run impact analysis -> replay 1 year of blocks with new pricing and analyse revert rates ([more info here](https://notes.ethereum.org/HWoV1k0hTF6Jqw4LitBPqQ?view#Uncovering-issues-with-state-creation-gas-increases))
- [ ] Run EEST benchmarks with new pricing


#### Data

- [ ] Verify harmonization of data costs


#### General

- [ ] Build general tool for impact analysis with replaying. Can we do this with a gas simulator?
- [ ] We should also look into the memory expansion cost -> is it aligned with our anchors?
