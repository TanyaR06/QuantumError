# Methodology & Theoretical Framework: Q-CROSS

## 1. Physical Surface Code & Crosstalk Error Formulation

### 1.1 Rotated Surface Code Architecture
Q-CROSS investigates rotated surface code memory experiments generated using the Stim stabilizer circuit simulator (`surface_code:rotated_memory_{basis}`) with code distance $d \in \{3, 5, 7, 9\}$ and $d$ syndrome extraction rounds.

A rotated surface code of distance $d$ encodes a single logical qubit into $n = d^2$ physical data qubits, interleaved with $(d^2 - 1)$ measurement ancillas representing alternating weight-2 and weight-4 $X$- and $Z$-type stabilizers.

The base physical noise model applies depolarizing noise $p_0 = 0.003$ ($0.3\%$) across four primary fault locations:
1. `after_clifford_depolarization = p0`
2. `before_round_data_depolarization = p0`
3. `before_measure_flip_probability = p0`
4. `after_reset_flip_probability = p0`

### 1.2 Crosstalk Noise Channels
In a multi-tenant quantum processor, physical proximity between tenant boundaries gives rise to microwave electromagnetic coupling and residual $ZZ$ interactions:
1. **Independent Depolarising Crosstalk (Z-Memory)**:
   $$\mathcal{E}_{\text{ind}}(\rho) = (1 - p)\rho + \frac{p}{3}(X\rho X + Y\rho Y + Z\rho Z)$$
   Injected on single data qubits located in the two rows nearest the attacker's boundary.
2. **Correlated ZZ Crosstalk (X-Memory)**:
   $$\mathcal{E}_{ZZ}(\rho) = (1 - p)\rho + p(Z_a Z_b)\rho(Z_a Z_b)$$
   plus single-qubit $Z$ phase errors $\mathcal{E}_Z(\rho) = (1 - p)\rho + p Z\rho Z$ on adjacent pairs of data qubits at coordinate distance 2.0 ($|x_1 - x_2| + |y_1 - y_2| = 2$).

### 1.3 The Wrong-Basis Invariant
In surface codes, the logical $Z$ operator $\bar{Z}$ consists of a transversal chain of $Z$ Pauli operators along the $Z$-boundary. Because physical $Z$ errors commute with all $Z$ stabilizers and with the logical observable $\bar{Z}$, pure $Z$-type errors ($Z$ or $ZZ$) produce **zero syndrome detection events** and **zero logical flips** on a $Z$-memory experiment. Consequently, an attacker attempting to poison an $X$-basis crosstalk prior against a $Z$-memory experiment induces exactly $1.0\times$ damage (no degradation).

---

## 2. Adaptive Decoders & Decoder Prior Poisoning

### 2.1 Minimum-Weight Perfect Matching (MWPM)
The victim decodes syndromes using PyMatching. For each physical fault mechanism $j$ with prior probability $p_j$, PyMatching assigns an edge weight $w_j$ in the matching graph:
$$w_j = \ln\left(\frac{1 - p_j}{p_j}\right)$$

### 2.2 Mechanism of Prior Poisoning
Self-calibrating decoders periodically re-estimate error rates $\{p_j\}$ from syndrome frequency data during dedicated calibration windows.

When a malicious co-tenant generates artificial crosstalk errors $\{p_{\text{attack}}\}$ during calibration, the victim's self-calibration absorbs these higher probabilities:
$$p_{\text{belief}, j} \leftarrow p_{\text{attack}, j} \gg p_0 \implies w_{\text{poisoned}, j} \ll w_0$$
When the attacker ceases execution, physical hardware returns to clean noise $p_0$, but the decoder continues operating with the poisoned prior. Because boundary edge weights are artificially depressed, the decoder's Blossom V matching algorithm incorrectly pairs interior detection events to the boundary rather than to their true physical partners, inducing an observable logical error.

---

## 3. REINFORCE Policy Gradient Formulation

To identify the most damaging spatial crosstalk allocation under a fixed budget $B = 0.25$, the adversary uses single-step REINFORCE:

1. **Policy Parameterization**:
   The policy parameters $\theta \in \mathbb{R}^n$ define a Gaussian policy over softmax logits:
   $$Z \sim \mathcal{N}(\theta, \sigma^2 I), \quad \sigma = 0.6$$
   The candidate allocation vector $A \in \mathbb{R}^n$ is projected using numerically stable softmax:
   $$A = B \cdot \frac{\exp(Z - \max(Z))}{\sum_{i=1}^n \exp(Z_i - \max(Z))}$$
   ensuring $A_i \ge 0$ and $\sum_i A_i = B$.

2. **Variance-Reduced Reward Evaluation**:
   At each training iteration, $K = 6$ candidate allocations are generated. The reward $R_k$ is the logical error damage evaluated on the **exact same pre-sampled clean syndrome dataset**:
   $$R_k = \frac{\text{LER}(A_k)}{\text{LER}_{\text{clean}}}$$
   Using a fixed physical sample set isolates parameter variance from Monte Carlo sampling variance.

3. **Standardized Advantage & Gradient Ascent**:
   $$\text{adv}_k = \frac{R_k - \bar{R}}{\sigma_R + 10^{-9}}$$
   $$\theta \leftarrow \theta + \frac{\eta}{K \sigma} \sum_{k=1}^K \text{adv}_k (Z_k - \theta), \quad \eta = 0.6$$

4. **Held-Out Generalization**:
   The final policy $\pi = B \cdot \text{softmax}(\theta)$ is evaluated on a fresh, unseen Monte Carlo seed ($10^5$ shots) to ensure honest out-of-sample validation.

---

## 4. Tenant-Attributed Statistical Defence (PriorGuard)

The defender owns the cloud queue and interleaves tenant execution into two distinct calibration window types:
- **Neighbour-ACTIVE windows**: Calibration executed while the adjacent co-tenant is running.
- **Neighbour-PAUSED windows**: Calibration executed while the adjacent co-tenant is paused.

### 4.1 Statistical Hypothesis Test
Let $P_i$ denote the empirical detection event rate of near-boundary detector $i$ in ACTIVE windows, and $A_i$ denote the rate in PAUSED windows across $N$ shots.
Under the null hypothesis $H_0$ (no crosstalk poisoning from the neighbour):
$$H_0: \mathbb{E}[P_i] = \mathbb{E}[A_i]$$
The pooled detection rate and standard error are:
$$\bar{p}_i = \frac{P_i + A_i}{2}$$
$$\text{SE}_i = \sqrt{\frac{2 \bar{p}_i (1 - \bar{p}_i)}{N}} + 10^{-12}$$
The test statistic is the maximum one-sided standard score across boundary detectors:
$$Z_{\text{score}} = \max_{i \in \text{boundary}} \frac{P_i - A_i}{\text{SE}_i}$$

### 4.2 Decision Rule
- If $Z_{\text{score}} > 4.5$: The calibration update is **rejected** (prior preserved; security alarm triggered).
- If $Z_{\text{score}} \le 4.5$: The calibration update is **accepted** (valid physical drift accommodated).

### 4.3 Residual Damage
For an attacker scaling its crosstalk intensity by factor $\alpha \in [0, 1]$ to evade detection:
$$\text{Residual Damage} = 1 + (1 - \text{Flag Rate}) \times (\text{Damage} - 1)$$
As calibration shot budgets increase ($2 \times 10^4 \to 5 \times 10^5$), the guard's statistical power expands, driving residual damage down toward $1.0\times$ (complete nullification).
