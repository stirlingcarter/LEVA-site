## Latency-Optimized Flashcard Scheduling for Symbolic Musical Practice

### Introduction

This paper outlines a data-driven approach for optimizing latency reduction in symbolic musical practice, particularly where learners must respond to musical symbols with physical execution (e.g., playing a chord on an instrument). Our goal is to minimize both the expected latency and variance of execution times across a set of symbols, analogous to optimizing retention in a flashcard system.

### Latency Dataset Structure

We collect a dataset of latency measurements $\ell_x$, where each $\ell_x^{(i)}$ denotes the time (in milliseconds) between presentation (e.g., seeing "EbMaj7") and response (e.g., playing the chord).

Example:

```json
{
  "Eb$Maj7$RightHand" : [592, 892, 752, ...],
  "Ab$MajTriad$Inversion2" : [2321, 2211, 1892, ...],
  ...
}
```

Each symbol is composite, consisting of atomic elements (e.g., root note, quality, inversion, hand). These can be disentangled:

```json
{
  "Eb" : [592, 892, 752, ...],
  "Maj7" : [592, 892, 752, ...],
  ...
}
```

This structure enables hierarchical tracking of fluency at both component and composite levels.

### Objective

Let $S$ be the set of all practiced symbols, and let $\mathbb{E}[\ell_x]$ denote the expected latency for symbol $x \in S$. Our goal is to reduce and normalize $\mathbb{E}[\ell_x]$ for all $x$, improving fluency efficiently across the entire constellation of symbols.

### Baseline: Max-Latency Selection

A naïve algorithm selects the symbol with the highest recent average latency over a sliding window $w$:

$x^* = \arg\max_x \frac{1}{w} \sum_{i=1}^{w} \ell_x^{(i)}$

This method ignores decay, volatility, spacing effects, and latent structure among symbols.

### Proposed Model: Composite Priority-Based Scheduling

Inspired by SM2, Ebisu, and reinforcement learning, we define a composite score $P(x)$:

```math
$P(x) = w_1 R(x) + w_2 L(x) + w_3 V(x) + w_4 C(x) + w_5 LS(x) + w_6 CO5(x) + w_7 DIA(x)$
```
Each $w_i$ is a tunable scalar. We describe the exact implementation for each function below.

---

### Recency Penalty: $R(x)$

#### Formula

``` math
R(x) = \begin{cases}
1 & \text{if never practiced} \\
\frac{\max(T) - T_x}{\max(T) - \min(T)} \cdot c & \text{otherwise}
\end{cases}
```

Where $T_x$ is the timestamp when $x$ was last practiced, $\max(T)$ and $\min(T)$ are the most recent and least recent timestamps in the current set of eligible items, and $c$ is a scaling factor (typically 0.99 if never-practiced items exist, 1 otherwise).

#### Implementation

```js
// Collect all timestamps and find min/max
// (Code simplified for clarity)
// Most recent item (maxTimestamp) gets penalty=0
// Least recent (minTimestamp) gets penalty approaching 1
const R_penalty = (maxTimestamp - thisTime) / (maxTimestamp - minTimestamp) * maxPenalty;
```

* If never practiced: $R = 1$ (maximum penalty)
* If most recently practiced: $R = 0$ (minimum penalty)
* Otherwise: Linear scaling between 0 and 1 based on relative recency

#### Domain

```math
T_x \in \mathbb{R}^+
```
 (Unix timestamps)

#### Range

```math
R(x) \in [0, 1]$
```

#### Normalization

Already in $[0,1]$; no transformation required.

---

### Latency Gap: $L(x)$

#### Formula

```math
L(x) = \min\left(1, \frac{\text{mean latency}_x}{\text{LATENCY\_NORMALIZATION\_CAP}}\right)
```
Where $\text{mean latency}_x$ is the average latency over the lookback window for item $x$, and `LATENCY_NORMALIZATION_CAP` is a fixed upper bound (e.g., 6000ms). This score represents the item's recent absolute performance, normalized to the range $[0, 1]$. Unlike a gap-based score, this ensures that even latencies below a specific target contribute to the priority, encouraging continuous improvement towards instantaneous (0ms) execution, which corresponds to $L(x)=0$.

#### Implementation

```js
// Example assuming LATENCY_NORMALIZATION_CAP is defined
const meanLatency = Util.arrayAvg(recentTimes);
const L_norm = Math.min(1, meanLatency / LATENCY_NORMALIZATION_CAP);
return L_norm;
```

#### Domain

```math
\ell_x^{(i)} \in \mathbb{R}^+$, $\text{LATENCY\_NORMALIZATION\_CAP} \in \mathbb{R}^+
```

#### Range

```math
L(x) \in [0, 1]
```

#### Normalization

Normalization is performed by the $L(x)$ function using the `LATENCY_NORMALIZATION_CAP`, ensuring the output is always in the $[0, 1]$ range.

---

### Volatility: $V(x)$

#### Formula

```math
V(x) = \min\left(1, \frac{5 \times \sqrt{\text{std deviation of recent latencies}_x}}{\text{VOLATILITY\_CAP}}\right)
```

This score represents the recent performance consistency (standard deviation), transformed with a square root and scaled, then normalized to the range $[0, 1]$ using a fixed `VOLATILITY_CAP` (e.g., 100).

#### Implementation

```js
const stdDev = Util.standardDeviation(recentTimes);
const V_raw = 5 * Math.sqrt(stdDev);
const V_norm = Math.min(1, V_raw / VOLATILITY_CAP); // Assuming VOLATILITY_CAP is defined
return V_norm; // The function returns the normalized value
```

#### Domain

```math
\ell_x^{(i)} \in \mathbb{R}^+
```
minimum 2 entries.

#### Range

```math
V(x) \in [0, 1]
```

#### Normalization

Normalization is performed *internally* by the $V(x)$ function using the `VOLATILITY_CAP` constant, ensuring the output is always in the $[0, 1]$ range.

---

### Cluster Cohesion: $C(x)$

#### Formula

```math
C(x) = \frac{1}{|G|} \sum_{y \in G} \text{sim}(x, y)
```

#### Implementation

Normalized Jaccard similarity over string components

#### Domain

All pairs $x, y$ with string encodings split by $$'$'$$

#### Range

```math
C(x) \in [0, 1]
```

#### Normalization

Already normalized

---

### Circle-of-Fifths Proximity: $CO5(x)$

#### Formula

```math
CO5(x) = \max\left(0, 1 - \frac{\text{dist}_{co5}(\text{key}(x), \text{key}_{recent})}{d_{max}}\right)
```

#### Implementation

```js
CO5 = Math.max(0, 1 - distance / 6);
```

#### Domain

```math
\text{key}(x) \in \{C, G, D, A, ...\}
```

#### Range

```math
CO5(x) \in [0, 1]
```

#### Normalization

Already normalized

---

### Diatonic Compatibility: $DIA(x)$

#### Formula

```math
DIA(x) = \max_{K \in S} \frac{|\text{notes}(x) \cap K|}{|\text{notes}(x)|}
```

#### Implementation

**Stubbed: returns 0**

#### Domain / Range

```math
DIA(x) \in [0,1]
```

by definition, when implemented

#### Normalization

None needed once implemented

---

### Latent Similarity: $LS(x)$

#### Formula

```math
LS(x) = \frac{1}{|F|} \sum_{y \in F} \text{sim}(z_x, z_y)
```

Where $z_x$ is a latent embedding vector.

#### Implementation

**Stubbed: returns 0**

#### Domain / Range

```math
LS(x) \in [0,1]
```
 by design

#### Normalization

None needed once implemented

---

### Data Scarcity: $DS(x)$

#### Formula

```math
DS(x) = \begin{cases}1 & \text{if count}(x) < T \\ 0 & \text{otherwise}\end{cases}
```

#### Implementation

```js
DS = (timesCount < threshold) ? 1 : 0;
```

#### Domain

```math
\text{count}(x) \in \mathbb{N}
```

#### Range

```math
DS(x) \in \{0, 1\}
```

#### Normalization

Already normalized

---

### Symbol Selection

At each step, the system computes $P(x)$ and samples via softmax:

```math
\text{Pr}(x) = \frac{\exp(\beta P(x))}{\sum_y \exp(\beta P(y))}
```

To prevent repeats, the last-selected item's priority is set to $-\infty$.

---

### Component-Aware Aggregation

A composite symbol $x = x_1 + x_2 + ... + x_n$ has estimated latency:

```math
\text{estimated latency}_x = \sum_i \alpha_i \cdot \text{mean latency}_{x_i}
```

This logic is implied but not implemented in current scheduler.

---

### Normalization Strategy: Purpose and Rationale

All scoring functions must fall within a comparable numerical range (ideally $[0,1]$) to:

* Ensure weight parameters $w_i$ have consistent interpretability
* Prevent any single function from dominating due to numeric scale
* Enable future transformations (e.g., adaptive weighting, learned models)

We normalize using fixed, domain-informed min-max bounds, not dataset-dependent stats, to ensure stability and maintain online applicability.

---

### Conclusion and Future Work

This framework combines theory-driven and empirical insights to adaptively schedule musical practice items. Future enhancements include full implementations of $LS(x)$, $DIA(x)$, and the application of the above normalization transformations before score aggregation.

### Latency Score: $L(x)$ (Revised)

#### Formula

```math
L(x) = \min\left(1, \frac{\text{mean latency}_x}{\text{LATENCY\_NORMALIZATION\_CAP}}\right)
```

Where $\text{mean latency}_x$ is the average latency over the lookback window for item $x$, and `LATENCY_NORMALIZATION_CAP` is a fixed upper bound (e.g., 6000ms). This score represents the item's recent absolute performance, normalized to the range $[0, 1]$. Unlike a gap-based score, this ensures that even latencies below a specific target contribute to the priority, encouraging continuous improvement towards instantaneous (0ms) execution, which corresponds to $L(x)=0$.

#### Implementation

```js
// Example assuming LATENCY_NORMALIZATION_CAP is defined
const meanLatency = Util.arrayAvg(recentTimes);
const L_norm = Math.min(1, meanLatency / LATENCY_NORMALIZATION_CAP);
return L_norm;
```

#### Domain

```math
\ell_x^{(i)} \in \mathbb{R}^+$, $\text{LATENCY\_NORMALIZATION\_CAP} \in \mathbb{R}^+
```

#### Range

```math
L(x) \in [0, 1]
```

#### Normalization

Normalization is performed by the $L(x)$ function using the `LATENCY_NORMALIZATION_CAP`, ensuring the output is always in the $[0, 1]$ range.

---

The whitepaper reflects current implementation, including deviations and stubbed logic, to support iterative development.
