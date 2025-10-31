# Gas Repricings for Glamsterdam

### Maria Silva, October 2025

In this note, we provide an overview of all the repricing EIPs proposed for Glamsterdam, suggest bundle options, and discuss how to prioritize them.

We will not go into detail about why repricings are important for Glamsterdam. For that, please refer to the [opinion piece](https://notes.ethereum.org/hdqymropR6eFMSXLoz_vTQ) from Caspar and Ansgar. The TLDR is that gas repricings play two key roles in L1 scalability:

1. They directly impact scaling through harmonization.
2. They unlock latent scaling in both headliners (ePBS and BALs).

I want to thank Ansgar and Caspar for the review and comments.

## EIPs and bundle options

There are three types of EIPs within the repricing scope:

1. **Broad harmonisation**. These EIPs reprice a class of operations to harmonize them and remove single bottlenecks.
2. **Pricing extension**. These EIPs make targeted changes to a specific opcode or component of the gas model, usually coupled with a new mechanism.
3. **Supporting**. These EIPs do not directly perform repricings; instead, they introduce changes that support other repricing EIPs or enhance the scalability potential of repricing.

For the complete list of EIPs, refer to the [Meta EIP-8007](https://eips.ethereum.org/EIPS/eip-8007). Next, we discuss options for bundling these EIPs.

### Minimal Repricing Bundle

The Minimal Repricing includes a broad repricing of compute and state operations, thus fixing the majority of mispriced opcodes, aligning the cost of ETH transfers with the relevant compute and state operations, and updating data costs.

It is important to stress that this set of EIPs should be considered together. If we leave specific resources untouched while repricing others, we will introduce new sources of imbalance and bottlenecks into our pricing model.

This bundle includes:

- **[EIP-7904](https://eips.ethereum.org/EIPS/eip-7904): General Repricing.**
  - This proposal harmonizes the costs of compute operations, thus removing single bottlenecks and allowing a higher throughput of compute operations.
- **[EIP-8038](https://eips.ethereum.org/EIPS/eip-8038): State-access gas cost update.**
  - This proposal updates the costs of state access operations, thus removing state access as a scaling bottleneck.
- **[EIP-2780](https://eips.ethereum.org/EIPS/eip-2780): Reduce intrinsic transaction gas.**
  - This proposal aligns the cost of ETH transfers with the rest of the compute and state operations, thus increasing the throughput of ETH transfers and aligning their cost with similar operations.
- **[EIP-7981](https://eips.ethereum.org/EIPS/eip-7981): Increase access list cost.**
  - This proposal charges access lists for their data footprint, thus lowering the worst-case block size achieved through call data.
- **[EIP-7976](https://eips.ethereum.org/EIPS/eip-7976): Increase Calldata Floor Cost.**
  - This proposal increases calldata cost for data-heavy transactions, thus lowering the worst-case block size achieved through call data. Depending on the final parameters for the remaining resources, we may also need to adjust the base cost of calldata for all transactions.
- **[EIP-8037](https://eips.ethereum.org/EIPS/eip-8037): State Creation Gas Cost Increase.**
  - This proposal increases the cost of state creation operations, thus allowing for higher transaction throughput without accelerating state growth.

This is the minimum viable bundle that allows us to remove single bottlenecks while leveraging the gains obtained from ePBS and BALs.

On the one hand, we are doing a broad repricing of key burst resources (compute, state access, and data) that will take into account the relative performance of operations within individual resources, as well as changes in available execution time and data propagation delivered by ePBS and BALs.

On the other hand, we are increasing the cost of state growth to ensure its growth rate does not exceed unfeasible levels, given the increased transaction throughput. State growth is the most pressing of the persistent resources to address, as it affects hardware requirements and state access (and thus execution time). Additionally, there is no clear path yet to a long-term solution for state growth, which makes mitigating it even more critical.

It is important to note that the parameters for these EIPs are not yet finalized. First, benchmarks are still ongoing, especially for state operations. Second, we need to empirically measure the impact of ePBS and BALs on the performance of different operations before settling on the final numbers.

### Recommended Additions

Besides the minimal repricing bundle, we recommend a set of EIPs for their positive impact on scalability and relatively low complexity. They either mitigate worst-case blocks or enhance throughput on the average case. In contrast to the minimal pricing, these EIPs are not bundled and can be considered independently.

The recommended additions are:

- **[EIP-7778](https://eips.ethereum.org/EIPS/eip-7778): Block Gas Accounting without Refunds.**
  - This proposal excluded refunds from block accounting, thus mitigating the worst-case block from state access operations.
  - This should be a simple change that tackles a concrete bottleneck, and as such, this is our top recommendation for addition to the minimal bundle.
- **[EIP-8032](https://eips.ethereum.org/EIPS/eip-8032): Size-Based Storage Gas Pricing.**
  - This proposal scales the cost of `SSTORE` with the contract's storage size, thus allowing smaller contracts to write to their storage more cheaply, while solving the state access bottleneck of large contracts.
  - Compared with [EIP-8038](https://eips.ethereum.org/EIPS/eip-8038), this change improves the throughput of `SSTORE` operations on all contracts smaller than ~8 GB of data by making them cheaper.
  - This EIP does include a change to the trie, which introduces additional complexity. However, given that state access is a key bottleneck, the benefits arguably justify that complexity.
- **[EIP-7971](https://eips.ethereum.org/EIPS/eip-7971): Hard Limits for Transient Storage.**
  - This proposal decreases the costs of `TLOAD` and `TSTORE` and adds a limit to available storage slots, thus improving the usability of transient storage.
  - Pushing more users to leverage transient storage will ease the burden on state access.
  - It also requires the addition of a new transaction-level counter to track the number of transient storage slots used, which adds some complexity but is manageable.
- **[EIP-8053](https://eips.ethereum.org/EIPS/eip-8053): Milli-gas for High-precision Gas Metering** OR **[EIP-8059](https://eips.ethereum.org/EIPS/eip-8059): Gas Units Rebase for High-precision Metering.**
  - Both proposals remove rounding errors from efficient compute operations, which occupy on average 4% of block space per transaction.
  - The choice between the two EIPs depends on the choice between impacting UX versus adding complexity to the EVM.
  - Milli-gas adds the complexity of a new gas counter plus transaction-level rounding. In contrast, a gas rebase changes the UX by making all gas parameters 1000x larger and changing how gas is displayed to users.
- **[EIP-8058](https://eips.ethereum.org/EIPS/eip-8058): Contract Bytecode Deduplication Discount.**
  - This proposal reduces the cost of deploying duplicated contracts, thus lowering the cost of popular contract deployments such as Uniswap pools.
  - The proposal leverages BALs to identify duplicated bytecode, thus reducing the implementation complexity.

### Possible Additions

This final set of EIPs includes repricings that introduce new mechanisms or complexity that warrant further consideration. While they tackle mispriced operations or components, in our view, their added complexity should be considered only after the previous EIPs have been included.

As with the recommended additions, these EIPs can be considered independently. These EIPs are:

- **[EIP-7973](https://eips.ethereum.org/EIPS/eip-7973): Warm Account Write Metering.**
  - This proposal introduces a discount to account writes to warm accounts (i.e., accounts that were previously updated in the same transaction), thus making multiple writes to the same account cheaper.
  - While this repricing does tackle a mispriced operation, it introduces the complexity of tracking whether the account was previously changed in any account write operation.
- **[EIP-7923](https://eips.ethereum.org/EIPS/eip-7923): Linear, Page-Based Memory Costing **OR** [EIP-7686](https://eips.ethereum.org/EIPS/eip-7686): Linear EVM memory limits.**
  - These two proposals change the memory expansion cost from quadratic to linear, thus making memory-heavy transactions cheaper.
  - EIP-7686 only updates the gas cost and leaves the memory management implementation up to the choice of clients.
  - EIP-7923 additionally describes a page-based memory model to improve memory management.
- **[EIP-8011](https://eips.ethereum.org/EIPS/eip-8011): Multidimensional Gas Metering.**
  - This proposal introduces a multidimensional gas accounting by EVM resource. Instead of tracking a gas total, the EVM would track a separate vector of gas usage by resource (e.g., compute, state access, state growth, data, etc.). Transaction fees remain the same, but gas limits are enforced on the bottleneck resource (i.e., the one spending the most gas).
  - This new accounting enables more throughput and better control of excessive resource usage, with minimal changes to the UX.
  - However, it does introduce complexity to the EVM by changing how gas is tracked and limits are enforced.
- **[EIP-8057](https://eips.ethereum.org/EIPS/eip-8057): Inter-Block Temporal Locality Gas Discounts.**
  - This proposal introduces gas discounts for state access operations, based on whether the same state was accessed in previous blocks. It leverages BALs to track state access patterns across blocks.
  - This pricing aligns with the current caching logic on clients and makes state access cheaper for frequently accessed state.
  - However, more analysis is needed to understand whether these discounts are accurate across the various clients and whether no new bottlenecks are introduced.
- **[EIP-2926](https://eips.ethereum.org/EIPS/eip-2926): Chunk-Based Code Merkleization.**
  - This proposal introduces a fixed-sized chunk approach to code merkleization that reduces the block witness sizes in stateless or partial-stateless Ethereum implementations. It also updates the code deposit cost to reflect the new chunk-based approach.
