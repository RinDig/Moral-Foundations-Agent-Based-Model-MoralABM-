# Moral Foundations Agent-Based Model (MoralABM)

## Overview

### This is a Modular restrcuture of Nicolas Navarre's ABM: https://github.com/navarrenicolas/MoralABM

This agent-based model (ABM) simulates the dynamics of moral beliefs in a population of agents with different moral foundations. The model is based on Moral Foundations Theory, which posits that people's moral intuitions are built upon several innate foundations: care, fairness, loyalty (ingroup), authority, and purity. The model explores how agents with different moral profiles interact, signal their values, and update their beliefs about each other over time.
![image](https://github.com/user-attachments/assets/079f3c37-982b-4577-86f8-fee7324b3a01)

## Mathematical Framework 

[//]: # We should include a section on the graph representation of the model too, the paper will be helpful.

### 1. Moral Representation

Each agent's moral values are represented using either:

- **Dirichlet Distribution**: A multinomial probability distribution with concentration parameters α₁, α₂, ..., α₅ for each moral foundation.
- **Beta Distribution**: Each moral foundation has its own beta distribution with shape parameters α and β.

[//]: # (It might be worth noting that for both the beta and dirichlet initial parameters of agents' own values are estimated with a large empirical dataset.)

The probability distributions represent both: 
 
- The agent's own moral values               
- The agent's beliefs about other agents' moral values

[//]: # (This distinction and role of the distribution of other agents' moral values vs. an agent's own could be a little clearer)
[//]: # (the paper should be helpful with how to think about and phrase this.)

### 2. Signal Generation

At each time step, agents generate signals (actions or statements) that reflect their moral values:

For an agent i at step t:
1. Sample moral values from the agent's belief distribution for themselves:
   ```
   morals = sample_moral_values(agent_i, agent_i, step)
   ```

2. Apply softmax transformation with reliability parameter β:
   ```
   P(foundation_k) = exp(β * morals_k) / sum_j(exp(β * morals_j))
   ```
   where β represents how consistently an agent acts according to their strongest moral foundation.

3. Probabilistically select one moral foundation as a signal:
   ```
   signal = random_choice(foundations, p=P)
   ```

### 3. Belief Updating

#### 3.1 Direct Bayesian Updating

[//]: # "direct" here is unnecessary, I would say.

When agent i observes a signal from agent a:

For **Dirichlet** representation:
- Add 1 to the parameter corresponding to the observed moral foundation:
  ```
  M_agents[agent_i, agent_a, signal_a, step+1] += 1
  ```

For **Beta** representation:
- Add 1 to the α parameter of the observed foundation
- Add 1/4 to the β parameters of the other foundations:
  ```
  M_agents[agent_i, agent_a, 2*signal_a, step+1] += 1  # Alpha parameter
  M_agents[agent_i, agent_a, other_beta_parameters, step+1] += 1/4  # Beta parameters
  ```

This implements a Bayesian update where the prior is the current belief distribution and the likelihood is determined by the observed signal.

#### 3.2. Social Influence (Belief Homophily)

Agents are influenced by others' signals based on belief similarity:

1. Calculate KL divergence between agent i's belief about agent a and agent i's own beliefs:
   ```
   KL(P||Q) = sum(P(x) * log(P(x)/Q(x)))
   ```
   where P is agent i's belief about agent a's values and Q is agent i's own values.

2. Calculate influence weight: 
   ```
   weight = exp(-KL_divergence) / (n_agents - 1)
   ```
   The exponential transformation ensures that similar agents (lower KL divergence) have greater influence.

3. Update agent i's own beliefs proportionally to this weight:
   ```
   M_agents[agent_i, agent_i, signal_dimension, step+1] += weight
   ```

This creates a feedback loop where agents with similar moral profiles influence each other more, potentially leading to polarization.

### 4. KL Divergence

The Kullback-Leibler divergence measures how one probability distribution differs from another:

For **Dirichlet distributions** P and Q:
```
KL(P||Q) = log(Γ(sum(αᵖ))/Γ(sum(αᵠ))) + sum(log(Γ(αᵠ)/Γ(αᵖ))) + sum((αᵖ-αᵠ)*(ψ(αᵖ)-ψ(sum(αᵖ))))
```
where Γ is the gamma function and ψ is the digamma function.

For **Beta distributions**, the KL divergence is calculated for each moral foundation pair and summed.

## Code Architecture

### Core Classes

1. **MoralABM**: The main simulation class that:
   - Initializes agents with moral beliefs
   - Runs the simulation
   - Tracks belief changes and network formation

2. **InteractionStrategy**: Defines how agents update beliefs based on signals
   - `InteractionStrategy`: Default positive influence
   - `BackfireStrategy`: Agents may strengthen opposing beliefs when disagreement is high

3. **ConnectionStrategy**: Defines which agents remain connected (can influence each other)
   - `ConnectionStrategy`: All agents remain connected
   - `EchoChamberStrategy`: Agents disconnect from those with sufficiently different beliefs

### Main Data Structures

1. **M_agents**: 4D array (n_agents × n_agents × n_params × n_steps) storing:
   - All agents' beliefs about all other agents' moral foundations
   - All agents' reliability parameters (β)
   - For each time step in the simulation

2. **signals**: 2D array (n_agents × n_steps) storing:
   - Signals generated by each agent at each time step

3. **connections**: 3D array (n_agents × n_agents × n_steps) storing:
   - Which agents are connected at each time step (1=connected, 0=disconnected)

4. **belief_graph** and **mf_graph**: 3D arrays (n_steps × n_agents × n_agents) storing:
   - KL divergence between agents' beliefs (for network analysis)

## Simulation Process

1. **Initialization**:
   - Agents are initialized with moral foundation parameters (conservative or liberal)
   - Initially, agents have accurate beliefs about themselves and apply those to others
   
[//]: # (I am pretty sure there is two initialization strategies; apply your own beliefs to others or initiate their distributions randomly)

2. **Simulation Loop**:
   - For each time step:
     1. Each agent generates a moral signal
     2. Agents update their beliefs about other agents (Bayesian update)
     3. Agents update connection statuses (based on ConnectionStrategy)
     4. Agents update their own beliefs (social influence)
     5. KL divergences are calculated and stored

3. **Visualization**:
   - Polarization is measured as the average KL divergence between groups
   - Results show how moral beliefs evolve and potentially polarize over time

## Interaction Strategies

[//]: # (These are good ideas! But we need to stay focused on the taking the easiest steps first)

[//]: # (that is, re-indexing KL-similarity as such that there is negative and positive influence.)

[//]: # (I don't think the backfire strategy should necessarily be implemented currently as I think (a) it will lead to too strong polarization)

[//]: # (and (b) I don't think it is realistic for your average person to engage in this a lot of the time.)

[//]: # (there is, however, evidence for that when people engage in moral decision making (specifically trade offs) they revise their internal states)

[//]: # (to maintain coherence. Either way I think these effects are too complex for the aim of the model.)

### Default Strategy
Agents positively influence each other based on belief similarity:
```
update_weight = exp(-KL_divergence) / (n_agents - 1)
```

### Backfire Strategy
When disagreement exceeds a threshold, agents may strengthen opposing beliefs:
```
if KL_divergence > threshold:
    update_weight = -backfire_strength * exp(-KL_divergence) / (n_agents - 1)
else:
    update_weight = exp(-KL_divergence) / (n_agents - 1)
```
This captures the psychological phenomenon where people sometimes strengthen existing beliefs when confronted with opposing viewpoints.

## Connection Strategies

### Fully Connected
All agents remain connected throughout the simulation, allowing universal influence.

### Echo Chamber
Agents disconnect from those with sufficiently different beliefs:
```
if KL_divergence > disconnect_threshold:
    connections[agent_i, agent_a] = 0
```
Agents may reconnect if beliefs become more similar:
```
if KL_divergence < reconnect_threshold and current_connection == 0:
    connections[agent_i, agent_a] = 1
```
This models selective exposure and information filtering in social networks.

## Running Simulations

The `run_simulation()` function provides a simple interface to configure and run simulations:
- Choose interaction strategies (Default or Backfire)
- Choose connection strategies (FullyConnected or EchoChamber)
- Run the simulation with selected parameters
- Visualize polarization over time

## Interpreting Results

### Polarization Metric
The average KL divergence between conservative and liberal agents measures polarization:
- Higher values indicate more extreme differences in beliefs
- Increasing trends suggest growing polarization
- Decreasing trends suggest convergence of beliefs

### Key Phenomena
The model can reproduce several social phenomena:
1. **Group polarization**: Groups becoming more extreme through interaction
2. **Echo chambers**: Selective exposure leading to reinforced beliefs
3. **Backfire effect**: Opposing information strengthening existing beliefs
4. **Belief homophily**: Similar agents influencing each other more strongly

## Technical Notes

1. **Numerical Stability**:
   - The code includes safeguards for numerical stability in probability calculations
   - KL divergence computation handles edge cases to prevent NaN or infinite values

2. **Model Parameters**:
   - n_agents: Number of agents (default: 20)
   - n_steps: Number of simulation steps (default: 100)
   - beta_prior: Whether to use Beta distributions (true) or Dirichlet (false)
   - normalize: Whether to normalize parameters to maintain valid distributions

## Extensions and Variants

The modular design allows for easy extension:

1. **New Interaction Strategies**:
   - Create subclasses of InteractionStrategy with custom update rules
   - Example: conformity effects, bounded confidence models

2. **Alternative Connection Dynamics**:
   - Create subclasses of ConnectionStrategy with custom connection rules
   - Example: random networks, scale-free networks, strategic network formation

3. **Parameter Studies**:
   - Vary thresholds, strengths, or other parameters to explore sensitivity
   - Test different initial distributions of moral foundations
