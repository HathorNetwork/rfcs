- Feature Name: reduce_daa_target
- Start Date: 2026-01-06
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

This RFC proposes reducing the Difficulty Adjustment Algorithm (DAA) target average time between blocks from
the current 30 seconds to 7.5 seconds, with 15 seconds considered as an intermediate alternative. This change
would significantly accelerate transaction confirmation times, particularly benefiting nano contract execution
which depends on block confirmations.

# Motivation
[motivation]: #motivation

The current DAA target of 30 seconds between blocks creates a bottleneck for nano contract execution. Nano
contracts require block confirmations to proceed, and the 30-second average block time introduces noticeable
delays in transaction finality. By reducing the target to 7.5 seconds (or 15 seconds as an alternative), we can:

1. **Accelerate nano contract execution**: Reduce the average waiting time for block confirmations by 75%
   (or 50% for the 15s option)
2. **Improve user experience**: Provide faster transaction confirmations across the network
3. **Enhance competitiveness**: Align Hathor's confirmation times more closely with other modern blockchain
   networks

The primary tradeoff is between transaction confirmation speed and the frequency of orphan blocks. This RFC
provides a comprehensive analysis of this tradeoff to inform the decision.

**Note on Long-term Strategy**: This DAA target reduction is a short-term measure to improve the user experience
for nano contract execution. In parallel, we are developing a long-term solution based on a decentralized
sequencer architecture, which is expected to provide transaction finality in a few seconds, independent of
block confirmation times. The DAA reduction will provide immediate benefits while the sequencer solution is
being developed and deployed.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What is the DAA Target?

The DAA (Difficulty Adjustment Algorithm) target is a parameter that determines how frequently blocks should be
mined on the Hathor network. Currently set to 30 seconds, it means the network adjusts mining difficulty to
maintain an average of one block every 30 seconds.

## What Changes?

This proposal would reduce the target to 7.5 seconds, meaning blocks would be found 4 times more frequently. An
intermediate option of 15 seconds (2 times more frequent) is also analyzed.

## Impact on Users

**For regular users:**
- Transactions will be confirmed approximately 4 times faster (or 2 times for the 15s option)
- Nano contract operations will execute more quickly
- The overall user experience will feel more responsive

**For node operators:**
- Increased storage requirements due to more frequent blocks
- Potential for more stale and orphan blocks (blocks that are mined but not included in the main chain)
- Must upgrade their nodes when the feature is activated

**For miners:**
- More frequent block rewards, but proportionally smaller rewards per block
- Slightly higher orphan rate due to network propagation delays

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Block Time Distribution

Block discovery follows a Poisson process, where the time between blocks follows an exponential distribution
with mean $\mu$ (the DAA target). The cumulative distribution function is:

$$F(x) = P(X \leq x) = 1 - e^{-x/\mu}$$

This means that while the average time is $\mu$, there is natural variance, and some blocks will take much
longer to find.

## Analysis of Long Block Times

For network reliability, it's important to understand how often unusually long gaps between blocks occur. The
expected number of blocks per year that take longer than a threshold $T$ is:

$$E[\\#(X > T) \mid \mu] = \frac{L}{\mu} \cdot e^{-T/\mu}$$

where $L = 365 \times 24 \times 3600 = 31{,}536{,}000$ seconds per year.

### Comparison Table: Expected Number of Slow Blocks Per Year

| Threshold | $\mu$ = 30s | $\mu$ = 15s | $\mu$ = 7.5s | $\mu$ = 3.75s |
|-----------|-------------|-------------|--------------|---------------|
| > 90s     | 52,348      | 5,214       | 26           | 0.0013        |
| > 120s    | 19,237      | 704         | 0.47         | 0.000000058   |
| > 180s    | 2,607       | 13          | 0.00016      | ~0            |
| > 240s    | 352         | 0.24        | ~0           | ~0            |
| > 300s    | 48          | 0.004       | ~0           | ~0            |

**Key insights:**
- With $\mu = 30s$, we expect about **52,000 blocks per year** to take more than 90 seconds
- With $\mu = 7.5s$, this drops to only **26 blocks per year** exceeding 90 seconds
- Blocks exceeding 2 minutes become extremely rare with $\mu = 7.5s$ (less than 1 per year)

This demonstrates a significant improvement in worst-case confirmation times, which is crucial for applications
requiring predictable execution times.

## Stale and Orphan Block Analysis

### Understanding Stale and Orphan Blocks

When a block is found, it takes time (denoted as $dt$) to be processed and propagate through the network. This
$dt$ represents the total time from when a block is found until it reaches all nodes, and consists of:

1. **Processing time at the finder node**:
   - Checking the submitted solution to the mining job is correct
   - Verifying the block validity
   - Running the consensus algorithm
   - Relaying the block to all peers

2. **Propagation and processing time at each receiving node**: Each node in the network must independently:
   - Verify the block validity
   - Run the consensus algorithm
   - Relay the block to its peers

During this $dt$ period, other miners continue working on the previous block because they haven't received the
new block yet. If another block is found during this $dt$ seconds, it will create a fork, resulting in stale
and orphan blocks.

The probability that a block is found during the propagation time $dt$ is:

$$P(\text{orphan} \mid \mu) = 1 - e^{-dt/\mu}$$

For small $dt/\mu$, this approximates to $P(\text{orphan} \mid \mu) = dt/\mu$.

### Expected Orphan Blocks Per Day

The number of blocks per day is approximately $86{,}400/\mu$. Therefore, the expected number of orphan blocks
per day is:

$$E[\text{orphans/day} \mid \mu] = \frac{86{,}400}{\mu} \times \left(1 - e^{-dt/\mu}\right)$$

### Orphan Block Comparison

Let's compare the orphan rates for different propagation times and DAA targets:

**For dt = 1 second:**

| DAA Target | Blocks/Day | Orphan Probability | Expected Orphans/Day |
|------------|------------|--------------------|----------------------|
| 30s        | 2,880      | 0.0328 (3.28%)     | 94.5                 |
| 15s        | 5,760      | 0.0645 (6.45%)     | 371.5                |
| 7.5s       | 11,520     | 0.1246 (12.46%)    | 1,435.4              |
| 3.75s      | 23,040     | 0.2358 (23.58%)    | 5,432.8              |

**For dt = 2 seconds:**

| DAA Target | Blocks/Day | Orphan Probability | Expected Orphans/Day |
|------------|------------|--------------------|----------------------|
| 30s        | 2,880      | 0.0645 (6.45%)     | 185.8                |
| 15s        | 5,760      | 0.1246 (12.46%)    | 717.7                |
| 7.5s       | 11,520     | 0.2358 (23.58%)    | 2,716.4              |
| 3.75s      | 23,040     | 0.4142 (41.42%)    | 9,542.4              |

### Ratio of Orphan Blocks

The ratio of expected orphan blocks when changing from $\mu_1$ to $\mu_2$ is:

$$\text{Ratio} = \frac{\mu_1}{\mu_2} \times \frac{1 - e^{-dt/\mu_2}}{1 - e^{-dt/\mu_1}}$$

**Important property**: When $dt/\mu$ is small (which is typically the case for well-connected networks), we can
use the approximation $1 - e^{-x} \approx x$, yielding:

$$\text{Ratio} \approx \frac{\mu_1}{\mu_2} \times \frac{dt/\mu_2}{dt/\mu_1} = \left(\frac{\mu_1}{\mu_2}\right)^2$$

This shows that **the ratio is independent of the propagation time $dt$** (as long as $dt/\mu$ remains small).
The orphan block ratio depends only on the ratio of the DAA targets.

**Practical implications:**
- Reducing from 30s to 15s: Orphan blocks increase by a factor of ~4
- Reducing from 30s to 7.5s: Orphan blocks increase by a factor of ~16
- Reducing from 30s to 3.75s: Orphan blocks increase by a factor of ~64

The actual values depend on network propagation time, but the relative increase is determined by the square of
the target ratio.

### Mining Distribution Assumptions

**Important**: The orphan analysis above assumes **uniformly distributed mining power** (sparse mining). Under
this assumption, when any miner finds a block, all other miners (representing nearly 100% of hashrate) are
temporarily mining on the outdated chain during propagation time $dt$. This represents the **worst-case scenario**
for orphan rates.

### Impact of Mining Concentration

In practice, mining is often concentrated among a few large players. When a dominant miner (or pool) finds a block:

1. They immediately have the new block (no propagation delay to themselves)
2. Only the remaining miners (a fraction of total hashrate) are "outdated"
3. The effective competing hashrate during propagation is reduced

Let $\alpha$ represent the **fraction of hashrate that is outdated** during block propagation. In the sparse
mining case, $\alpha \approx 1$. With a dominant player holding $(1-\alpha)$ of the hashrate who just found a
block, only $\alpha$ fraction is competing.

The orphan probability becomes:

$$P(\text{orphan} \mid \mu, \alpha) = 1 - e^{-\alpha \cdot dt / \mu}$$

### Orphan Rates by Mining Concentration (dt = 1 second)

**Expected Orphan Blocks Per Day**:

| $\alpha$ (outdated fraction) | $\mu$ = 30s | $\mu$ = 15s | $\mu$ = 7.5s | $\mu$ = 3.75s |
|------------------------------|-------------|-------------|--------------|---------------|
| **100% (sparse)**            | 94.5        | 371.5       | 1,437.7      | 5,390.4       |
| **50%**                      | 47.5        | 188.6       | 743.0        | 2,858.2       |
| **25%**                      | 23.9        | 95.0        | 377.9        | 1,474.6       |
| **10%**                      | 9.6         | 38.2        | 152.1        | 600.5         |
| **5%**                       | 4.8         | 19.1        | 76.5         | 302.0         |
| **1%**                       | 1.0         | 3.8         | 15.3         | 60.7          |

**Orphan Probability Per Block**:

| $\alpha$ (outdated fraction) | $\mu$ = 30s | $\mu$ = 15s | $\mu$ = 7.5s | $\mu$ = 3.75s |
|------------------------------|-------------|-------------|--------------|---------------|
| **100% (sparse)**            | 3.28%       | 6.45%       | 12.46%       | 23.41%        |
| **50%**                      | 1.65%       | 3.28%       | 6.45%        | 12.46%        |
| **25%**                      | 0.83%       | 1.65%       | 3.28%        | 6.41%         |
| **10%**                      | 0.33%       | 0.66%       | 1.32%        | 2.61%         |
| **5%**                       | 0.17%       | 0.33%       | 0.66%        | 1.31%         |
| **1%**                       | 0.03%       | 0.07%       | 0.13%        | 0.26%         |

### Interpretation

**Scenario Analysis**:

1. **Sparse Mining ($\alpha$ = 100%)**: The worst case. Orphan rates at 7.5s are significant (12.46%).

2. **Very High Concentration ($\alpha$ = 10%)**: Orphan rates become negligible across all targets.

**Key Insight**: The orphan analysis in this RFC represents an upper bound. Real-world orphan rates depend heavily
on mining distribution. Before finalizing the target, the current Hathor mining distribution should be analyzed
to estimate realistic orphan rates.

### Caveats

This concentrated mining analysis has important caveats:

1. **Security Trade-off**: Concentrated mining improves orphan rates but increases centralization risk
2. **Variable $\alpha$**: The effective $\alpha$ varies depending on which miner finds each block
3. **Pool Behavior**: Mining pools have internal latency; not all pool hashrate instantly switches
4. **Network Effects**: Concentrated miners may have better connectivity, further reducing effective $dt$

## Storage Growth Analysis

With more frequent blocks, blockchain storage will grow faster. Assuming an average block size of 1 KB:

**Note on Block Size**: Hathor's 1 KB average block size is relatively small compared to other blockchains.
For reference, Ethereum's average block size is approximately 50-100 KB (varying based on network activity and
gas usage). Hathor's smaller block size means that even with 4x more blocks, the storage growth remains very
manageable.

### Annual Storage Growth

| DAA Target | Blocks/Year | Storage/Year |
|------------|-------------|--------------|
| 30s        | 1,051,200   | ~1.0 GB      |
| 15s        | 2,102,400   | ~2.0 GB      |
| 7.5s       | 4,204,800   | ~4.0 GB      |
| 3.75s      | 8,409,600   | ~8.0 GB      |

Given modern storage capacities, even the 3.75s target results in manageable storage growth (~40 GB over 5 years),
which is negligible for most node operators.

## Implementation Considerations

### Feature Activation

This change must be implemented through a feature activation mechanism:

1. **Hard Fork Approach**: The DAA target change requires consensus across the network
2. **Activation Height**: A specific block height will be designated for activation
3. **Upgrade Window**: Node operators will need a reasonable window to upgrade their software
4. **Backward Compatibility**: Old nodes will reject blocks after activation height if not upgraded

**Testnet Deployment Strategy:**

The feature will first be activated on testnet before mainnet deployment. However, it's important to note that
testnet validation has significant limitations:

- **Low Hashrate**: Testnet has minimal mining activity compared to mainnet
- **Lack of Miners**: Very few active miners on testnet means network propagation patterns differ significantly
- **Different Network Topology**: Testnet's network structure doesn't accurately represent mainnet conditions

Therefore, while testnet activation is a necessary preliminary step to catch obvious bugs, it will not provide
accurate data about orphan rates, propagation times, or DAA behavior under real-world conditions. The testnet
deployment should be viewed as a technical validation rather than a comprehensive performance test.

Only after successful testnet activation and sufficient monitoring period will the feature be activated on
mainnet.

### Difficulty Adjustment

The DAA will continue to function as before, but will adjust difficulty to maintain the new target (7.5s or 15s)
instead of 30s. The adjustment algorithm itself does not need to change, only the target parameter.

**Post-Activation Behavior:**

After the feature is activated at the designated block height, the following will occur:

1. **Initial Period**: Blocks will continue to be found at approximately the old rate (30s average) because the
   mining difficulty has not yet adjusted
2. **DAA Response**: The DAA will detect that blocks are taking longer than the new target (7.5s or 15s) and
   will begin reducing the mining difficulty
3. **Convergence**: Over the adjustment period, the difficulty will gradually decrease until blocks are being
   found at the new target rate
4. **Stabilization**: Once the new equilibrium is reached, the DAA will maintain the difficulty to sustain the
   new target average block time

The time required for this convergence depends on the DAA's adjustment parameters and window size.

### Mining Reward Adjustment

To maintain Hathor's inflation rate, the mining reward per block must be adjusted. Currently, the network is
designed to mint HTR at a rate of 8 HTR every 30 seconds. When the block time is reduced, the reward per block
must be reduced proportionally:

| DAA Target | Reward Multiplier | New Reward per Block | HTR per 30 seconds |
|------------|------------------|---------------------|-------------------|
| 30s        | 1.0x             | 8 HTR               | 8 HTR             |
| 15s        | 0.5x             | 4 HTR               | 8 HTR             |
| 7.5s       | 0.25x            | 2 HTR               | 8 HTR             |
| 3.75s      | 0.125x           | 1 HTR               | 8 HTR             |

This ensures that:
- The total inflation rate remains constant
- The economic model of the network is preserved
- Miners' expected rewards over time remain proportional to their hashrate contribution

**Important**: This reward adjustment is critical for maintaining the network's economic properties and must be
implemented simultaneously with the DAA target change.

# Drawbacks
[drawbacks]: #drawbacks

1. **Increased Orphan Rate**: The most significant drawback is the increase in orphan blocks, which represents
   wasted computational work and could lead to:
   - Short-term forks becoming more common
   - Reduced mining efficiency
   - Potential security implications if orphan rate becomes too high

2. **Storage Growth**: Blockchain storage will grow 2-4 times faster, though this remains manageable with modern
   storage capacities.

3. **Network Bandwidth**: Increased block propagation frequency will require more network bandwidth.

4. **Testing Complexity**: The change needs extensive testing to ensure the DAA functions correctly at the new
   target and doesn't introduce instability.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alternative: Reduce to 15 seconds

A more conservative approach would be to reduce the target to 15s:

**Advantages:**
- Lower orphan rate increase (4x vs 16x)
- Half the storage growth compared to 7.5s option

**Disadvantages:**
- Only 2x improvement in confirmation times
- Still leaves significant waiting time for nano contracts
- May require another change in the future if 15s proves insufficient

## Impact of Not Doing This

If we don't reduce the DAA target:
- Nano contract adoption may be limited by slow execution times
- User experience will lag behind competing networks
- The network may be perceived as outdated or slow

# Prior art
[prior-art]: #prior-art

## Block Times in Other Blockchains

Many successful blockchain networks operate with block times faster than Hathor's current 30s:

- **Ethereum**: ~12 seconds (historically 13-15s)
- **Litecoin**: 2.5 minutes (150s)
- **Bitcoin**: 10 minutes (600s)
- **Cardano**: ~20 seconds
- **Solana**: ~0.4 seconds (very different architecture)
- **Polygon**: ~2 seconds

Hathor's proposed 7.5s target would place it in the faster half of established networks, while 15s would be
comparable to Ethereum.

## Lessons from Other Networks

1. **Ethereum's Experience**: Ethereum has operated successfully at ~12-15s block times for years,
   demonstrating that this range is practical for a global network with varying connectivity.

2. **Orphan Rates**: Networks with faster block times (e.g., Ethereum) do experience higher uncle/orphan rates,
   but have managed this successfully through:
   - Good network connectivity and fast propagation
   - Block size limits to ensure fast propagation
   - Uncle reward mechanisms (though not proposed here)

3. **Storage Growth**: Even Bitcoin's 10-minute block time has led to significant storage growth over time. The
   primary driver is transaction volume, not block frequency. Modern storage is inexpensive enough that 2-4x
   increase in block count is not a significant concern.

## Research Papers

The relationship between block time, security, and orphan rates has been studied extensively:

- Block propagation time is the critical factor in determining orphan rates
- Smaller, more frequent blocks can actually improve security by reducing the variance in confirmation times
- The exponential distribution of block times means that reducing the average significantly reduces worst-case
  delays

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## To Resolve Before Merging This RFC

1. **Network Propagation Measurements**: What is the actual average propagation time ($dt$) in the current Hathor
   network? This is crucial for accurate orphan rate predictions.

2. **Choice Between 7.5s and 15s**: Should we go with the more aggressive 7.5s or the conservative 15s target?

3. **Orphan Tolerance**: What orphan rate should we consider acceptable? At what point does it become a security
   concern?

4. **Activation Timeline**: How much notice do node operators need?

## To Resolve During Implementation

1. **DAA Stability**: Does the current DAA algorithm remain stable at the new target? May require parameter
   tuning.

2. **Network Monitoring**: What metrics should we monitor after activation to ensure the change is successful?

3. **Rollback Plan**: What is the contingency plan if the new target proves problematic?

## Out of Scope for This RFC

1. **Uncle Block Rewards**: Should we implement an uncle/orphan reward system like Ethereum? This could mitigate
   some orphan concerns but adds complexity.

2. **Further Reductions**: Could we eventually reduce to even faster targets (e.g., 5s or less)?

# Future possibilities
[future-possibilities]: #future-possibilities

## Orphan Management

Future enhancements could include:
- **Uncle Block Rewards**: Rewarding orphaned blocks to reduce wasted work
- **GHOST Protocol**: Using orphaned blocks in the consensus weight

## Impact on Other Features

Faster block times could enable or enhance:
- **Lightning-style Payment Channels**: Faster settlement times
- **Cross-chain Bridges**: Quicker finality for bridged assets
- **DeFi Applications**: More responsive smart contract platforms
- **Gaming and Microtransactions**: Better UX for small, frequent transactions
