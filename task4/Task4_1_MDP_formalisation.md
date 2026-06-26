# 4.1 Smart Charging: Problem Formalisation as an MDP

## 1. Problem in words

An electric taxi arrives home each day at a fixed time and leaves a fixed interval later. Within that window a charging agent decides at regular short intervals, how much power to draw, so that:

1. the car has **enough energy for the next working day**. The day's energy demand is stochastic and only revealed at departure, and running out must be avoided at almost any cost
2. the **recharging cost is as low as possible**. Cost rises *exponentially* with charging power and varies with the time of day.

This is a sequential decision problem under uncertainty with a clear trade-off (charge enough vs. charge cheaply), which is exactly what a **Markov Decision Process (MDP)** describes: at each step the agent observes a state, takes an action, the state transitions, and a reward is received. The next state depends only on the current state and action (the Markov property), which holds here because the battery level summarises all relevant history.

## 2. Decision timing and horizon

The charging window has fixed length and is divided into intervals of equal duration $\Delta t$, giving a **finite horizon of $N$ decision steps**:

$$t \in \{0, 1, \dots, N-1\}$$

At each step $t$ the agent picks a charging power, the battery state updates, and time advances to $t+1$. Step $t = N$ is the **terminal step (departure)**: no action is taken, the daily energy demand is realised, and the terminal reward (any shortfall penalty) is applied. The process is **episodic**, so one episode equals one charging window.

## 3. State space $\mathcal{S}$

The state captures everything needed to act optimally, and nothing redundant:

$$s_t = (t,\; b_t), \qquad t \in \{0,\dots,N\},\quad b_t \in [0, B_{\max}]$$

- $t$: the **current time step**. Relevant because both the price coefficient and the remaining charging opportunities depend on it.
- $b_t$: the **battery State of Charge (SoC)**, bounded by battery capacity of the vehicle $B_{\max}$.

**Why nothing else is needed.** The time-of-use price coefficient $\alpha_t$ is a deterministic function of $t$, so it carries no extra information. The daily energy demand $D$ is **not** part of the state as it is unknown until departure. This uncertainty is the essence of the problem and is precisely what forces the agent to hold a safety buffer rather than charge to an exact target.

## 4. Action space $\mathcal{A}$

The physical control is a charging power in $[0, P_{\max}]$. To permit **discrete-action** methods, the range is discretised into $K$ ordered levels (e.g. zero / low / medium / high):

$$\mathcal{A} = \{a_1, a_2, \dots, a_K\}, \qquad 0 = a_1 < a_2 < \dots < a_K = P_{\max}$$

The energy delivered in one interval by action $a$ is

$$e(a) = a \cdot \Delta t.$$

## 5. Transition dynamics $P(s_{t+1}\mid s_t, a_t)$

For non-terminal steps ($t = 0,\dots,N-1$) the transition is **deterministic**. The SoC increases by the energy charged, capped at capacity:

$$b_{t+1} = \min\big(b_t + a_t \cdot \Delta t,\; B_{\max}\big), \qquad t \leftarrow t+1.$$

The cap enforces the battery-capacity constraint $b_t \le B_{\max}$ by construction. Energy that would exceed capacity is not stored. 

**Stochasticity** enters only at the **terminal transition** ($t = N-1 \to N$). The day's energy demand is drawn from a distribution with mean $\mu_D$ and standard deviation $\sigma_D$,

$$D \sim \mathcal{N}(\mu_D, \sigma_D^2), \qquad D \leftarrow \max(D, 0),$$

generated exactly at departure and compared against the final SoC $b_N$. The truncation at zero reflects that energy demand cannot be negative.

## 6. Reward function $R$

The agent **maximises reward = − total cost**. Total cost has two components: the recharging cost accumulated over the charging intervals, and a penalty if the car departs with insufficient charge.

### 6.1 Recharging cost (per step, $t = 0,\dots,N-1$)

Charging cost is an **exponential function of the charging power**, scaled by a time-dependent coefficient:

$$c_t(a_t) = \alpha_t \left(e^{\beta a_t} - 1\right)$$

- The $(- 1)$ term ensures **zero power ⇒ zero cost**.
- $\beta > 0$ sets the **convexity**: higher power is disproportionately expensive, which discourages bursts of fast charging.
- $\alpha_t \ge 0$ is the **time-of-use price coefficient**, a deterministic function of the step $t$.

### 6.2 Shortfall penalty (terminal, $t = N$)

If the final SoC cannot cover the realised demand, a **large penalty** proportional to the shortfall is applied:

$$g(b_N, D) = \lambda \cdot \max\big(0,\; D - b_N\big), \qquad \lambda \gg 0.$$

The penalty rate $\lambda$ is chosen (in 4.3) large enough that any shortfall dominates every plausible charging cost, encoding the requirement that running out of energy must be avoided.

### 6.3 Per-step and total reward

$$r_t = -\,c_t(a_t)\ \ \text{for } t = 0,\dots,N-1, \qquad r_N = -\,g(b_N, D).$$

The return for one episode is therefore

$$R_{\text{episode}} = -\sum_{t=0}^{N-1}\alpha_t\big(e^{\beta a_t}-1\big)\;-\;\lambda\,\max\big(0,\, D - b_N\big).$$

Because the horizon is finite and episodic, an **undiscounted** objective ($\gamma = 1$) is appropriate, though a discount $\gamma \in (0,1]$ may alternatively be used (4.4).

## 7. Objective

Find a policy $\pi(a \mid s)$ — a charging rule mapping each state $(t, b_t)$ to a power level — that maximises the **expected episode return**:

$$\pi^\star = \arg\max_{\pi}\ \mathbb{E}_{D}\!\left[\,\sum_{t=0}^{N} r_t \;\Big|\; \pi\,\right],$$

where the expectation is taken over the random daily demand $D$.

## Notation

| Symbol | Meaning |
|---|---|
| $\Delta t$ | control interval length |
| $N$ | number of decision steps (horizon) |
| $t$ | time-step index |
| $b_t$ | battery State of Charge at step $t$ |
| $B_{\max}$ | battery capacity |
| $a_t$ | charging power chosen at step $t$ |
| $P_{\max}$ | maximum charging power |
| $\mathcal{A}=\{a_1,\dots,a_K\}$ | discrete action set ($K$ levels) |
| $e(a)=a\,\Delta t$ | energy charged in one interval |
| $D$ | realised daily energy demand (random) |
| $\mu_D,\ \sigma_D$ | mean and std. dev. of demand |
| $\alpha_t$ | time-of-use price coefficient |
| $\beta$ | cost-convexity coefficient |
| $\lambda$ | shortfall-penalty rate |
| $\gamma$ | discount factor |