# 8.5.6 Game Theory -- 博弈论

## 1. Simple Introduction

Game theory provides the mathematical framework for modeling strategic interactions between self-interested agents. In multi-agent systems, each agent acts as a "player" pursuing its own utility, and the outcome of the system depends on the combined choices of all agents. Game theory answers a critical question: **what should a rational agent do when the outcome depends not only on its own action but also on the actions of others?**

Key concepts from game theory that directly apply to multi-agent systems:

- **Nash Equilibrium**: A stable state where no agent can improve its payoff by unilaterally changing its strategy, assuming all other agents keep their strategies fixed. This is the most fundamental solution concept in game theory.
- **Prisoner's Dilemma**: A canonical example showing why two rational agents might fail to cooperate even when cooperation would benefit both. It reveals the tension between individual and collective rationality -- the central challenge in multi-agent coordination.
- **Coordination Games**: Situations where agents share aligned interests but must converge on a common choice. Examples include which side of the road to drive on or which communication protocol to use.
- **Zero-Sum Games**: Strictly competitive settings where one agent's gain is exactly another's loss. Useful for modeling adversarial multi-agent environments.

In a multi-agent system, game theory transforms the design question from "how do we program each agent" to "how do we design incentives so that rational agents naturally produce the desired global outcome." This shift -- from prescribing behavior to shaping incentives -- is the foundational insight of game-theoretic multi-agent design.

---

## 2. Basic Principles

### 2.1 The Three Components of a Game

Every game-theoretic model has three essential components:

```
  +--------------------------------------------------------------------+
  |                   GAME = (Players, Strategies, Payoffs)            |
  +--------------------------------------------------------------------+
  |                                                                     |
  |  1. PLAYERS      -- The decision-making entities in the system     |
  |                    (agents, bots, algorithms, human participants)   |
  |                                                                     |
  |  2. STRATEGIES   -- The set of possible actions available to each  |
  |                    player (cooperate/defect, bid price, route, etc) |
  |                                                                     |
  |  3. PAYOFFS      -- The utility/reward each player receives for    |
  |                    every combination of strategy choices            |
  |                                                                     |
  +--------------------------------------------------------------------+
```

### 2.2 The Prisoner's Dilemma Payoff Matrix

The Prisoner's Dilemma is the most studied game in all of game theory. Two agents are arrested for a crime. Each can either **Cooperate** (stay silent) or **Defect** (betray the other). Their sentences depend on the combination of choices:

```
                    Agent B
              +-----------+-----------+
              | Cooperate |  Defect   |
  +-----------+-----------+-----------+
  | Cooperate |  (3, 3)   |  (0, 5)   |
  |           | both serve| A serves  |
  |           |  1 year   |  3 years  |
  |           |           | B goes    |
  |           |           |  free     |
Agent A +-----------+-----------+-----------+
  | Defect    |  (5, 0)   |  (1, 1)   |
  |           | A goes    | both serve|
  |           |  free     |  2 years  |
  |           | B serves  |           |
  |           |  3 years  |           |
  +-----------+-----------+-----------+

  Payoffs in format: (Agent A payoff, Agent B payoff)
  Higher number = better outcome (reduced sentence)

  KEY INSIGHT:
  Defect is the DOMINANT STRATEGY for each agent individually,
  but both defecting produces the WORST collective outcome (1, 1).
  The collectively optimal result (3, 3) requires mutual cooperation.
```

### 2.3 Core Solution Concepts

**Dominant Strategy** -- A strategy that provides a higher payoff than any other strategy, regardless of what the other players do. In the Prisoner's Dilemma, Defect is a dominant strategy because:

- If B Cooperates: A gets 3 (cooperate) vs 5 (defect) -- Defect wins
- If B Defects: A gets 0 (cooperate) vs 1 (defect) -- Defect wins

**Nash Equilibrium** -- A strategy profile where each player's strategy is the best response to all other players' strategies. No player has incentive to deviate unilaterally.

```
  NASH EQUILIBRIUM VISUALIZATION

              Agent B strategy
              +-------------------+
              |    fixed at s_B   |
  +-----------+-------------------+
  |           |                   |
  | Agent A   |  Payoff for A     |
  | tries     |  given s_B        |
  | all       |                   |
  | possible  |    ***            |
  | strategies|   *   *           |
  |           |  *     *  <-- A's best response
  |           | *  MAX  *         |
  |           |  *     *          |
  |           |   *   *           |
  |           |    ***            |
  |           |                   |
  +-----------+-------------------+

  s_A* is A's best response to s_B
  s_B* is B's best response to s_A
  (s_A*, s_B*) is the Nash Equilibrium

  At equilibrium:
    payoff_A(s_A*, s_B*) >= payoff_A(s_A, s_B*)  for all s_A
    payoff_B(s_A*, s_B*) >= payoff_B(s_A*, s_B)  for all s_B
```

**Pareto Optimality** -- An outcome is Pareto optimal if no player can be made better off without making at least one other player worse off. In the Prisoner's Dilemma:

- (3, 3) is Pareto optimal (both cooperate)
- (5, 0) is Pareto optimal (A defects, B cooperates)
- (0, 5) is Pareto optimal (A cooperates, B defects)
- (1, 1) is NOT Pareto optimal (both defect -- (3, 3) makes both better)

The tragedy: the Nash equilibrium (1, 1) is the only non-Pareto-optimal outcome.

### 2.4 Game Taxonomy

```
                    GAMES
                       |
      +----------------+----------------+
      |                |                |
  COOPERATIVE      NON-COOPERATIVE     MECHANISM DESIGN
  (coalitions)     (individual)        (reverse game theory)
      |                |
  +---+---+      +-----+------+
  |       |      |            |
Shapley  Core  STATIC      DYNAMIC
Value           (one-shot)  (repeated/sequential)
                   |            |
              +----+----+    +--+--+
              |         |    |     |
          COMPLETE  INCOMPLETE  EXTENSIVE
          INFO      INFO        FORM
          (perfect) (Bayesian)  (game tree)

    STATIC COMPLETE INFO:  Prisoner's Dilemma, Battle of the Sexes
    STATIC INCOMPLETE:     Auctions (unknown valuations)
    DYNAMIC COMPLETE:      Chess, Go (perfect information)
    DYNAMIC INCOMPLETE:    Poker, negotiations (hidden cards)
```

---

## 3. Background and Development

### 3.1 Timeline

```
  1944 -- Von Neumann & Morgenstern publish "Theory of Games and
          Economic Behavior". Foundation of game theory.
          Introduced: zero-sum games, minimax theorem, coalitional
          games, utility theory.

  1950 -- John Nash proves existence of equilibrium in all finite
          games (Nash Equilibrium). Transforms game theory from
          two-player zero-sum to general n-player games.
          "Non-Cooperative Games" (1951).

  1950 -- Merrill Flood and Melvin Dresher at RAND Corporation
          invent the Prisoner's Dilemma. Albert W. Tucker formalizes
          the name and story.

  1953 -- Lloyd Shapley introduces the Shapley Value for fair
          division in cooperative games.

  1960s -- Thomas Schelling applies game theory to conflict and
          bargaining ("The Strategy of Conflict"). Focal points,
          commitment, credibility.

  1970s -- John Maynard Smith introduces Evolutionary Game Theory
          and the Evolutionary Stable Strategy (ESS). Game theory
          applied to biology.

  1980s -- Robert Axelrod's iterated Prisoner's Dilemma tournaments.
          Tit-for-Tat emerges as the most robust strategy.
          Foundation for understanding cooperation emergence.

  1990s -- Applied to computer science: algorithmic game theory,
          mechanism design for auctions (Vickrey-Clarke-Groves),
          computational social choice.

  2000s -- Multi-agent systems adopt game theory for agent
          interaction modeling. Automated mechanism design,
          computational equilibrium finding.

  2010s -- Deep learning + game theory: no-regret learning in
          games, generative adversarial networks as minimax games,
          multi-agent reinforcement learning with game theory.

  2020s -- LLM-based agents modeled through game theory lenses.
          Strategic reasoning in language models. Auction design
          for AI compute markets. Agent-to-agent negotiation.
```

### 3.2 Mechanism Design: Reverse Game Theory

Mechanism design flips the game theory question:

```
  GAME THEORY:  Given rules (game), predict agent behavior (outcome)
  MECHANISM DESIGN:  Given desired outcome, design rules (game)

  Flow:
    +----------+     +-----------+     +-----------+
    | Desired  | --> | Mechanism | --> | Agent     |
    | Outcome  |     | Designer  |     | Behavior  |
    | (social  |     | sets up   |     | (strategic|
    | welfare) |     | rules/    |     |  choices) |
    +----------+     | incentives|     +-----+-----+
                     +-----------+           |
                                            v
                                     +-----------+
                                     | Observed  |
                                     | Outcome   |
                                     +-----------+
                                          |
                                     matches desired?
                                          |
                                     +----+----+
                                     |         |
                                    YES       NO --> redesign
```

A mechanism is **incentive compatible** if agents achieve the desired outcome by truthfully revealing their private information. The **Revelation Principle** states that any outcome achievable by some mechanism can also be achieved by a direct truthful mechanism.

---

## 4. Previous Approaches and Limitations

### 4.1 Before Game Theory: Heuristic Coordination

Before game-theoretic methods entered multi-agent systems, coordination was handled through simpler approaches:

**Approach 1: Hardcoded Protocols**
```
  Agents follow predetermined rules:
    IF another agent is at intersection THEN yield
    IF resource_count < threshold THEN request more
    IF task_assigned THEN execute

  Problem: No strategic reasoning. Agents follow rules blindly
  and can be exploited by agents that deviate from expected behavior.
```

**Approach 2: Centralized Scheduling**
```
  +---------------------+
  | Central Coordinator |  -- assigns tasks, allocates resources
  +---------------------+
       |    |    |
  +----+ +--+-+ +-+----+
  | A1 | | A2 | | A3  |
  +----+ +----+ +-----+

  Problem: Single point of failure, not scalable,
  requires complete information at central node.
```

**Approach 3: Simple Negotiation Protocols**
```
  Contract Net Protocol:
    1. Manager broadcasts task announcement
    2. Agents bid on tasks
    3. Manager awards contract to best bid

  Problem: Agents may bid dishonestly if it gives them
  advantage. No mechanism to ensure truthful bidding.
```

### 4.2 System-Specific Limitations

| Approach | Limitation |
|----------|------------|
| Hardcoded Protocols | No adaptability; agents can't respond to strategic exploitation; brittle in dynamic environments |
| Centralized Scheduling | Single point of failure; requires perfect global knowledge; doesn't scale to large systems |
| Simple Negotiation | Assumes truthful behavior; naive bidding leads to inefficient allocations; no strategic modeling |
| Voting/Consensus | Naive voting (simple majority) ignores strategic voting -- agents may vote insincerely to manipulate outcomes |
| Rule-based Resource Sharing | Fixed allocation rules ignore heterogeneous agent valuations; resources go to agents who value them least |

### 4.3 Key Failure Modes

**Exploitation**: An agent following "always cooperate" rules in a Prisoner's Dilemma gets exploited by defectors.

**Inefficiency**: Without proper pricing mechanisms, resources are allocated to agents who value them least.

**Instability**: Simple coordination rules can cause oscillation (e.g., two agents repeatedly yielding to each other at a doorway).

**Manipulation**: Without incentive-compatible mechanisms, agents game the system (e.g., strategically under-reporting capacity to get easier tasks).

Game theory addresses all four failures by explicitly modeling agent incentives and designing systems robust to strategic behavior.

---

## 5. Core Contradiction

### 5.1 Individual Rationality vs. Collective Rationality

The central tension in game-theoretic multi-agent systems:

```
  INDIVIDUAL RATIONALITY
  "Each agent maximizes its own utility"
  
  vs.
  
  COLLECTIVE RATIONALITY
  "The system as a whole achieves optimal global outcome"
```

These two objectives are often misaligned. The system designer wants agents to cooperate for the common good, but each agent individually may benefit from defecting. Resolving this tension is the fundamental challenge.

### 5.2 The Tragedy of the Commons

The most vivid illustration of this contradiction:

```
  THE TRAGEDY OF THE COMMONS
  (Shared Resource Depletion Game)

  Scenario: N agents share a common resource pool.
  Each agent decides how much to consume.

  +----------------------------------------------------+
  |                                                    |
  |  Resource Pool (e.g., bandwidth, compute, data)    |
  |  +----------------------------------------------+ |
  |  |                                              | |
  |  |  Each agent consumes X_i units               | |
  |  |                                              | |
  |  |  Total consumption = sum(X_i)                | |
  |  |  Resource collapses if sum(X_i) > Capacity   | |
  |  |                                              | |
  |  +----------------------------------------------+ |
  |                                                    |
  |  Individual incentive: consume AS MUCH as possible |
  |  Collective incentive: consume SUSTAINABLE amount  |
  |                                                    |
  |  Result: each agent over-consumes -> resource      |
  |          collapses -> EVERYONE loses                |
  |                                                    |
  +----------------------------------------------------+

  Agent's payoff function:
    u_i = Benefit(X_i) - Share_of_Cost(Total_Consumption)

  If Benefit(X_i) is private, Cost is shared:
    Agent i ignores cost imposed on others
    -> overconsumption is individually rational
    -> collapse is the collective outcome
```

### 5.3 Formalization

```
  Let:
    a_i  = action of agent i (from strategy set S_i)
    u_i  = utility function of agent i
    a    = (a_1, ..., a_n) = joint action profile

  Individual Rationality:
    For each agent i, a_i maximizes u_i(a_i, a_{-i})
    i.e., a_i is a best response to others' actions

  Collective Rationality (Pareto Optimality):
    There is no a' such that u_i(a') >= u_i(a) for all i
    and u_j(a') > u_j(a) for some j

  The contradiction:
    Nash equilibria may be Pareto-dominated by non-equilibrium
    outcomes. The system is STUCK in a bad equilibrium.
```

### 5.4 Methods to Resolve the Contradiction

| Method | How It Works | Example |
|--------|-------------|---------|
| External Enforcement | Penalty for defection | Taxation, fines |
| Repeated Interaction | Future retaliation deters current defection | Tit-for-Tat |
| Preference Modification | Change agent utility functions | Altruism bonus |
| Contracting | Binding agreements with enforcement | Smart contracts |
| Mechanism Design | Align incentives through clever rules | VCG auctions |
| Communication | Cheap talk enables coordination | Pre-play negotiation |

---

## 6. Main Optimization Directions

### 6.1 Repeated Games and Reciprocity

In one-shot interactions, defection dominates. In repeated interactions, cooperation can emerge because defection is punished in future rounds.

**Tit-for-Tat Strategy**: Cooperate on the first move, then copy the opponent's previous move. This strategy won Robert Axelrod's tournaments and has four key properties:

```
  TIT-FOR-TAT PROPERTIES

  1. NICENESS: Never defects first
  2. RETALIATION: Punishes defection immediately
  3. FORGIVENESS: Resumes cooperation if opponent does
  4. CLARITY: Simple, transparent, easy to understand

  Performance in Axelrod's tournament:

  +----------------------------------------------------+
  | Strategy          | Avg Score | Rank | Cooperation  |
  |-------------------+-----------+------+--------------|
  | Tit-for-Tat       | 504.12    |  1   | High         |
  | Always Cooperate  | 472.17    |  4   | 100%         |
  | Always Defect     | 401.36    |  8   | 0%           |
  | Random (50/50)    | 276.34    | 15   | ~50%         |
  | Grudger           | 482.83    |  2   | High then 0% |
  | Tit-for-Two-Tats  | 478.34    |  3   | Very High    |
  +----------------------------------------------------+
```

**Folk Theorem**: In repeated games with sufficiently low discount rates, ANY feasible, individually rational payoff can be sustained as a Nash equilibrium. This means there are infinitely many equilibria -- the problem shifts from "finding an equilibrium" to "selecting among equilibria."

### 6.2 Cooperative Game Theory

For settings where agents can form coalitions and binding agreements:

**Shapley Value**: Distributes total reward fairly among cooperating agents based on their marginal contributions.

```
  Shapley Value for agent i:
                    |S|! (n - |S| - 1)!
  phi_i(v) = sum  --------------------- * (v(S U {i}) - v(S))
              S subset   n!
              N\{i}

  where:
    N = set of all agents
    S = coalition (subset of agents not including i)
    v(S) = value generated by coalition S
    v(S U {i}) - v(S) = marginal contribution of i to S

  Properties:
    - EFFICIENCY: sum(phi_i) = v(N)
    - SYMMETRY: identical contributors get equal share
    - LINEARITY: phi distributes across independent games
    - NULL PLAYER: agents adding no value get nothing
```

**Core**: The set of payoff distributions that no coalition can improve upon. An allocation is in the core if no group of agents would do better by leaving and forming their own coalition.

### 6.3 Evolutionary Game Theory

Models how strategies evolve in a population over time through replication, mutation, and selection:

```
  EVOLUTIONARY DYNAMICS

  +----------------------------------------------------+
  |                                                    |
  |  Population of agents                             |
  |  +---------+  +---------+  +---------+            |
  |  | Cooper- |  | Defect  |  | Tit-for |  ...       |
  |  | ate     |  |         |  | -Tat    |            |
  |  +---------+  +---------+  +---------+            |
  |       |            |            |                  |
  |       v            v            v                  |
  |  Fitness: 3    Fitness: 5    Fitness: 4            |
  |                                                    |
  |  Replicator Dynamics:                              |
  |    dx_i/dt = x_i * (f_i(x) - avg_f(x))            |
  |                                                    |
  |  where:                                            |
  |    x_i = proportion of strategy i                 |
  |    f_i = fitness (expected payoff) of strategy i  |
  |    avg_f = average fitness of population          |
  |                                                    |
  +----------------------------------------------------+
```

**Evolutionary Stable Strategy (ESS)**: A strategy that, if adopted by the entire population, cannot be invaded by any alternative strategy through natural selection. ESS is a refinement of Nash equilibrium.

### 6.4 Auction Mechanism Design

Auctions are fundamental mechanisms for resource allocation among agents:

| Auction Type | Rules | Strategic Property |
|-------------|-------|-------------------|
| English (ascending) | Bidders raise bids until one remains | May reveal private info |
| Dutch (descending) | Price decreases until someone bids | Faster, strategic timing |
| First-Price Sealed | Highest bid wins, pays bid | Bids < true value (shading) |
| **Vickrey (Second-Price)** | Highest bid wins, pays SECOND highest | **Truthful bidding is dominant** |
| All-Pay | Highest bid wins, ALL pay | Common in contests, lobbying |
| Combinatorial | Bids on bundles of items | Exposure problem |

**Vickrey-Clarke-Groves (VCG) Mechanisms**: A general class of mechanisms where truthful reporting is a dominant strategy. Agents report their valuations, the mechanism allocates to maximize total value, and each agent pays the "externality" they impose on others.

### 6.5 Automated Mechanism Design

Using computational optimization to design mechanisms, rather than deriving them analytically:

```
  AUTOMATED MECHANISM DESIGN

  Input:
    - Set of possible agent types/valuations
    - Set of possible outcomes
    - Designer's objective function
    - (Optional) constraints (budget balance, etc.)

  Optimization:
    Search over possible mechanisms (payment + allocation rules)
    subject to:
      - Incentive compatibility (truthful reporting is optimal)
      - Individual rationality (agents voluntarily participate)
      - (Optional) Budget balance

  Output:
    - Optimal mechanism for the given environment

  +------------------+     +-------------------+     +-----------+
  | Agent Type Space | --> | Algorithmic       | --> | Optimal   |
  | (valuations,     |     | Mechanism Search  |     | Mechanism |
  |  constraints)    |     | (Linear Program,  |     | (rules,   |
  +------------------+     |  MILP, ML-based)  |     |  payments)|
                           +-------------------+     +-----------+
```

### 6.6 Bayesian Games for Incomplete Information

In most multi-agent systems, agents don't know each other's private information (valuations, costs, capabilities). Bayesian games model this uncertainty:

```
  A Bayesian game includes:
    - Types: each agent i has a private type t_i (their private info)
    - Beliefs: common prior distribution over type profiles P(t_1,...,t_n)
    - Strategies: a_i(t_i) -- action depends on own type
    - Payoffs: u_i(a_1(t_1), ..., a_n(t_n); t_i)

  Bayes-Nash Equilibrium:
    Each agent maximizes expected payoff given its type and
    the distribution over others' types:
      E[ u_i(a_i, a_{-i}(t_{-i}); t_i) | t_i ] >=
      E[ u_i(a_i', a_{-i}(t_{-i}); t_i) | t_i ]

  The expectation is taken over the types of other agents.
```

---

## 7. Implementation Challenges

### 7.1 Computational Complexity

Computing Nash equilibria is computationally hard:

```
  COMPLEXITY LANDSCAPE FOR GAME-THEORETIC COMPUTATIONS

  +------------------------------------------------------+
  | Problem                         | Complexity Class    |
  |---------------------------------+---------------------|
  | Finding Nash Equilibrium        | PPAD-complete       |
  |   (2-player general-sum)       | (no poly-time       |
  |                                 |  algorithm known)   |
  |---------------------------------+---------------------|
  | Finding Nash Equilibrium        | P (Lemke-Howson     |
  |   (2-player zero-sum)           |  = linear program)  |
  |---------------------------------+---------------------|
  | Finding Nash Equilibrium        | FIXP-complete       |
  |   (3+ players)                  | (even harder)       |
  |---------------------------------+---------------------|
  | Computing Shapley Value         | #P-complete         |
  |   (exact)                       | (exponential in     |
  |                                 |  number of agents)  |
  |---------------------------------+---------------------|
  | Computing Core                  | co-NP-complete      |
  |   (checking emptiness)          |                     |
  |---------------------------------+---------------------|
  | Optimal mechanism design        | Varies: NP-hard     |
  |                                 | to inapproximable   |
  +------------------------------------------------------+

  PPAD-complete means:
    - Nash equilibrium always exists (by Nash's theorem)
    - But finding it is believed to be exponentially hard
    - No polynomial-time algorithm is known
    - Reductions to/from other PPAD problems (fixed points)
```

**Practical Implications**:

- For large games (many agents, large strategy spaces), computing exact Nash equilibria is infeasible.
- Use approximation algorithms, regret minimization, or learning-based approaches.
- Repeated games are often more tractable because agents can learn over time.
- Zero-sum games are much more tractable than general-sum games.

### 7.2 Modeling Opponent Strategies

To play optimally, an agent must model what other agents will do. This requires:

```
  CHALLENGE: How does agent A model agent B's behavior?

  Options (in increasing order of sophistication):

  1. STATIC MODEL
     Assume B follows a known fixed strategy
     Problem: B may adapt, exploit A's rigidity

  2. TYPE-BASED MODEL
     Assume B has a fixed but unknown type
     Infer type from observed behavior (Bayesian updating)
     Problem: Computationally expensive with many types

  3. RECURSIVE MODELING
     "I think that B thinks that I think..."
     Infinite regress problem: level-k reasoning
     Level-0: random/naive behavior
     Level-1: best response to level-0
     Level-2: best response to level-1
     ...

  4. LEARNING MODEL
     Observe B's past actions, learn pattern
     Use no-regret learning, neural networks, LLM prompting
     Problem: Non-stationary (B is also learning)

  5. THEORY OF MIND
     Model B's beliefs, goals, and reasoning process
     Most human-like, most computationally expensive
```

### 7.3 Dealing with Irrational Agents

Not all agents are rational (in the game-theoretic sense):

| Type of "Irrationality" | Description | Mitigation |
|-------------------------|-------------|------------|
| Bounded Rationality | Limited computation, can't find optimal | Use satisficing, heuristic strategies |
| Noisy Behavior | Random errors in action selection | Robust strategies, averaging |
| Systematic Bias | Cognitive biases (overconfidence, etc.) | Explicit bias modeling |
| Altruism | Agent values others' payoffs | Adjust utility functions |
| Spite | Agent values others' losses | Add spite parameter to model |
| Adversarial | Agent actively tries to harm | Robust mechanism design |

### 7.4 Incentive Compatibility in Practice

Designing mechanisms that are truly incentive compatible is challenging:

```
  PRACTICAL CHALLENGES IN INCENTIVE COMPATIBILITY

  1. COLLUSION
     Agents may collude to manipulate the mechanism
     VCG is not collusion-proof
     Solution: collusion-resistant mechanisms (rare)

  2. BUDGET BALANCE
     VCG is not budget-balanced in general
     Payments may not sum to zero
     Solution: accept deficit, or use suboptimal
     budget-balanced mechanisms

  3. COMPUTATIONAL INCENTIVE COMPATIBILITY
     Even if truthful reporting is theoretically optimal,
     computing the optimal truthful report may be NP-hard
     Solution: approximation mechanisms

  4. DYNAMIC PARTICIPATION
     Agents join and leave over time
     Incentives change with population
     Solution: online mechanism design

  5. MULTI-DIMENSIONAL TYPES
     Agents have multiple private attributes
     General characterization of IC impossible
     Solution: restrict to specific type spaces
```

### 7.5 The Equilibrium Selection Problem

Most games have multiple Nash equilibria. Which one will agents play?

```
  Example: Stag Hunt Game

                    Agent B
              +-----------+-----------+
              |   Stag    |   Hare    |
  +-----------+-----------+-----------+
  | Stag      |  (4, 4)   |  (0, 3)   |
  Agent A     |           |           |
  +-----------+-----------+-----------+
  | Hare      |  (3, 0)   |  (3, 3)   |
  +-----------+-----------+-----------+

  Two Nash equilibria: (Stag, Stag) and (Hare, Hare)
  (Stag, Stag) is Pareto superior
  (Hare, Hare) is risk-dominant (safe)
  Which one do agents play?

  Solution approaches:
    - Focal points (Schelling): salient equilibrium
    - Risk dominance: Harsanyi-Selten refinement
    - Payoff dominance: equilibrium with highest payoffs
    - Learning: history-dependent equilibrium selection
```

---

## 8. Capabilities and Boundaries

### 8.1 Where Game Theory Excels

```
  +-----------------------------------------------------------+
  |                    STRONG SUIT AREAS                       |
  +-----------------------------------------------------------+
  |                                                           |
  |  Resource Allocation:                                     |
  |    Auctions allocate scarce resources to highest-value    |
  |    users. Proven in spectrum auctions, ad auctions,       |
  |    cloud compute markets.                                 |
  |                                                           |
  |  Strategic Interaction:                                   |
  |    Any system where agents actively respond to each       |
  |    other's choices. Negotiation, bargaining, bidding.     |
  |                                                           |
  |  Incentive Design:                                        |
  |    Setting up rules so that selfish agents produce good   |
  |    collective outcomes. Tax systems, reward sharing,      |
  |    penalty schemes.                                       |
  |                                                           |
  |  Competitive Settings:                                    |
  |    Zero-sum or adversarial interactions. Security games,  |
  |    cybersecurity, military strategy, sports betting.      |
  |                                                           |
  |  Coalition Formation:                                     |
  |    Agents forming groups for mutual benefit. Team         |
  |    formation, supply chains, collaborative AI systems.    |
  |                                                           |
  +-----------------------------------------------------------+
```

### 8.2 Where Game Theory Struggles

```
  +-----------------------------------------------------------+
  |                    WEAK SUIT AREAS                         |
  +-----------------------------------------------------------+
  |                                                           |
  |  Open-Ended Creative Tasks:                               |
  |    Writing, art, brainstorming. Payoffs are hard to       |
  |    quantify. Strategic behavior is not the bottleneck.    |
  |                                                           |
  |  Unquantifiable Outcomes:                                 |
  |    Game theory requires numeric payoffs. When outcomes    |
  |    are qualitative, subjective, or multidimensional,      |
  |    modeling is fragile.                                   |
  |                                                           |
  |  Massive Strategy Spaces:                                 |
  |    Games with infinite or astronomically large strategy   |
  |    sets. Finding equilibria is computationally hopeless.  |
  |                                                           |
  |  Strongly Bounded Agents:                                 |
  |    If agents can't compute best responses (limited        |
  |    cognition), equilibrium predictions break down.        |
  |                                                           |
  |  Rapidly Changing Environments:                           |
  |    Game theory assumes stable payoff structures. In       |
  |    highly dynamic environments, the game changes before   |
  |    equilibrium is reached.                                |
  |                                                           |
  |  Normative vs. Descriptive Gap:                           |
  |    Game theory says what rational agents SHOULD do.       |
  |    Real agents (including LLMs) often deviate.            |
  |                                                           |
  +-----------------------------------------------------------+
```

### 8.3 Practical Boundary Summary

```
  +-----------------------------------------------------------+
  | Apply Game Theory When:                                    |
  |  - Clear, measurable agent payoffs exist                   |
  |  - Agents are approximately rational or can learn          |
  |  - Strategic interdependence is significant                |
  |  - Number of agents and strategies is manageable           |
  |  - We can design or influence the rules of interaction     |
  |                                                           |
  | Avoid Game Theory When:                                    |
  |  - Payoffs are fundamentally unmeasurable                  |
  |  - Agents follow fixed rules (no strategic choice)         |
  |  - Environment changes faster than agents can respond      |
  |  - The system has a single agent (no strategic interaction)|
  |  - Computational constraints prevent any game-theoretic    |
  |    modeling                                                |
  +-----------------------------------------------------------+
```

---

## 9. Comparison with Other Patterns

### 9.1 Game Theory vs. Market Mechanism

```
  Feature               | Game Theory           | Market Mechanism
  -----------------------+-----------------------+-------------------------------
  **Role**              | Model & predict       | Implement specific game
                        | strategic behavior    | mechanisms for allocation
  -----------------------+-----------------------+-------------------------------
  **Abstraction Level** | Conceptual framework  | Concrete protocol/system
  -----------------------+-----------------------+-------------------------------
  **Scope**             | All strategic         | Specifically resource
                        | interactions           | allocation via prices
  -----------------------+-----------------------+-------------------------------
  **Output**            | Predictions,          | Prices, allocations,
                        | equilibria            | trades
  -----------------------+-----------------------+-------------------------------
  **Key Tool**          | Nash Equilibrium      | Supply-demand curve,
                        |                       | auction rules
  -----------------------+-----------------------+-------------------------------
  **Relationship**      | Market mechanisms     | A market IS a specific
                        | ARE instances of       | game form designed through
                        | mechanism design       | mechanism design (applied
                        | (a branch of game      | game theory)
                        | theory)               |
  -----------------------+-----------------------+-------------------------------

  ANALOGY:
    Game theory is to markets as physics is to bridges.
    Physics provides the laws; bridges are specific engineered
    structures that obey those laws. Game theory provides
    the principles; markets are specific mechanisms designed
    within those principles.
```

### 9.2 Game Theory vs. Parliament (Voting)

```
  Feature               | Game Theory           | Parliament Pattern
  -----------------------+-----------------------+-------------------------------
  **Core Action**       | Strategically choose  | Vote on proposals
                        | actions (not just     |
                        | voting)               |
  -----------------------+-----------------------+-------------------------------
  **Decision Rule**     | Equilibrium concepts  | Voting rules (majority,
                        | (Nash, Bayes-Nash,    | Borda, approval,
                        |  subgame-perfect)     |  Condorcet)
  -----------------------+-----------------------+-------------------------------
  **Strategy Space**    | Rich: any action in   | Restricted: vote for
                        | the agent's set       | an option
  -----------------------+-----------------------+-------------------------------
  **Key Overlap**       | Strategic voting is   | Voting rules ARE mechanisms
                        | modeled by game       | studied through the lens of
                        | theory (voters have   | mechanism design in game
                        | preferences, vote     | theory
                        | strategically)        |
  -----------------------+-----------------------+-------------------------------
  **Analysis**          | Game theory analyzes  | Parliament pattern analyzes
                        | ALL strategic choices | voting PROCEDURES specifically
  -----------------------+-----------------------+-------------------------------
  **Design Focus**      | Align incentives to   | Choose voting rule that
                        | produce desired       | best aggregates preferences
                        | behavior              |
  -----------------------+-----------------------+-------------------------------
  **Example Question**  | "Will agents           | "Should we use majority
                        | cooperate or defect?" | or approval voting?"

  Game theory provides the analytical tools to understand strategic
  voting (e.g., Gibbard-Satterthwaite theorem: no non-dictatorial
  voting rule is strategy-proof). The Parliament pattern implements
  concrete voting protocols whose properties are analyzed through
  game theory.
```

### 9.3 Summary Comparison Across Patterns

```
  Pattern        | Strategic | Predicts | Designs | Computational
                 | Modeling  | Behavior | Rules   | Load
  ---------------+-----------+----------+---------+---------------
  Game Theory    | Explicit  | Yes      | Yes     | High (equilibrium)
  Market         | Implicit  | Yes      | No      | Medium (prices)
  Parliament     | Implicit  | Yes      | No      | Low (counting)
  Coalition      | Explicit  | Yes      | Yes     | High (Shapley, core)
  Bidding         | Explicit  | Yes      | Yes     | Medium (auctions)
```

---

## 10. Core Advantages

### 10.1 Rigorous Mathematical Foundation

Game theory is not heuristic or ad-hoc. It provides:

- **Existence proofs**: Nash's theorem guarantees that every finite game has at least one equilibrium.
- **Uniqueness conditions**: Certain games (strictly concave, potential games) have unique equilibria.
- **Convergence guarantees**: Under specific dynamics (fictitious play, gradient ascent), learning converges to equilibrium.
- **Optimality bounds**: Mechanisms can be proven to achieve a fraction of optimal welfare (approximation guarantees).

This rigor means results are **provable**, not just empirically observed. When a system is designed with game theory, you can prove properties like incentive compatibility and individual rationality.

### 10.2 Predictive Power for Agent Behavior

Given knowledge of agent payoffs, game theory predicts:

```
  PREDICTIVE CAPABILITIES

  +----------------------------------------------------+
  | Setting                | Prediction                |
  |------------------------+---------------------------|
  | Vickrey Auction        | Bidders bid truthfully    |
  | Prisoner's Dilemma     | Both defect (one-shot)   |
  | Prisoner's Dilemma     | Possible cooperation     |
  |   (repeated, infinite)  | (Folk Theorem)           |
  | Sealed-bid first-price | Shading: bid < value     |
  | Auctions with common   | Winner's curse: winner   |
  |   value                 | overpays                 |
  | Coordination game      | Convergence to one of    |
  |                        | multiple equilibria      |
  | Zero-sum game          | Minimax strategy is      |
  |                        | optimal                  |
  +----------------------------------------------------+

  These predictions guide system design even before deployment.
```

### 10.3 Mechanism Design: Building Incentive-Aligned Systems

The most powerful advantage: game theory provides a **constructive methodology** for designing multi-agent systems:

```
  DESIGN PROCESS

  1. DEFINE DESIRED OUTCOME
     - Efficient allocation? Fair distribution? Truthful reporting?

  2. IDENTIFY AGENT INCENTIVES
     - What do agents want? What are their private information?
     - What would they do in a naive system?

  3. DESIGN INCENTIVE STRUCTURE
     - Choose allocation rule
     - Design payment/penalty scheme
     - Ensure incentive compatibility (IC)
     - Ensure individual rationality (IR)
     - Optional: budget balance (BB)

  4. VERIFY PROPERTIES
     - Check IC: can any agent gain by misreporting?
     - Check IR: do all agents want to participate?
     - Check efficiency: does the mechanism maximize welfare?

  5. DEPLOY AND MONITOR
     - Agents play the mechanism
     - Verify predicted behavior matches observed behavior
     - Adjust parameters if needed
```

### 10.4 Modularity and Composability

Game-theoretic mechanisms compose well:

- **Layered mechanisms**: An auction can use a payment mechanism internally, which uses a redistribution mechanism.
- **Hierarchical games**: A multi-level system where each level is a game (e.g., agents bid for tasks, then negotiate sub-tasks).
- **Combined incentives**: Agents can participate in multiple simultaneous games; total utility can be additive across games.

### 10.5 Side-Effect Transparency

Unlike black-box coordination approaches (deep learning), game-theoretic mechanisms are transparent:

- Rules are explicit and interpretable.
- Agent incentives can be audited.
- Properties can be formally verified.
- Failures can be diagnosed and fixed.

---

## 11. Engineering Optimizations

### 11.1 Approximation Algorithms for Large Games

When exact computation is infeasible, use approximations:

```
  APPROXIMATION STRATEGIES

  1. epsilon-Nash Equilibrium
     No agent can improve by more than epsilon
     Much easier to compute than exact equilibrium
     Use: anytime algorithms, gradually refine

  2. Coarse Correlated Equilibrium
     Agents follow a correlating signal (mediator)
     Set of all CCE is convex, easy to compute
     Use: regret minimization algorithms

  3. Potential Game Approximation
     If the game is approximately a potential game,
     simple learning dynamics converge
     Use: for games with structure

  4. Sampling/Simulation
     Sample a subset of agents or strategies
     Compute equilibrium on the reduced game
     Use: for massive agent populations

  5. Mean-Field Approximation
     Assume each agent interacts with the "average" agent
     Instead of N agents, solve a single-agent problem
     Use: for homogeneous agents, N > 1000
```

### 11.2 Focus on Repeated Games

Repeated games offer several engineering advantages over one-shot:

```
  ADVANTAGES OF REPEATED GAMES

  1. LEARNING
     Agents can learn from experience
     No need to pre-compute equilibrium
     No-regret algorithms guarantee convergence
     to correlated equilibrium

  2. ROBUSTNESS
     Errors can be corrected in subsequent rounds
     Agents adapt to opponent behavior
     Mistakes don't permanently damage outcomes

  3. SIMPLICITY
     Simple strategies (Tit-for-Tat) perform well
     No need for complex opponent modeling
     Clear, implementable algorithms

  4. COOPERATION EMERGENCE
     Cooperation becomes rational in repeated games
     Even selfish agents may cooperate for future benefit
     Self-enforcing without external enforcement

  TRACTABLE REPEATED GAME APPROACH:

    for round in range(num_rounds):
        action = choose_action(opponent_history, my_history)
        opponent_action = get_opponent_action()
        my_payoff = compute_payoff(action, opponent_action)
        history.append((action, opponent_action, my_payoff))
        update_strategy(history)
        # optional: explicitly model opponent
```

### 11.3 Regret Minimization

Instead of computing equilibrium, minimize regret:

```
  REGRET MINIMIZATION FRAMEWORK

  External Regret:
    How much better could I have done by playing a SINGLE
    fixed action in all rounds?

    Regret_T = max_a sum_t (u_t(a) - u_t(a_t))

  Internal Regret:
    How much better could I have done by replacing EVERY
    instance of action a with action a'?

  No-Regret Algorithm Guarantee:
    Regret_T / T -> 0 as T -> infinity
    i.e., average performance approaches best fixed action

  IMPLEMENTATION: Multiplicative Weights Update

    # Initialize weights
    w = {a: 1.0 for a in actions}

    for t in range(T):
        # Choose action proportional to weights
        probs = {a: w[a] / sum(w.values()) for a in actions}
        action = sample(probs)

        # Observe payoff for ALL actions (full info)
        payoffs = get_all_payoffs()

        # Update weights
        for a in actions:
            w[a] = w[a] * (1 + eta * payoffs[a])

  PROPERTIES:
    - Guarantees regret <= O(sqrt(T) * log|A|)
    - If ALL agents use no-regret learning:
      empirical play converges to CORRELATED EQUILIBRIUM
    - Computationally efficient
    - No opponent modeling needed
```

### 11.4 Shapley Value Approximation

Since exact Shapley value computation is #P-complete, use Monte Carlo sampling:

```
  MONTE CARLO SHAPLEY VALUE

  # Approximate Shapley value via random permutations
  def approximate_shapley(N, v, num_samples=1000):
      """
      N: set of all agents
      v: value function v(S) -> float
      num_samples: number of random permutations to sample
      """
      contributions = {i: 0 for i in N}

      for _ in range(num_samples):
          # Random permutation of agents
          perm = shuffle(list(N))
          S = set()

          for i in perm:
              # i's marginal contribution to S
              marginal = v(S | {i}) - v(S)
              contributions[i] += marginal
              S.add(i)

      # Average across samples
      total = sum(contributions.values())
      return {i: c / num_samples for i, c in contributions.items()}

  Properties:
    - O(num_samples * n * cost_of_v) time
    - Error decreases as O(1/sqrt(num_samples))
    - Accurate within 1-5% with 1000-10000 samples
    - Much faster than exact computation for large n
```

### 11.5 Practical Mechanism Design Guidelines

| Principle | Guideline | Why |
|-----------|-----------|-----|
| Simplicity | Use simple, transparent rules | Agents can't respond optimally to complex rules they don't understand |
| Robustness | Test against strategic behavior | Agents will find loopholes you didn't anticipate |
| Incremental | Start simple, add complexity as needed | Over-designed mechanisms fail for unexpected reasons |
| Information | Minimize what agents need to reveal | More information = more strategic manipulation |
| Alignment | Align agent rewards with system goals | Direct incentive is stronger than indirect |
| Defaults | Choose good default strategies | Agents often accept defaults (status quo bias) |
| Penalties | Make penalties proportional and credible | Empty threats don't deter defection |

### 11.6 Software Architecture for Game-Theoretic Systems

```
  +----------------------------------------------------------+
  |              GAME-THEORETIC AGENT SYSTEM                  |
  +----------------------------------------------------------+
  |                                                            |
  |  +------------------+   +-----------------------------+    |
  |  | Mechanism Layer  |   | Strategy Layer              |    |
  |  |                  |   |                             |    |
  |  | - Auction rules  |   | - Strategy selection        |    |
  |  | - Payment rules  |   | - Opponent modeling         |    |
  |  | - Allocation     |   | - Learning/adaptation       |    |
  |  | - Verification   |   | - Regret minimization       |    |
  |  +--------+---------+   +-------------+---------------+    |
  |           |                             |                  |
  |  +--------+-----------------------------+----------+       |
  |  |                    Game State Layer             |       |
  |  |                                                 |       |
  |  |  - Payoff computation   - History tracking      |       |
  |  |  - Agent registry       - Equilibrium finding   |       |
  |  |  - Type/belief storage  - Reward distribution   |       |
  |  +--------+----------------------------------------+       |
  |           |                                                 |
  |  +--------+---------+                                       |
  |  | Communication    |                                       |
  |  | Layer            |                                       |
  |  |                  |                                       |
  |  | - Message passing|                                       |
  |  | - Action exchange|                                       |
  |  | - State broadcast|                                       |
  |  +------------------+                                       |
  |                                                            |
  +----------------------------------------------------------+
```

---

## 12. Code Examples

### 12.1 Prisoner's Dilemma Payoff Matrix and Agent Strategy

```python
"""
Prisoner's Dilemma -- Basic implementation

Actions: Cooperate (C) or Defect (D)
Payoff matrix:
                  Opponent
                C       D
        Player C (3,3)  (0,5)
               D (5,0)  (1,1)
"""

from enum import Enum
from typing import Tuple, Dict, List


class Action(Enum):
    COOPERATE = "C"
    DEFECT = "D"


# Payoff matrix: (my_payoff, opponent_payoff)
PAYOFF_MATRIX: Dict[Tuple[Action, Action], Tuple[int, int]] = {
    (Action.COOPERATE, Action.COOPERATE): (3, 3),
    (Action.COOPERATE, Action.DEFECT):    (0, 5),
    (Action.DEFECT,    Action.COOPERATE): (5, 0),
    (Action.DEFECT,    Action.DEFECT):    (1, 1),
}


def get_payoff(my_action: Action, opp_action: Action) -> Tuple[int, int]:
    """Return (my_payoff, opponent_payoff) for the given action pair."""
    return PAYOFF_MATRIX[(my_action, opp_action)]


# === Agent Strategies ===

def always_cooperate(history: List[Tuple[Action, Action]]) -> Action:
    """Always cooperates. Gets exploited by defectors."""
    return Action.COOPERATE


def always_defect(history: List[Tuple[Action, Action]]) -> Action:
    """Always defects. Never leaves money on the table."""
    return Action.DEFECT


def random_strategy(history: List[Tuple[Action, Action]]) -> Action:
    """Randomly picks C or D with equal probability."""
    import random
    return random.choice([Action.COOPERATE, Action.DEFECT])


def tit_for_tat(history: List[Tuple[Action, Action]]) -> Action:
    """
    Classic Tit-for-Tat strategy:
    - First move: Cooperate
    - Subsequent moves: Copy opponent's previous action
    """
    if not history:
        return Action.COOPERATE
    # Copy opponent's last action
    return history[-1][1]


def grudger(history: List[Tuple[Action, Action]]) -> Action:
    """
    Cooperate until opponent defects once, then defect forever.
    Also called: Grim Trigger
    """
    if not history:
        return Action.COOPERATE
    # Check if opponent ever defected
    for _, opp_action in history:
        if opp_action == Action.DEFECT:
            return Action.DEFECT
    return Action.COOPERATE


def tit_for_two_tats(history: List[Tuple[Action, Action]]) -> Action:
    """
    Only defect after opponent defects TWICE in a row.
    More forgiving than standard Tit-for-Tat.
    """
    if len(history) < 2:
        return Action.COOPERATE
    if history[-1][1] == Action.DEFECT and history[-2][1] == Action.DEFECT:
        return Action.DEFECT
    return Action.COOPERATE


# === Game Runner ===

def play_round(
    action_a: Action,
    action_b: Action,
    history_a: List[Tuple[Action, Action]],
    history_b: List[Tuple[Action, Action]],
) -> Tuple[int, int]:
    """
    Play a single round, record in both histories.
    Returns (payoff_a, payoff_b).
    """
    payoff_a, payoff_b = get_payoff(action_a, action_b)
    history_a.append((action_a, action_b))
    history_b.append((action_b, action_a))  # Reversed for agent B's perspective
    return payoff_a, payoff_b


def play_game(
    strategy_a,
    strategy_b,
    num_rounds: int = 10,
    verbose: bool = True,
) -> Tuple[int, int, List[Tuple[Action, Action]]]:
    """
    Play a multi-round Prisoner's Dilemma.
    Returns (total_a, total_b, history).
    """
    history_a: List[Tuple[Action, Action]] = []
    history_b: List[Tuple[Action, Action]] = []
    total_a = 0
    total_b = 0

    if verbose:
        print(f"{'Round':<6} {'A':<10} {'B':<10} {'Payoffs':<15}")
        print("-" * 45)

    for round_num in range(num_rounds):
        action_a = strategy_a(history_a)
        action_b = strategy_b(history_b)

        payoff_a, payoff_b = play_round(
            action_a, action_b,
            history_a, history_b,
        )
        total_a += payoff_a
        total_b += payoff_b

        if verbose:
            print(
                f"{round_num + 1:<6} {action_a.value:<10} "
                f"{action_b.value:<10} ({payoff_a}, {payoff_b})"
            )

    if verbose:
        print("-" * 45)
        print(f"{'Total:':<6} {total_a:<10} {total_b:<10}")
        print(f"Average A: {total_a / num_rounds:.2f}, "
              f"Average B: {total_b / num_rounds:.2f}")

    return total_a, total_b, history_a


# === Tournament ===

def run_tournament():
    """Run a small tournament comparing all strategies."""
    strategies = {
        "Always Cooperate": always_cooperate,
        "Always Defect":    always_defect,
        "Random":           random_strategy,
        "Tit-for-Tat":      tit_for_tat,
        "Grudger":          grudger,
        "Tit-for-Two-Tats": tit_for_two_tats,
    }

    results: Dict[str, Dict[str, Tuple[int, int]]] = {}

    for name_a, strat_a in strategies.items():
        results[name_a] = {}
        for name_b, strat_b in strategies.items():
            total_a, total_b, _ = play_game(
                strat_a, strat_b,
                num_rounds=50,
                verbose=False,
            )
            results[name_a][name_b] = (total_a, total_b)

    # Display results matrix (Agent A scores)
    print("Tournament Results (Agent A total payoff over 50 rounds):")
    print(f"{'A \\ B':<20}", end="")
    for name_b in strategies:
        print(f"{name_b:<20}", end="")
    print()
    print("-" * (20 + 20 * len(strategies)))

    for name_a in strategies:
        print(f"{name_a:<20}", end="")
        for name_b in strategies:
            score = results[name_a][name_b][0]
            print(f"{score:<20}", end="")
        print()

    # Total scores
    print("\nStrategy Rankings (total score):")
    rankings = []
    for name_a in strategies:
        total = sum(results[name_a][name_b][0] for name_b in strategies)
        rankings.append((total, name_a))
    rankings.sort(reverse=True)
    for rank, (total, name) in enumerate(rankings, 1):
        print(f"  {rank}. {name:<20} {total}")


if __name__ == "__main__":
    print("=== Prisoner's Dilemma: Tit-for-Tat vs Always Defect ===\n")
    play_game(tit_for_tat, always_defect, num_rounds=10)

    print("\n=== Tournament ===\n")
    run_tournament()
```

---

### 12.2 Iterated Game with Tit-for-Tat and Cooperative Emergence

```python
"""
Iterated Prisoner's Dilemma with multiple agents and
the emergence of cooperative clusters through spatial structure.
"""

import random
from typing import Callable, List, Tuple
from enum import Enum


class Action(Enum):
    C = "C"  # Cooperate
    D = "D"  # Defect


# Payoff matrix
R = 3  # Reward for mutual cooperation
T = 5  # Temptation to defect
S = 0  # Sucker's payoff
P = 1  # Punishment for mutual defection


def get_payoff(a: Action, b: Action) -> Tuple[int, int]:
    if a == Action.C and b == Action.C:
        return (R, R)
    if a == Action.C and b == Action.D:
        return (S, T)
    if a == Action.D and b == Action.C:
        return (T, S)
    return (P, P)


# === Strategy Implementations ===

def strategy_tit_for_tat(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    if not opp_history:
        return Action.C
    return opp_history[-1]


def strategy_always_defect(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    return Action.D


def strategy_always_cooperate(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    return Action.C


def strategy_random(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    return random.choice([Action.C, Action.D])


def strategy_suspicious_tit_for_tat(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    """
    Defects on first move, then copies opponent.
    Performs much worse than standard Tit-for-Tat.
    """
    if not opp_history:
        return Action.D
    return opp_history[-1]


def strategy_pavlov(
    my_history: List[Action],
    opp_history: List[Action],
) -> Action:
    """
    Win-stay, lose-shift.
    If last round outcome was good, repeat action.
    Otherwise, switch.
    """
    if not my_history:
        return Action.C

    last_my = my_history[-1]
    last_opp = opp_history[-1]
    last_payoff = get_payoff(last_my, last_opp)[0]

    # If payoff was high (R or T), repeat last action
    if last_payoff >= R:  # R=3 is the threshold
        return last_my
    # Otherwise, switch
    return Action.C if last_my == Action.D else Action.D


# === Spatial Tournament (cooperation emergence) ===

class SpatialAgent:
    """Agent on a grid, plays with neighbors."""

    def __init__(
        self,
        agent_id: int,
        strategy: Callable,
        strategy_name: str,
    ):
        self.agent_id = agent_id
        self.strategy = strategy
        self.strategy_name = strategy_name
        self.history: List[Tuple[Action, Action]] = []

    def play(self, opponent: "SpatialAgent") -> int:
        """Play one round against opponent. Returns payoff."""
        my_action = self.strategy(
            [h[0] for h in self.history],
            [h[1] for h in self.history],
        )
        opp_action = opponent.strategy(
            [h[0] for h in opponent.history],
            [h[1] for h in opponent.history],
        )
        self.history.append((my_action, opp_action))
        opponent.history.append((opp_action, my_action))
        return get_payoff(my_action, opp_action)[0]


def run_spatial_tournament():
    """
    Demonstrate that tit-for-tat creates clusters of cooperation
    in a spatial environment.
    """
    # Create a grid with mixed strategies
    grid_size = 10
    agents: List[List[SpatialAgent]] = []
    agent_id = 0

    strategies = [
        (strategy_tit_for_tat, "TFT"),
        (strategy_always_defect, "ALL_D"),
        (strategy_always_cooperate, "ALL_C"),
        (strategy_random, "RAND"),
        (strategy_pavlov, "PAV"),
    ]

    for i in range(grid_size):
        row = []
        for j in range(grid_size):
            strat, name = random.choice(strategies)
            row.append(SpatialAgent(agent_id, strat, name))
            agent_id += 1
        agents.append(row)

    # Count initial strategy distribution
    counts = {}
    for row in agents:
        for a in row:
            counts[a.strategy_name] = counts.get(a.strategy_name, 0) + 1

    print("Initial strategy distribution:")
    for name, count in sorted(counts.items()):
        print(f"  {name}: {count}")

    # Run generations
    num_generations = 50
    for gen in range(num_generations):
        # Each agent plays with neighbors
        payoffs: List[List[int]] = [[0] * grid_size for _ in range(grid_size)]

        for i in range(grid_size):
            for j in range(grid_size):
                neighbors = []
                for di in (-1, 0, 1):
                    for dj in (-1, 0, 1):
                        if di == 0 and dj == 0:
                            continue
                        ni, nj = i + di, j + dj
                        if 0 <= ni < grid_size and 0 <= nj < grid_size:
                            neighbors.append((ni, nj))

                total = 0
                for ni, nj in neighbors:
                    total += agents[i][j].play(agents[ni][nj])
                payoffs[i][j] = total

        # Reproduction: agents copy the most successful neighbor's strategy
        new_agents = [[None] * grid_size for _ in range(grid_size)]

        for i in range(grid_size):
            for j in range(grid_size):
                neighbors = [(i, j)]
                for di in (-1, 0, 1):
                    for dj in (-1, 0, 1):
                        if di == 0 and dj == 0:
                            continue
                        ni, nj = i + di, j + dj
                        if 0 <= ni < grid_size and 0 <= nj < grid_size:
                            neighbors.append((ni, nj))

                # Find neighbor with highest payoff
                best_i, best_j = max(
                    neighbors,
                    key=lambda pos: payoffs[pos[0]][pos[1]],
                )
                best_agent = agents[best_i][best_j]

                new_agents[i][j] = SpatialAgent(
                    agents[i][j].agent_id,
                    best_agent.strategy,
                    best_agent.strategy_name,
                )

        agents = new_agents

        # Report every 10 generations
        if (gen + 1) % 10 == 0:
            counts = {}
            for row in agents:
                for a in row:
                    counts[a.strategy_name] = counts.get(a.strategy_name, 0) + 1
            print(f"\nGeneration {gen + 1}:")
            for name, count in sorted(counts.items()):
                print(f"  {name}: {count}")

    # Final results
    counts = {}
    for row in agents:
        for a in row:
            counts[a.strategy_name] = counts.get(a.strategy_name, 0) + 1
    print(f"\nFinal strategy distribution (generation {num_generations}):")
    for name, count in sorted(counts.items()):
        print(f"  {name}: {count}")
    print("\nTit-for-Tat dominates due to: niceness + retaliation + forgiveness")


if __name__ == "__main__":
    run_spatial_tournament()
```

---

### 12.3 Shapley Value Computation for Reward Distribution

```python
"""
Shapley Value -- Fair reward distribution among cooperative agents.

Given a set of agents who together produce value, the Shapley value
distributes the total reward proportionally to each agent's marginal
contribution, averaged over all possible coalition formation orders.
"""

import itertools
from typing import Set, Dict, Callable, List
import random


def shapley_value_exact(
    agents: Set[str],
    value_function: Callable[[Set[str]], float],
) -> Dict[str, float]:
    """
    Compute exact Shapley values for all agents.

    Args:
        agents: Set of agent identifiers (e.g., {"agent_1", "agent_2"})
        value_function: v(S) returns the value generated by coalition S

    Returns:
        Dictionary mapping agent -> Shapley value

    Complexity: O(n! * 2^n) -- exponential, only feasible for n <= 10

    Formula:
                  |S|! (n - |S| - 1)!
    phi_i = sum  --------------------- * (v(S U {i}) - v(S))
            S subset    n!
            N\{i}
    """
    n = len(agents)
    agents_list = list(agents)
    phi = {a: 0.0 for a in agents}

    for i, agent in enumerate(agents_list):
        # Iterate over all subsets S not containing agent i
        other_agents = [a for a in agents_list if a != agent]
        for r in range(n):
            for subset in itertools.combinations(other_agents, r):
                S = set(subset)
                marginal = value_function(S | {agent}) - value_function(S)
                weight = (
                    factorial(r) * factorial(n - r - 1) / factorial(n)
                )
                phi[agent] += marginal * weight

    return phi


def factorial(k: int) -> int:
    """Compute k!"""
    return 1 if k <= 1 else k * factorial(k - 1)


def shapley_value_monte_carlo(
    agents: Set[str],
    value_function: Callable[[Set[str]], float],
    num_samples: int = 10000,
    seed: int = 42,
) -> Dict[str, float]:
    """
    Approximate Shapley values using Monte Carlo sampling.

    This is the practical approach for systems with many agents.
    Error decreases as O(1 / sqrt(num_samples)).

    Args:
        agents: Set of agent identifiers
        value_function: v(S) returns the value generated by coalition S
        num_samples: Number of random permutations to sample
        seed: Random seed for reproducibility

    Returns:
        Dictionary mapping agent -> approximate Shapley value
    """
    rng = random.Random(seed)
    agents_list = list(agents)
    contributions = {a: 0.0 for a in agents_list}

    for _ in range(num_samples):
        # Generate random permutation of agents
        perm = agents_list.copy()
        rng.shuffle(perm)

        coalition = set()
        for agent in perm:
            marginal = value_function(coalition | {agent}) - value_function(
                coalition
            )
            contributions[agent] += marginal
            coalition.add(agent)

    n = len(agents_list)
    return {
        a: c / num_samples for a, c in contributions.items()
    }


# === Example: ML Model Training Reward Distribution ===

def example_ml_training_reward():
    """
    Three AI agents collaborate to train a model.
    Each agent brings different capabilities.
    The value is the model's accuracy.
    """
    agents = {"data_provider", "model_trainer", "hyperparameter_tuner"}

    # Value function: accuracy of the resulting model
    def accuracy(coalition: Set[str]) -> float:
        if len(coalition) == 0:
            return 0.0  # No agents, no model

        base = 0.5  # Baseline accuracy (random guessing)
        bonuses = {
            frozenset({"data_provider"}): 0.05,
            frozenset({"model_trainer"}): 0.10,
            frozenset({"hyperparameter_tuner"}): 0.03,
            frozenset({"data_provider", "model_trainer"}): 0.25,
            frozenset({"data_provider", "hyperparameter_tuner"}): 0.12,
            frozenset({"model_trainer", "hyperparameter_tuner"}): 0.20,
            frozenset(
                {"data_provider", "model_trainer", "hyperparameter_tuner"}
            ): 0.40,
        }
        return base + bonuses.get(frozenset(coalition), 0.0)

    print("=== Shapley Value: ML Model Training ===")
    print(f"Agents: {agents}")
    print(f"\nValue function (accuracy):")
    print(f"  Empty set:                 {accuracy(set()):.2f}")
    print(f"  { { 'data_provider' }}:                {accuracy({'data_provider'}):.2f}")
    print(f"  { { 'model_trainer' }}:               {accuracy({'model_trainer'}):.2f}")
    print(f"  { { 'hyperparameter_tuner' }}:        {accuracy({'hyperparameter_tuner'}):.2f}")
    print(f"  { { 'data_provider', 'model_trainer' }}:       {accuracy({'data_provider', 'model_trainer'}):.2f}")
    print(f"  { { 'data_provider', 'hyperparameter_tuner' }}:{accuracy({'data_provider', 'hyperparameter_tuner'}):.2f}")
    print(f"  { { 'model_trainer', 'hyperparameter_tuner' }}:{accuracy({'model_trainer', 'hyperparameter_tuner'}):.2f}")
    print(f"  All three:                {accuracy(set(agents)):.2f}")

    # Exact Shapley values
    exact = shapley_value_exact(agents, accuracy)
    print(f"\nExact Shapley Values (reward distribution):")
    total_accuracy = accuracy(set(agents))
    for agent, value in sorted(exact.items(), key=lambda x: -x[1]):
        print(f"  {agent:<25}: {value:.4f}  "
              f"({(value / total_accuracy) * 100:.1f}% of total)")

    # Verify efficiency: sum equals total coalition value
    total = sum(exact.values())
    print(f"\n  Sum of Shapley values: {total:.4f}  "
          f"(should equal total accuracy: {total_accuracy:.4f})")

    # Monte Carlo approximation
    mc = shapley_value_monte_carlo(agents, accuracy, num_samples=50000)
    print(f"\nMonte Carlo Approximation (50,000 samples):")
    for agent in sorted(mc, key=lambda a: -mc[a]):
        print(f"  {agent:<25}: {mc[agent]:.4f}  "
              f"(error: {abs(mc[agent] - exact[agent]):.4f})")


# === Example: Task Contribution Reward ===

def example_task_contribution():
    """
    Multiple agents contribute to completing a complex task.
    Reward is proportional to contribution.
    """
    agents = {"A", "B", "C", "D"}

    def task_value(coalition: Set[str]) -> float:
        """
        Value generated by a team of agents.
        Some agents are specialists, some are generalists.
        """
        if len(coalition) == 0:
            return 0.0

        # Base per-agent contribution
        base = len(coalition) * 0.5

        # Synergy bonuses for specific agent combinations
        agent_list = frozenset(coalition)

        synergy_bonus = 0.0
        if agent_list >= frozenset({"A", "B"}):
            synergy_bonus += 3.0  # A and B work very well together
        if agent_list >= frozenset({"C", "D"}):
            synergy_bonus += 1.5  # C and D have moderate synergy

        # Diminishing returns for large teams
        if len(coalition) >= 4:
            synergy_bonus *= 0.8

        return base + synergy_bonus

    print("\n=== Shapley Value: Task Contribution ===")
    print(f"Agents: {agents}")
    print(f"Total value (all agents): {task_value(agents):.2f}")

    exact = shapley_value_exact(agents, task_value)
    print(f"\nShapley Values:")
    for agent, value in sorted(exact.items(), key=lambda x: -x[1]):
        print(f"  Agent {agent}: {value:.2f}")

    total = sum(exact.values())
    print(f"\n  Sum: {total:.2f}  (should equal {task_value(agents):.2f})")
    print(f"\nNote: Agents A and B get more because they have higher synergy.")


if __name__ == "__main__":
    example_ml_training_reward()
    example_task_contribution()
```

---

### 12.4 Auction Mechanism (Vickrey Auction) Implementation

```python
"""
Vickrey (Second-Price Sealed-Bid) Auction.

Properties:
  - TRUTHFUL: Bidding true value is a dominant strategy
  - EFFICIENT: Item goes to the agent who values it most
  - INDIVIDUALLY RATIONAL: Winners pay <= their bid
"""

from typing import List, Dict, Tuple, Optional
from dataclasses import dataclass
from abc import ABC, abstractmethod


@dataclass
class Bid:
    """A bid from an agent in an auction."""
    agent_id: str
    value: float  # The agent's TRUE valuation (private)
    bid_amount: Optional[float] = None  # The amount actually bid


class AuctionMechanism(ABC):
    """Abstract base class for auction mechanisms."""

    @abstractmethod
    def run(
        self, bids: List[Bid]
    ) -> Tuple[Optional[str], Optional[float], Dict[str, float]]:
        """
        Run the auction.

        Returns:
            (winner_id, payment, all_payments)
            winner_id: The agent who wins (None if no bids)
            payment: Amount the winner pays (None if no bids)
            all_payments: Dictionary of all payments (incl. losers pay 0)
        """
        ...


class VickreyAuction(AuctionMechanism):
    """
    Vickrey (Second-Price Sealed-Bid) Auction.

    Rules:
      1. Each agent submits a sealed bid
      2. Highest bidder wins
      3. Winner pays the SECOND HIGHEST bid price

    Strategic property:
      Bidding truthfully (bid_amount = true_value) is a WEAKLY DOMINANT
      strategy. No agent can ever do better by bidding != true value,
      regardless of what other agents do.
    """

    def __init__(self, reserve_price: float = 0.0):
        self.reserve_price = reserve_price

    def run(
        self, bids: List[Bid]
    ) -> Tuple[Optional[str], Optional[float], Dict[str, float]]:
        if not bids:
            return None, None, {}

        # Use bid_amount if provided, otherwise use true value
        amounts = []
        for bid in bids:
            amount = bid.bid_amount if bid.bid_amount is not None else bid.value
            amounts.append((bid.agent_id, amount, bid.value))

        # Sort by bid amount descending
        amounts.sort(key=lambda x: -x[1])

        winner_id, winning_bid, true_value = amounts[0]

        # Check reserve price
        if winning_bid < self.reserve_price:
            # No sale
            payments = {bid.agent_id: 0.0 for bid in bids}
            return None, None, payments

        # Second price
        if len(amounts) >= 2:
            payment = amounts[1][1]  # Second highest bid
        else:
            payment = self.reserve_price

        payments = {bid.agent_id: 0.0 for bid in bids}
        payments[winner_id] = payment

        return winner_id, payment, payments


class FirstPriceAuction(AuctionMechanism):
    """
    First-Price Sealed-Bid Auction (for comparison).

    Rules:
      1. Each agent submits a sealed bid
      2. Highest bidder wins
      3. Winner pays their OWN bid

    Strategic property:
      Bidding truthfully is NOT optimal. Agents must "shade" their
      bids below true value (bid shading). Optimal shading depends on
      beliefs about other bidders.
    """

    def run(
        self, bids: List[Bid]
    ) -> Tuple[Optional[str], Optional[float], Dict[str, float]]:
        if not bids:
            return None, None, {}

        amounts = []
        for bid in bids:
            amount = bid.bid_amount if bid.bid_amount is not None else bid.value
            amounts.append((bid.agent_id, amount))

        amounts.sort(key=lambda x: -x[1])
        winner_id, payment = amounts[0]

        payments = {bid.agent_id: 0.0 for bid in bids}
        payments[winner_id] = payment

        return winner_id, payment, payments


class AllPayAuction(AuctionMechanism):
    """
    All-Pay Auction.

    Rules:
      1. Each agent submits a sealed bid
      2. Highest bidder wins
      3. EVERYONE pays their bid (not just the winner)

    Strategic property:
      Very aggressive for high-value items. Used in contests,
      lobbying, and R&D races.
    """

    def run(
        self, bids: List[Bid]
    ) -> Tuple[Optional[str], Optional[float], Dict[str, float]]:
        if not bids:
            return None, None, {}

        amounts = []
        for bid in bids:
            amount = bid.bid_amount if bid.bid_amount is not None else bid.value
            amounts.append((bid.agent_id, amount))

        amounts.sort(key=lambda x: -x[1])
        winner_id, _ = amounts[0]

        payments = {}
        for agent_id, amount in amounts:
            payments[agent_id] = amount

        return winner_id, amounts[0][1], payments


class CombinatorialVickreyAuction:
    """
    Vickrey-Clarke-Groves (VCG) mechanism for allocating
    MULTIPLE items where agents can bid on bundles.

    This is the general form: truthful bidding is dominant,
    and the mechanism allocates to maximize total value.
    """

    def __init__(self, items: List[str]):
        self.items = items

    def run(
        self, bids: Dict[str, Dict[frozenset, float]]
    ) -> Tuple[Dict[str, frozenset], Dict[str, float]]:
        """
        bids: {agent_id: {bundle: value}}
              e.g., {"A": {frozenset({"item1"}): 10, frozenset({"item1","item2"}): 15}}

        Returns:
            (allocation, payments)
            allocation: {agent_id: won_bundle}
            payments: {agent_id: payment}
        """
        agents = list(bids.keys())
        all_bundles = list(
            set().union(*[set(b.keys()) for b in bids.values()])
        )

        # Step 1: Find the allocation that maximizes total value
        # (In practice, this is NP-hard -- use integer programming)
        best_allocation, best_value = self._find_optimal_allocation(
            agents, bids, set(self.items)
        )

        # Step 2: Compute VCG payments
        # Each agent pays the "externality" they impose on others:
        # (value of optimal allocation without agent) -
        # (value to others in current allocation)
        payments = {}
        for agent in agents:
            if best_allocation.get(agent, frozenset()) == frozenset():
                payments[agent] = 0.0
                continue

            # Value of optimal allocation without this agent
            other_agents = [a for a in agents if a != agent]
            _, value_without = self._find_optimal_allocation(
                other_agents, bids, set(self.items)
            )

            # Value to other agents in current allocation
            value_to_others = 0.0
            for a, bundle in best_allocation.items():
                if a != agent and bundle:
                    value_to_others += bids[a].get(bundle, 0.0)

            # Payment = externality imposed
            payments[agent] = value_without - value_to_others

        return best_allocation, payments

    def _find_optimal_allocation(
        self,
        agents: List[str],
        bids: Dict[str, Dict[frozenset, float]],
        items: set,
    ) -> Tuple[Dict[str, frozenset], float]:
        """
        Brute-force optimal allocation (exponential).
        In practice, use constraint programming or integer linear programming.
        Only feasible for small numbers of items and agents.
        """
        best_value = 0.0
        best_allocation = {a: frozenset() for a in agents}

        # Generate all possible allocations
        # (Simplified: assign bundles to first k agents)
        if len(agents) == 0:
            return best_allocation, best_value

        agent = agents[0]
        agent_bids = bids[agent]

        # Option 1: Agent gets nothing
        alloc, val = self._find_optimal_allocation(
            agents[1:], bids, items
        )
        if val > best_value:
            best_value = val
            best_allocation = {**alloc, agent: frozenset()}

        # Option 2: Agent gets a bundle
        for bundle, value in agent_bids.items():
            if bundle.issubset(items):
                alloc, val = self._find_optimal_allocation(
                    agents[1:], bids, items - bundle
                )
                total = value + val
                if total > best_value:
                    best_value = total
                    best_allocation = {**alloc, agent: bundle}

        return best_allocation, best_value


# === Simulation ===

def demonstrate_vickrey_truthfulness():
    """
    Demonstrate that truthful bidding is optimal in Vickrey auction.
    """
    print("=" * 70)
    print("VICKREY AUCTION: Truthful Bidding is Dominant")
    print("=" * 70)

    agents = ["Alice", "Bob", "Charlie"]
    true_values = {"Alice": 100, "Bob": 80, "Charlie": 60}

    print("\nTrue Values:")
    for agent, value in true_values.items():
        print(f"  {agent}: ${value}")

    # Truthful bidding
    print("\n--- Scenario 1: All bid truthfully ---")
    truthful_bids = [
        Bid(aid, val, val) for aid, val in true_values.items()
    ]
    auction = VickreyAuction()
    winner, payment, payments = auction.run(truthful_bids)
    print(f"  Winner: {winner}")
    print(f"  Payment: ${payment:.2f}")
    print(f"  Winner surplus: ${true_values[winner] - payment:.2f}")

    # Alice tries to underbid
    print("\n--- Scenario 2: Alice underbids ($70) ---")
    strategic_bids = [
        Bid("Alice", 100, 70),
        Bid("Bob", 80, 80),
        Bid("Charlie", 60, 60),
    ]
    winner, payment, payments = auction.run(strategic_bids)
    print(f"  Winner: {winner} (Alice lost by underbidding!)")
    print(f"  Alice's surplus: $0 (she could have won with truthful bid)")

    # Alice tries to overbid
    print("\n--- Scenario 3: Alice overbids ($150) ---")
    strategic_bids = [
        Bid("Alice", 100, 150),
        Bid("Bob", 80, 80),
        Bid("Charlie", 60, 60),
    ]
    winner, payment, payments = auction.run(strategic_bids)
    print(f"  Winner: {winner}")
    print(f"  Payment: ${payment:.2f}")
    print(f"  Alice's surplus: ${100 - payment:.2f}")
    print(f"  (Same as truthful bidding -- overbidding didn't help)")

    print("\n=== Key Insight ===")
    print("In a Vickrey auction:")
    print("  - Underbidding: may lose items you should win")
    print("  - Overbidding: may win but pay more than value")
    print("  - Truthful bidding: optimal in all cases")


def auction_comparison():
    """Compare Vickrey vs First-Price vs All-Pay auctions."""
    print("\n" + "=" * 70)
    print("AUCTION TYPE COMPARISON")
    print("=" * 70)

    bidders = [
        Bid("A", 100, 85),   # Shaded bid
        Bid("B", 80, 75),
        Bid("C", 60, 55),
        Bid("D", 50, 45),
    ]

    for auction_class, name in [
        (VickreyAuction, "Vickrey (Second-Price)"),
        (FirstPriceAuction, "First-Price Sealed-Bid"),
        (AllPayAuction, "All-Pay Auction"),
    ]:
        auction = auction_class()
        winner, payment, payments = auction.run(bidders)

        print(f"\n--- {name} ---")
        print(f"  Winner: {winner}")
        print(f"  Winner pays: ${payment:.2f}")
        print(f"  Total payments: ${sum(payments.values()):.2f}")
        print(f"  Auction revenue: ${payment:.2f}")

        if name == "Vickrey (Second-Price)":
            print(f"  Note: Winners pay SECOND price, not their own")
        elif name == "First-Price Sealed-Bid":
            print(f"  Note: Winner pays own bid -- shading required")
        elif name == "All-Pay Auction":
            print(f"  Note: EVERYONE pays their bid")


if __name__ == "__main__":
    demonstrate_vickrey_truthfulness()
    auction_comparison()
```

---

### 12.5 Mechanism Design Flow

```
  MECHANISM DESIGN PROCESS -- Complete Flow Diagram

  +=================================================================+
  |                    MECHANISM DESIGN FLOW                         |
  +=================================================================+
  |                                                                   |
  |  +------------------+                                             |
  |  | GOAL DEFINITION  |                                             |
  |  | "Design a system |                                             |
  |  |  where selfish   |                                             |
  |  |  agents allocate |                                             |
  |  |  resources       |                                             |
  |  |  efficiently"    |                                             |
  |  +--------+---------+                                             |
  |           |                                                       |
  |           v                                                       |
  |  +------------------+                                             |
  |  | ENVIRONMENT      |   Agent types: valuations, costs            |
  |  | ANALYSIS         |   Information: private/public               |
  |  |                  |   Constraints: budget, time, trust          |
  |  +--------+---------+                                             |
  |           |                                                       |
  |           v                                                       |
  |  +------------------+     +-----------------------------+         |
  |  | MECHANISM        | --> | Allocation Rule:            |         |
  |  | SPECIFICATION    |     |   x(v) = argmax sum v_i     |         |
  |  |                  |     | Payment Rule:                |         |
  |  |                  |     |   p_i(v) = externality_i    |         |
  |  +--------+---------+     +-----------------------------+         |
  |           |                                                       |
  |           v                                                       |
  |  +------------------+                                             |
  |  | PROPERTY CHECK   |   [IC] Truthful reporting dominant?        |
  |  |                  |   [IR] Agents want to participate?         |
  |  |                  |   [BB] Payments balance?                   |
  |  |                  |   [EFF] Welfare maximized?                 |
  |  +--------+---------+                                             |
  |           |                                                       |
  |     +-----+------+                                                |
  |     |            |                                                |
  |    YES           NO                                               |
  |     |            |                                                |
  |     v            v                                                |
  |  +--------+   +-----------+                                       |
  |  | DEPLOY  |   | REDESIGN |                                       |
  |  +--------+   +-----------+                                       |
  |     |            ^                                                |
  |     v            |                                                |
  |  +--------+      |                                                |
  |  | AGENTS  |     |  (loop until properties satisfied)             |
  |  | PLAY    |     |                                                |
  |  +--------+      |                                                |
  |     |            |                                                |
  |     v            |                                                |
  |  +--------+      |                                                |
  |  | OBSERVE | ----+  (failed: exploitation, low efficiency,        |
  |  | OUTCOME |          non-participation, collusion)               |
  |  +--------+                                                        |
  |                                                                   |
  +=================================================================+
```

---

### 12.6 Complete End-to-End Example

```python
"""
End-to-end example: A game-theoretic task allocation system.

Problem: Multiple agents bid on tasks. Each agent has private
costs and capabilities. We want to allocate tasks efficiently
while ensuring truthful reporting.

Solution: Use a VCG mechanism for task allocation.
"""

from typing import List, Dict, Set, Tuple
from dataclasses import dataclass
import itertools


@dataclass
class Task:
    task_id: str
    description: str
    value: float  # Value to the system of completing this task


@dataclass
class Capability:
    task_type: str
    cost: float  # Agent's cost per unit of this task type


class Agent:
    def __init__(self, agent_id: str, capabilities: List[Capability]):
        self.agent_id = agent_id
        self.capabilities = {c.task_type: c.cost for c in capabilities}

    def cost_to_complete(self, task: Task) -> float:
        """Agent's cost to complete a task (private information)."""
        # Simplified: cost depends on task type match
        return self.capabilities.get(task.description, float("inf"))


class VCGTaskAllocator:
    """
    Allocates tasks using VCG mechanism.
    Agents report their costs; the mechanism allocates to minimize
    total cost (maximize efficiency).
    """

    def __init__(self, agents: List[Agent], tasks: List[Task]):
        self.agents = agents
        self.tasks = tasks

    def allocate(
        self, reported_costs: Dict[str, Dict[str, float]]
    ) -> Tuple[Dict[str, str], Dict[str, float]]:
        """
        VCG task allocation.

        reported_costs: {agent_id: {task_id: cost}}

        Returns:
            (assignment, payments)
            assignment: {task_id: agent_id}
            payments: {task_id: payment_amount}
        """
        # Step 1: Find efficient allocation (minimize total cost)
        assignment, min_cost = self._find_efficient_allocation(
            reported_costs, list(self.tasks)
        )

        # Step 2: Compute VCG payments
        # Each agent pays the externality they impose
        payments = {}
        for task in self.tasks:
            agent = assignment.get(task.task_id)
            if agent is None:
                payments[task.task_id] = 0.0
                continue

            # Cost without this agent
            other_agents = [
                a for a in self.agents if a.agent_id != agent
            ]
            other_costs = {
                a_id: costs for a_id, costs in reported_costs.items()
                if a_id != agent
            }
            _, cost_without = self._find_efficient_allocation(
                other_costs,
                [t for t in self.tasks if t.task_id != task.task_id],
            )

            # Cost to others in current allocation
            cost_to_others = 0.0
            for other_task in self.tasks:
                if other_task.task_id == task.task_id:
                    continue
                other_agent = assignment.get(other_task.task_id)
                if other_agent:
                    cost_to_others += reported_costs[other_agent][
                        other_task.task_id
                    ]

            # Payment = externality
            payments[task.task_id] = cost_without - cost_to_others

        return assignment, payments

    def _find_efficient_allocation(
        self,
        costs: Dict[str, Dict[str, float]],
        tasks: List[Task],
    ) -> Tuple[Dict[str, str], float]:
        """
        Brute-force assignment to minimize cost.
        In practice, use Hungarian algorithm or linear programming.
        """
        agents = list(costs.keys())

        if not tasks or not agents:
            return {}, 0.0

        best_cost = float("inf")
        best_assignment = {}

        # Try assigning each task to each agent
        # (Simplified: sequential assignment)
        for perm in itertools.permutations(agents, len(tasks)):
            total_cost = 0.0
            assignment = {}
            for task, agent in zip(tasks, perm):
                cost = costs[agent].get(task.task_id, float("inf"))
                if cost == float("inf"):
                    total_cost = float("inf")
                    break
                total_cost += cost
                assignment[task.task_id] = agent

            if total_cost < best_cost:
                best_cost = total_cost
                best_assignment = assignment

        return best_assignment, best_cost


def run_task_allocation_example():
    """Demonstrate VCG task allocation with multiple agents."""
    print("=" * 70)
    print("VCG TASK ALLOCATION -- End-to-End Example")
    print("=" * 70)

    # Create agents
    agents = [
        Agent("Agent_1", [Capability("data_processing", 5),
                          Capability("model_training", 10)]),
        Agent("Agent_2", [Capability("data_processing", 8),
                          Capability("model_training", 6)]),
        Agent("Agent_3", [Capability("data_processing", 12),
                          Capability("model_training", 8)]),
    ]

    # Create tasks
    tasks = [
        Task("T1", "data_processing", 100),
        Task("T2", "model_training", 200),
    ]

    print("\nAgent Capabilities (private costs per task type):")
    for agent in agents:
        for task_type, cost in agent.capabilities.items():
            print(f"  {agent.agent_id}: {task_type} = ${cost}")

    print("\nTasks:")
    for task in tasks:
        print(f"  {task.task_id}: {task.description} (value=${task.value})")

    # Agents report their costs (truthfully, since VCG is IC)
    truthful_reports = {}
    for agent in agents:
        truthful_reports[agent.agent_id] = {}
        for task in tasks:
            truthful_reports[agent.agent_id][task.task_id] = (
                agent.cost_to_complete(task)
            )

    print("\nTruthful Cost Reports:")
    for agent_id, task_costs in truthful_reports.items():
        for task_id, cost in task_costs.items():
            print(f"  {agent_id} -> {task_id}: ${cost}")

    # Run VCG allocation
    allocator = VCGTaskAllocator(agents, tasks)
    assignment, payments = allocator.allocate(truthful_reports)

    print("\n" + "=" * 70)
    print("ALLOCATION RESULTS")
    print("=" * 70)
    print(f"\nEfficient Assignment (minimizes total cost):")
    total_cost = 0
    for task_id, agent_id in assignment.items():
        cost = truthful_reports[agent_id][task_id]
        total_cost += cost
        print(f"  {task_id} -> {agent_id} (cost=${cost})")
    print(f"\n  Total cost to system: ${total_cost:.2f}")

    print(f"\nVCG Payments (externality-based):")
    total_payment = 0
    for task_id in sorted(payments.keys()):
        print(f"  {task_id} payment: ${payments[task_id]:.2f}")
        total_payment += payments[task_id]
    print(f"  Total VCG payments: ${total_payment:.2f}")

    print("\nAgent Net Outcomes:")
    for agent in agents:
        tasks_for_agent = [
            t for t, a in assignment.items() if a == agent.agent_id
        ]
        if tasks_for_agent:
            total_pay = sum(
                truthful_reports[agent.agent_id][t] for t in tasks_for_agent
            )
            total_receive = sum(
                payments[t] for t in tasks_for_agent
            )
            print(f"  {agent.agent_id}: does {{{
                  ', '.join(tasks_for_agent)
            }}}, "
                  f"cost=${total_pay}, receives=${total_receive:.2f}, "
                  f"net=${total_receive - total_pay:.2f}")
        else:
            print(f"  {agent.agent_id}: no tasks assigned, net=$0")

    # Demonstrate truthfulness
    print("\n\n" + "=" * 70)
    print("TRUTHFULNESS CHECK: Would Agent_1 gain by lying?")
    print("=" * 70)

    # Agent_1 tries to overstate cost to get more payment
    dishonest_reports = dict(truthful_reports)
    dishonest_reports["Agent_1"]["T1"] = 20  # Lie: inflate cost

    dishonest_assignment, dishonest_payments = allocator.allocate(
        dishonest_reports
    )

    print(f"\nWith truthful report:")
    truthful_assign = assignment.get("T1", "none")
    truthful_pay = payments.get("T1", 0)
    if truthful_assign == "Agent_1":
        truthful_net = truthful_pay - truthful_reports["Agent_1"]["T1"]
    else:
        truthful_net = 0
    print(f"  Agent_1 assigned to: {truthful_assign}")
    print(f"  Agent_1 net: ${truthful_net:.2f}")

    print(f"\nWith inflated cost report ($5 -> $20 for T1):")
    dishonest_assign = dishonest_assignment.get("T1", "none")
    dishonest_pay = dishonest_payments.get("T1", 0)
    if dishonest_assign == "Agent_1":
        dishonest_net = dishonest_pay - dishonest_reports["Agent_1"]["T1"]
    else:
        dishonest_net = 0
    print(f"  Agent_1 assigned to: {dishonest_assign}")
    print(f"  Agent_1 net: ${dishonest_net:.2f}")
    print(f"\nConclusion: Agent_1 cannot gain by lying!")
    print("(In fact, lying may cause them to lose the assignment)")


if __name__ == "__main__":
    run_task_allocation_example()
```

---

## References and Further Reading

1. **Von Neumann, J. & Morgenstern, O. (1944)**. *Theory of Games and Economic Behavior*. The foundational text.

2. **Nash, J. (1950)**. "Equilibrium Points in N-Person Games." *Proceedings of the National Academy of Sciences*, 36(1), 48-49.

3. **Axelrod, R. (1984)**. *The Evolution of Cooperation*. Basic Books. The classic study of iterated Prisoner's Dilemma and the emergence of cooperation.

4. **Shapley, L.S. (1953)**. "A Value for n-Person Games." *Contributions to the Theory of Games*, 2(28), 307-317.

5. **Roughgarden, T. (2016)**. *Twenty Lectures on Algorithmic Game Theory*. Cambridge University Press. Computational aspects of game theory.

6. **Nisan, N., Roughgarden, T., Tardos, E., & Vazirani, V.V. (2007)**. *Algorithmic Game Theory*. Cambridge University Press. The reference for algorithmic and computational game theory.

7. **Shoham, Y. & Leyton-Brown, K. (2008)**. *Multiagent Systems: Algorithmic, Game-Theoretic, and Logical Foundations*. Cambridge University Press. Comprehensive textbook on multi-agent systems with game theory.

8. **Myerson, R.B. (1991)**. *Game Theory: Analysis of Conflict*. Harvard University Press. Standard graduate-level game theory textbook.

9. **Parkes, D.C. (2001)**. *Iterative Combinatorial Auctions: Achieving Economic and Computational Efficiency*. PhD thesis, University of Pennsylvania.

10. **Sandholm, T. (2002)**. "Algorithm for Optimal Winner Determination in Combinatorial Auctions." *Artificial Intelligence*, 135(1-2), 1-54.

11. **Cramton, P., Shoham, Y., & Steinberg, R. (2006)**. *Combinatorial Auctions*. MIT Press.

12. **Zinkevich, M. (2003)**. "Online Convex Programming and Generalized Infinitesimal Gradient Ascent." *ICML 2003*. Foundation of no-regret learning in games.
