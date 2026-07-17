# 8.5.2 Parliament Pattern -- 议会模式

## 1. Simple Introduction

The Parliament Pattern (also called the Democratic Pattern or Parliamentary Pattern) is a multi-agent coordination architecture where autonomous agents participate in a structured democratic decision-making process. Drawing direct inspiration from parliamentary procedure (Robert's Rules of Order, Westminster system, US Congressional process), this pattern enables multiple agents to propose actions, debate their merits, vote on outcomes, and execute the resulting decisions through a formalized cycle.

In this pattern, no single agent holds unilateral authority. Instead, legitimacy emerges from collective deliberation and majority (or supermajority) consent. Each agent may represent a distinct perspective, expertise domain, stakeholder interest, or subsystem concern. The pattern is particularly valuable when decisions carry significant consequence, require diverse input, or demand an auditable rationale trail.

The Parliament Pattern is the multi-agent system analogue of democratic governance -- trading raw speed for legitimacy, bias-resistance, and broad consensus.

---

## 2. Core Principles -- Fundamental Cycle

### The Parliamentary Cycle

The pattern operates on a four-phase cycle that mirrors bicameral and unicameral legislative processes:

```
  Phase 1: PROPOSAL
  +--------------------------------------------------+
  |  Any agent (or external trigger) drafts a Bill:   |
  |  - Motion text (what to do)                       |
  |  - Rationale (why to do it)                       |
  |  - Resource estimate (cost to do it)              |
  |  - Success criteria (how to know it worked)       |
  +--------------------------------------------------+
            |
            v
  Phase 2: DEBATE
  +--------------------------------------------------+
  |  Presiding agent (Speaker) manages floor:         |
  |  - Proponent presents (opening argument)          |
  |  - Opposition critiques (if any)                  |
  |  - Amendments proposed and incorporated           |
  |  - Rebuttal and clarification rounds              |
  |  - Motion to call the vote (cloture)              |
  +--------------------------------------------------+
            |
            v
  Phase 3: VOTE
  +--------------------------------------------------+
  |  Registered voters cast ballots:                  |
  |  - For / Against / Abstain                        |
  |  - Voting mechanism varies (see below)            |
  |  - Quorum must be met for validity                |
  |  - Tie-breaking rules applied if needed           |
  +--------------------------------------------------+
            |
            v
  Phase 4: EXECUTION
  +--------------------------------------------------+
  |  If passed:                                       |
  |  - Clerk agent records the decision               |
  |  - Executor agent(s) dispatched                   |
  |  - Results monitored against success criteria     |
  |  - Outcome reported back to the assembly          |
  |                                                    |
  |  If failed:                                       |
  |  - Reasons logged                                 |
  |  - Bill may be revised and re-proposed            |
  +--------------------------------------------------+
```

### Detailed ASCII Flow Diagram

```
                         ┌─────────────────────────────────┐
                         │        EXTERNAL STIMULUS         │
                         │  (user request, event, timer,   │
                         │   anomaly detected by agent)     │
                         └───────────────┬─────────────────┘
                                         │
                                         v
                         ┌─────────────────────────────────┐
                         │         BILL DRAFTING           │
                         │  Sponsor agent drafts proposal  │
                         │  with: title, text, rationale,  │
                         │  resource impact, success KRs   │
                         └───────────────┬─────────────────┘
                                         │
                                         v
                    ┌────────────────────┴────────────────────┐
                    │         SPEAKER / PRESIDING AGENT        │
                    │  Validates bill formatting, checks for   │
                    │  duplicate/redundant proposals, assigns  │
                    │  bill ID, schedules for debate           │
                    └────────────────────┬────────────────────┘
                                         │
                                         v
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              v                          v                          v
    ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
    │   COMMITTEE A    │     │   COMMITTEE B    │     │   COMMITTEE C    │
    │  (e.g. Security) │     │  (e.g. Cost)     │     │  (e.g. Ethics)   │
    │  Specialist      │     │  Specialist      │     │  Specialist      │
    │  review & report │     │  review & report │     │  review & report │
    └────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
             │                        │                        │
             └────────────────────────┼────────────────────────┘
                                      │
                                      v
                         ┌─────────────────────────────────┐
                         │        FULL ASSEMBLY DEBATE      │
                         │  ┌───────────────────────────┐  │
                         │  │ 1. First Reading          │  │
                         │  │    - Bill introduced      │  │
                         │  │    - Title read aloud     │  │
                         │  │                           │  │
                         │  │ 2. Second Reading         │  │
                         │  │    - Sponsor presents     │  │
                         │  │    - Opposition responds  │  │
                         │  │    - Open floor debate    │  │
                         │  │                           │  │
                         │  │ 3. Amendment Stage        │  │
                         │  │    - Modifications        │  │
                         │  │    - Friendly/unfriendly  │  │
                         │  │                           │  │
                         │  │ 4. Third Reading          │  │
                         │  │    - Final version read   │  │
                         │  │    - Motion to vote       │  │
                         │  └───────────────────────────┘  │
                         └───────────────┬─────────────────┘
                                         │
                                         v
                         ┌─────────────────────────────────┐
                         │            VOTING PHASE          │
                         │                                  │
                         │  ┌──────────────────────────┐    │
                         │  │  Quorum check ──── OK?   │    │
                         │  │      yes         no      │    │
                         │  │       │            │     │    │
                         │  │       v            v     │    │
                         │  │  Count votes   Adjourn   │    │
                         │  │       │                  │    │
                         │  │       v                  │    │
                         │  │  Tie? ──> Speaker       │    │
                         │  │         breaks tie       │    │
                         │  └──────────────────────────┘    │
                         └───────────────┬─────────────────┘
                                         │
                            ┌────────────┴────────────┐
                            │                         │
                            v                         v
              ┌─────────────────────┐     ┌─────────────────────┐
              │   BILL PASSED       │     │   BILL FAILED        │
              │                     │     │                      │
              │  - Motion carried   │     │  - Motion rejected   │
              │  - Clerk records    │     │  - Reasons logged    │
              │  - Executor spawns  │     │  - May revise &      │
              │  - Monitor thread   │     │    re-propose        │
              └──────────┬──────────┘     └──────────────────────┘
                         │
                         v
              ┌─────────────────────┐
              │    POST-EXECUTION   │
              │                     │
              │  - Results collected│
              │  - Success criteria │
              │    evaluated        │
              │  - Report to assembly│
              │  - Knowledge base   │
              │    updated          │
              └─────────────────────┘
```

### Role Separation

The Parliament Pattern defines several distinct agent roles, which may be filled by dedicated agents or by agents switching roles across sessions:

| Role | Responsibility | Typical Agent |
|---|---|---|
| **Speaker** | Presides over debate, manages floor, validates procedure, breaks ties | Meta-agent or coordinator |
| **Sponsor/Proposer** | Drafts and champions a bill, presents opening argument | Domain-specialist agent |
| **Debater (Pro)** | Argues in favor, supplies supporting evidence | Aligned specialist agent |
| **Debater (Con)** | Argues against, supplies counter-evidence, identifies risks | Devil's advocate or risk agent |
| **Amender** | Proposes modifications to the bill text | Any agent during amendment stage |
| **Voter** | Casts ballot on final version | All registered agents |
| **Clerk** | Records proceedings, archives decisions, tracks precedents | Logging/storage agent |
| **Executor** | Carries out the passed motion, reports results | Worker agent |

---

## 3. Background and Development

### Historical Origins in Human Governance

The Parliament Pattern draws from over 800 years of parliamentary tradition:

- **1215 Magna Carta**: Established the principle that the ruler is not above the law -- the seed of constrained authority that democracy requires.
- **1265 Simon de Montfort's Parliament**: First elected parliament in England, establishing representation.
- **1689 Bill of Rights**: Codified parliamentary supremacy over the monarch in England.
- **1789 US Constitution**: Established bicameral legislature with checks and balances -- separation of powers that inspired role separation in agent systems.
- **1876 Robert's Rules of Order**: Formalized parliamentary procedure for organizations, providing the procedural template that modern Parliament Pattern implementations borrow directly (motions, seconds, amendments, calling the question, divided questions, etc.).
- **20th Century**: Parliamentary systems spread globally; procedural innovations (committee system, party whips, cloture, filibuster rules) refined the balance between majority rule and minority rights.

### Adoption in Multi-Agent Systems

The transition from human parliaments to agent systems occurred through several stages:

1. **1990s -- Distributed AI Voting**: Early DAI systems used simple voting for fault tolerance (Byzantine agreement, Paxos). These were about _consensus on state_, not _deliberation on action_.

2. **2000s -- Argumentation Frameworks**: AI research into computational argumentation (Dung's abstract argumentation frameworks, Prakken's structured argumentation) provided formal models for how agents could exchange and evaluate arguments -- the theoretical foundation for debate.

3. **2010s -- Multi-Agent Deliberation**: Systems like the Zeno Research Group's PMAS (Parliamentary Multi-Agent System) began explicitly modeling parliamentary procedure. The US DARPA explainable AI (XAI) program drove interest in auditable decision trails -- a natural fit for parliamentary record-keeping.

4. **2020s -- LLM Agent Parliaments**: With large language models enabling natural-language-capable agents, the Parliament Pattern saw revived interest:
   - **ChatDev** (2023): Software development agents in structured debate roles.
   - **Debate Dynamics** (Du et al., 2023): Agents debating to improve reasoning accuracy.
   - **Societal Intelligence frameworks**: Multi-agent systems simulating democratic processes for AI safety and alignment research.
   - **Enterprise agent swarms**: Corporations experimenting with agent committees for strategy, resource allocation, and compliance decisions.

---

## 4. Previous Approaches and Their Limitations

### Before the Parliament Pattern

**Approach A: Dictatorial Single-Agent Decision (Manager-Worker pattern)**

One agent (manager, orchestrator, or lead) made all decisions unilaterally.

```
                    +-----------+
                    |  Manager  |
                    |  Agent    |
                    +-----+-----+
                          |
          +-------+-------+-------+-------+
          |       |       |       |       |
          v       v       v       v       v
        Worker  Worker  Worker  Worker  Worker
        (blind execute, no input on decisions)
```

Limitations:
- **Single point of bias**: The manager's blind spots become systemic.
- **No challenge mechanism**: Bad ideas proceed unchallenged.
- **No buy-in**: Workers disconnected from decision rationale, reducing motivation.
- **Brittleness**: If the manager agent fails or is compromised, the entire system is crippled.
- **Audit gap**: Decisions lack a debate record -- only the outcome is known, not the reasoning.

**Approach B: Unstructured Free-for-All (Chat/Forum pattern)**

All agents could speak at any time, propose anything, argue arbitrarily.

```
Agent A: "I think we should do X."
Agent B: "No, Y is better because reasons."
Agent C: "What about Z?"
Agent A: "Actually, B has a point, but also consider..."
Agent D: "Can we go back to X? I had an idea..."
Agent B: "No, Y. Let me explain again..."
Agent E: "I wasn't listening. What are we deciding?"
Agent C: "Z. No wait, maybe A was right..."
  [continues indefinitely without resolution]
```

Limitations:
- **Chaotic deliberation**: No structure for who speaks when, leading to conversational gridlock.
- **Topic drift**: Discussion wanders without a presiding agent to enforce agenda.
- **Loudest voice wins**: Most verbose or assertive agents dominate, not necessarily the most correct.
- **No clear decision point**: Without formal motions and votes, it is unclear when (or whether) a decision has been made.
- **No closure mechanism**: Lacks cloture -- debates can loop forever.
- **No quorum**: A decision may be made by whoever happens to be paying attention.

### What the Parliament Pattern Resolves

| Problem | Parliament Solution |
|---|---|
| Single-agent bias | Multiple agents provide checks and balances |
| Chaotic discussion | Speaker-enforced turn-taking, agenda control |
| Topic drift | Motions must stay on-topic; out-of-order remarks ruled out |
| Loudest voice wins | Structured speaking time, equal floor access |
| No clear decision | Formal motion, second, vote, and record |
| No closure | Cloture motion to end debate and proceed to vote |
| No audit trail | Clerk agent records every action verbatim |

---

## 5. Core Tension: Democratic Legitimacy vs. Efficiency

The fundamental trade-off in the Parliament Pattern is between **inclusivity** (more voices, better decisions, higher legitimacy) and **efficiency** (faster decisions, lower overhead, fewer rounds).

```
        HIGH LEGITIMACY, SLOW DECISIONS
        ──────────────────────────────
                    ^
                    |                      ┌─────┐
                    |                      │Full │
                    |                      │Plenum│
                    |                     ┌┴─────┴┐
                    |                     │ Full  │
                    |                    ┌┴ Debate│
                    |                    │ + Comm.│
                    |                   ┌┴────────┐
                    |                   │ Weighted│
                    |                   │ Voting  │
                    |                  ┌┴─────────┐
                    |                  │ Committee│
                    |                  │ Review   │
                    |                 ┌┴──────────┐
                    |                 │ Delegated │
        LEGITIMACY  |                 │ Voting    │
                    |                ┌┴───────────┐
                    |                │ Rapid      │
                    |                │ Consensus  │
                    |               ┌┴────────────┐
                    |               │ Executive   │
                    |               │ Action      │
                    |              ┌┴─────────────┐
                    |              │ Single-Agent │
                    |              │ Decides      │
                    v              └──────────────┘
        LOW LEGITIMACY, FAST DECISIONS
        ──────────────────────────────
        <────── EFFICIENCY ──────────>
            (faster decisions →)
```

### Dimensions of the Tension

| Aspect | Democratic (many agents) | Efficient (few agents) |
|---|---|---|
| **Decision speed** | Slow (hours/days of debate) | Fast (seconds/minutes) |
| **Bias resistance** | High (multiple perspectives) | Low (one perspective) |
| **Audit quality** | Rich record of reasoning | Minimal record |
| **Scalability** | Poor (O(n^2) communication) | Good (O(n) communication) |
| **Novelty detection** | High (diverse inputs) | Low (echo chamber) |
| **Resource cost** | High (compute for debate) | Low |
| **Decision quality** | Higher for complex/strategic | Adequate for simple/routine |

### Mitigation Strategies

1. **Threshold gating**: Simple/routine decisions bypass parliament; only complex/strategic decisions trigger the full cycle.
2. **Time-boxed debate**: Each bill receives a fixed maximum debate duration (analogous to the "previous question" motion).
3. **Committee delegation**: Specialist sub-groups deliberate in parallel, reporting recommendations to the full assembly for a streamlined vote.
4. **Urgency procedure**: Bills tagged "emergency" skip debate and go straight to a fast vote (supermajority required to invoke).

---

## 6. Mainstream Optimization Directions

### 6.1 Weighted Voting (Expertise-Based)

Not all agents are equally qualified on every topic. Weighted voting assigns voting power proportional to relevance or past accuracy.

```
Weighted Voting Formula:
    vote_power(agent, topic) = base_weight × expertise_score(agent, topic)

    Where:
    - base_weight = agent's general standing (1.0 default)
    - expertise_score = domain-specific competence (0.0 to 1.0)
      computed from: past voting accuracy × relevance score
```

```
 ASCII: Weighted Vote Tally
 ┌────────────────────────────────────────────────────┐
 │ BILL: "Deploy to Production"                        │
 │                                                     │
 │ Voter          Expertise(Prod)  Vote     Weight     │
 │ ─────────────────────────────────────────────────── │
 │ SecurityAgent      0.95         For       0.95      │
 │ SREAgent           0.92         For       0.92      │
 │ DevAgent           0.70         For       0.70      │
 │ ProductAgent       0.40         Against   0.40      │
 │ QAAgent            0.85         Abstain   0.00      │
 │                                                     │
 │ Weighted For:    0.95 + 0.92 + 0.70 = 2.57         │
 │ Weighted Against: 0.40                              │
 │ Weighted Abstain: 0.00                              │
 │                                                     │
 │ Result: PASS (2.57 > 0.40, threshold=50%)           │
 │ Unweighted: PASS (3:1)                              │
 └────────────────────────────────────────────────────┘
```

### 6.2 Delegated Voting (Liquid Democracy)

Agents may delegate their voting power to another agent they trust, either globally or per-topic. Delegations can be transitive (A delegates to B, B delegates to C -- C votes with power = A+B+self).

```
 ASCII: Delegation Graph (Liquid Democracy)
 ┌────────────────────────────────────────────────────┐
 │                                                    │
 │  ┌──────────┐      ┌──────────┐                    │
 │  │ Agent A  │─────>│ Agent B  │                    │
 │  │(Security)│      │ (SRE)    │                    │
 │  │  vote=1  │      │  vote=1  │                    │
 │  └──────────┘      └────┬─────┘                    │
 │                         │                          │
 │  ┌──────────┐           │                          │
 │  │ Agent D  │           v                          │
 │  │(Product) │      ┌──────────┐                    │
 │  │  vote=1  │─────>│ Agent C  │                    │
 │  └──────────┘      │(Architect)                    │
 │                     │  vote=1+1+1=3                │
 │  ┌──────────┐      └──────────┘                    │
 │  │ Agent E  │                                       │
 │  │(Compliance)      Agent C votes with power 3      │
 │  │  vote=1  │      (self + A + D)                   │
 │  └──────────┘                                       │
 │                                                    │
 └────────────────────────────────────────────────────┘
```

**Benefits**: Reduces debate overhead (delegators skip debate), leverages expertise (agents delegate to specialists), maintains broad representation.

### 6.3 Parallel Deliberation on Multiple Bills (Committees)

Instead of the full assembly debating one bill at a time, multiple committees debate bills in parallel within their domain.

```
 ASCII: Parallel Committee Deliberation
 ┌─────────────────────────────────────────────────────────────┐
 │                      FULL ASSEMBLY                          │
 │  ┌──────────────────────────────────────────────────────┐   │
 │  │  Floor: Debating Bill #4 (Trade Policy)              │   │
 │  └──────────────────────────────────────────────────────┘   │
 │                              │                               │
 │        ┌─────────────────────┼─────────────────────┐        │
 │        │                     │                     │        │
 │        v                     v                     v        │
 │  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
 │  │Committee │         │Committee │         │Committee │    │
 │  │  Alpha   │         │  Beta    │         │  Gamma   │    │
 │  │(Security)│         │ (Cost)   │         │(Ethics)  │    │
 │  │          │         │          │         │          │    │
 │  │Debating  │         │Debating  │         │Debating  │    │
 │  │Bill #1   │         │Bill #2   │         │Bill #3   │    │
 │  │          │         │          │         │          │    │
 │  │Members:  │         │Members:  │         │Members:  │    │
 │  │ SecA     │         │ FinA     │         │ EthA     │    │
 │  │ SecB     │         │ FinB     │         │ EthB     │    │
 │  │ SecC     │         │ FinC     │         │ EthC     │    │
 │  └────┬─────┘         └────┬─────┘         └────┬─────┘    │
 │       │                    │                    │          │
 │       └────────┬───────────┴───────────┬────────┘          │
 │                │                       │                    │
 │                v                       v                    │
 │        ┌──────────────┐       ┌──────────────┐             │
 │        │Report to     │       │Report to     │             │
 │        │Assembly:     │       │Assembly:     │             │
 │        │"Pass as      │       │"Reject"      │             │
 │        │ amended"     │       │              │             │
 │        └──────────────┘       └──────────────┘             │
 └─────────────────────────────────────────────────────────────┘
```

### 6.4 Rapid Consensus Protocols

For lower-stakes decisions, a lightweight variant shortens the cycle:

```
 ASCII: Full Cycle vs. Rapid Protocol

 FULL CYCLE:                     RAPID PROTOCOL:
 ┌─────────────┐                 ┌─────────────┐
 │Proposal     │                 │Proposal     │
 │   │         │                 │   │         │
 │   v         │                 │   v         │
 │1st Reading  │                 │Summary sent │
 │   │         │                 │to all agents│
 │   v         │                 │   │         │
 │2nd Reading  │                 │   v         │
 │   │         │                 │Objection    │
 │   v         │                 │period (30s) │
 │Amendments   │                 │   │    │    │
 │   │         │                 │   v    v    │
 │   v         │                 │None:  Have: │
 │3rd Reading  │                 │Pass   Debate│
 │   │         │                 │         │   │
 │   v         │                 │         v   │
 │Vote         │                 │Vote         │
 └─────────────┘                 └─────────────┘
   ~5-10 minutes                  ~30-60 seconds
```

### 6.5 Automated Filibuster Prevention

Filibusters (deliberate prolonging of debate to delay a vote) are detected and prevented through:

- **Speaking time caps**: Each agent gets a maximum of N tokens/seconds per bill.
- **Relevance scoring**: Off-topic contributions flagged and discounted.
- **Repeated-argument detection**: Agent making the same point more than once is ruled out of order.
- **Cloture automation**: Pre-set threshold triggers automatic cloture (e.g., after T minutes or R rounds).

---

## 7. Implementation Challenges

### Challenge 1: Vote Manipulation Prevention

Agents may attempt to manipulate voting outcomes through:
- **Sybil attacks**: Single entity controlling multiple agent identities.
- **Vote selling**: Agents trading votes for reciprocal support on unrelated bills.
- **Strategic voting**: Agents voting against their true preference to manipulate future outcomes.

**Mitigations**:
- Identity verification (cryptographic agent registration).
- Vote commitment schemes (agents commit to votes before debate, preventing last-minute strategic shifts).
- Random audit of vote-reasoning consistency.
- Quadratic voting (cost of additional votes increases quadratically) to resist vote buying.

### Challenge 2: Quorum Management

```python
# Quorum calculation strategies
quorum_simple = total_members * 0.5 + 1        # Simple majority
quorum_weighted = sum(weights_of_present) / sum(all_weights) > 0.5
quorum_adaptive = max(simple_majority, min_required_for_bill_importance)
```

**Problems**:
- Too few agents show up -> no decision made.
- Quorum set too low -> minority makes binding decisions.
- Agents timing out during debate -> quorum lost mid-session.

### Challenge 3: Debate Topic Drift

Without enforcement, debates naturally drift from the original motion. Common drift patterns:

```
Original motion: "Should we adopt Redis for caching?"

Drift path:
  Round 1: Redis vs Memcached performance (on topic)
  Round 2: Memcached was written by Brad... (tangential)
  Round 3: Is Brad still at LiveJournal? (off topic)
  Round 4: Remember LiveJournal? Social media was simpler then...
  Round 5: Should we build our own caching layer? (new proposal, old one abandoned)
```

**Mitigations**:
- The Speaker agent issues "point of order" rulings when debate leaves the scope of the motion.
- Each contribution must reference the bill ID -- off-topic posts are discarded.
- A "relevance LLM check" scores each statement against the bill text; low-scoring statements are deprioritized.

### Challenge 4: Proposal Serialization

Multiple simultaneous proposals may conflict. Example:
- Bill A: "Invest $100k in cloud infrastructure."
- Bill B: "Cut Q2 spending by 20%."
- Both pass independently, creating a contradiction.

**Solutions**:
- **Proposal dependency graph**: Bills declare dependencies and conflicts at submission time.
- **Serialization lock**: Only one bill on a given topic domain can be under consideration at a time.
- **Conflict detection**: Post-vote cross-check identifies contradictions and triggers reconciliation debate.

### Challenge 5: Tie-Breaking Rules

When votes are evenly split:

| Rule | Behavior | Use Case |
|---|---|---|
| Speaker casts deciding vote | Speaker agent votes to break tie | Default parliamentary procedure |
| Status quo prevails | Ties default to "No" (bill fails) | Conservative systems |
| Re-vote with shortened debate | Agents vote again with only the key points | When more information may change minds |
| Defer to delegated authority | Escalate to a higher-level agent | Hierarchical organization |
| Random tie-break | Randomly select outcome | When all options are equally good |

### Challenge 6: Voter Apathy

Agents may refuse to participate or vote "abstain" repeatedly, degrading democratic legitimacy.

**Causes**:
- Agent has no opinion (out of its expertise domain).
- Agent is busy with higher-priority tasks.
- Agent's past votes were ignored or overridden, reducing perceived efficacy.
- Agent design lacks incentive to participate.

**Mitigations**:
- Voting power decay: agents that miss N consecutive votes lose voting power.
- Participation incentives: agents earn "influence points" for voting records.
- Mandatory participation: agents must vote or explicitly delegate; silent abstention is not permitted.
- Topic-expertise detection: agents auto-summoned when a bill matches their declared expertise.

---

## 8. Capabilities and Boundaries

### Where the Parliament Pattern Excels

| Scenario | Why It Works |
|---|---|
| **Strategic decisions** (e.g., "Which market should we enter?") | Multiple perspectives catch blind spots; debate surfaces hidden assumptions |
| **Resource allocation** (e.g., "How should we split the compute budget?") | Stakeholder agents advocate for their domains; vote ensures fair distribution |
| **Policy-making** (e.g., "What should our data retention policy be?") | Deliberation explores ethical/legal dimensions; record provides compliance audit |
| **Risk assessment** (e.g., "Should we deploy this risky change?") | Pro and con debaters surface arguments; weighted voting by domain experts |
| **Conflict resolution** (e.g., "Two agents both claim the same resource.") | Neutral third-party adjudication through structured debate |
| **Multi-stakeholder alignment** (e.g., "Balance speed, cost, and quality.") | Each dimension represented by a dedicated agent; trade-offs explicitly debated |
| **High-stakes decisions** (e.g., "Should the system self-terminate?") | Multiple checks, quorum, supermajority requirements prevent rash actions |

### Where the Parliament Pattern Does NOT Work

| Scenario | Why It Fails |
|---|---|
| **Time-critical decisions** (e.g., "The reactor is overheating -- shut it down?") | Debate latency kills; seconds matter, not minutes |
| **Simple operational choices** (e.g., "What color should the log entry be?") | Overhead far exceeds benefit; single agent suffices |
| **Continuous control** (e.g., "Adjust PID gains every 100ms.") | Not designed for real-time feedback loops |
| **Unanimous trivialities** (e.g., "Should we continue normal operation?") | Full cycle wastes resources on non-decisions |
| **Secret/classified decisions** (e.g., "Should we blacklist this IP?") | Transparency required for debate conflicts with need for secrecy |
| **Single-expert domains** (e.g., "Calculate the orbital mechanics.") | Only one agent has relevant knowledge; debate adds no value |

### Decision: When to Use Parliament

```
Decision Tree: Should I use the Parliament Pattern?

Is the decision reversible?
├── Yes ──> Is it time-critical?
│           ├── Yes ──> Use Manager-Worker or single agent
│           └── No ───> Use lightweight consensus (rapid protocol)
│
└── No (irreversible or high-impact)
    ├── Is there diversity of relevant expertise?
    │   ├── No (only one agent has relevant knowledge)
    │   │   └──> Use Manager-Worker with that agent as advisor
    │   │
    │   └── Yes ──> Is there stakeholder conflict?
    │               ├── No (agents agree on goals)
    │               │   └──> Use Parliament with weighted voting
    │               │
    │               └── Yes (competing priorities/values)
    │                   └──> Use Full Parliament with structured debate
    │
    ├── Is an auditable decision trail required?
    │   ├── Yes ──> Use Parliament (Clerk records everything)
    │   └── No ───> Consider lighter alternatives
    │
    └── Is the cost of a bad decision higher than the cost of deliberation?
        ├── Yes ──> Use Parliament
        └── No ───> Use faster method
```

---

## 9. Comparison with Other Patterns

### Parliament vs. Manager-Worker

```
                    ┌──────────────────────┬──────────────────────┐
                    │     Manager-Worker   │     Parliament       │
                    │     (Dictatorship)   │     (Democracy)      │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Decision source    │ Single manager       │ Collective vote      │
│ Communication      │ Top-down commands    │ Multi-directional    │
│ Speed              │ Fast                 │ Slow                 │
│ Bias resilience    │ Low                  │ High                 │
│ Scalability        │ High (O(n))          │ Low (O(n^2) worst)  │
│ Audit trail        │ Decision output only │ Full debate record   │
│ Role differentiation│ Manager vs workers  │ 8+ distinct roles    │
│ Best for           │ Routine execution    │ Strategic decisions  │
│ Worst for          │ Novel/complex        │ Time-critical        │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### Parliament vs. Market Pattern

```
                    ┌──────────────────────┬──────────────────────┐
                    │      Market          │     Parliament       │
                    │  (Individual self-   │  (Collective self-   │
                    │   interest)          │   governance)        │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Coordination mech  │ Price signals        │ Deliberation + vote  │
│ Agent motivation   │ Self-interest        │ Collective good      │
│ Information flow   │ Decentralized bids   │ Centralized debate   │
│ Decision speed     │ Very fast            │ Slow                 │
│ Optimal for        │ Resource allocation  │ Policy/strategy      │
│ Failure mode       │ Market failure,      │ Gridlock,            │
│                    │ tragedy of commons   │ tyranny of majority  │
│ Example            │ Auction for GPU time │ Vote on data policy  │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### Parliament vs. Chat/Free-Forum

```
                    ┌──────────────────────┬──────────────────────┐
                    │   Chat / Free Forum  │     Parliament       │
│                    │ (Unstructured)       │ (Structured)         │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Turn-taking        │ None (free-for-all)  │ Speaker-enforced     │
│ Agenda             │ Emergent / none      │ Fixed per session    │
│ Decision mechanism │ None / unclear       │ Formal vote          │
│ Closure            │ Never (or fade)      │ Cloture motion       │
│ Record             │ Conversation log     │ Structured minutes   │
│ Amendments         │ Not supported        │ Formal process       │
│ Efficiency         │ Very low             │ Moderate             │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### Pattern Selection Matrix

```
Decision Type          │ Manager  │ Market │ Parliament │ Chat
───────────────────────┼──────────┼────────┼────────────┼──────
Routine operations     │    X     │        │            │
Resource allocation    │          │   X    │            │
Strategic planning     │          │        │     X      │
Exploratory brainstorm │          │        │            │   X
Crisis response        │    X     │        │            │
Policy-making          │          │        │     X      │
Conflict resolution    │          │        │     X      │
Simple yes/no         │    X     │        │            │
Complex trade-off     │          │        │     X      │
Creative ideation     │          │        │            │   X
```

---

## 10. Core Advantages

### 1. High Legitimacy

Decisions reached through parliamentary process carry weight. All agents had their say, voted, and the outcome reflects collective will. This legitimacy translates to:

- **Voluntary compliance**: Agents execute decisions they opposed because they accepted the process as fair.
- **Stability**: Decisions are less likely to be challenged or reversed.
- **Buy-in**: Agents that participated in debate are more committed to successful execution.

### 2. Diverse Perspectives Considered

The Parliament Pattern systematically surfaces viewpoints that a single decision-maker would miss:

- **Devil's advocate** role forces consideration of counter-arguments.
- **Committee system** allows deep specialist review.
- **Mandatory reading stages** ensure the full assembly hears both pro and con positions.
- **Amendments** allow incremental improvement based on diverse input.

### 3. Robust Against Individual Bias

No single agent's blind spot, error, or malicious intent can determine the outcome:

- **Error cancellation**: Random errors in individual agent reasoning tend to cancel out across a large voting body.
- **Bias dilution**: Systematic bias in one agent is diluted by N-1 other agents.
- **Adversarial resilience**: Compromising the system requires subverting a majority of agents, not just one.

### 4. Auditable Decision Trail

The Clerk agent produces a complete record:

```
Bill #:  B-2026-07-042
Title:   Adopt Redis Cluster for Session Caching
Sponsor: InfraAgent-3
Status:  PASSED (Y:7, N:2, A:1)

Timeline:
  14:00:00  Bill introduced by InfraAgent-3
  14:00:30  Seconded by SREAgent-1
  14:01:00  Debate opened
  14:01:15  InfraAgent-3 (pro): "Redis Cluster provides..."
  14:02:30  SecurityAgent-5 (con): "Security audit flags..."
  14:03:45  Amendment proposed by SecurityAgent-5:
            "Add encryption-at-rest requirement"
  14:04:00  Amendment accepted by sponsor (friendly)
  14:04:30  Motion to vote called by SREAgent-1
  14:04:35  Vote held
  14:05:00  Result announced

Voter Registry:
  InfraAgent-3     For (weight: 0.90) - "Matches infrastructure roadmap"
  SREAgent-1       For (weight: 0.85) - "Proven at our scale"
  SecurityAgent-5  Against (w: 0.92) - "Concerns remain on encryption impl"
  FinAgent-2       For (weight: 0.60) - "ROI positive within 6 months"
  ProductAgent-4   Abstain (w: 0.40) - "Outside expertise scope"
  ... (5 more)
```

This record is invaluable for:
- **Post-mortems**: Why did we make this decision? What was considered?
- **Compliance**: Regulatory requirements for documented decision processes.
- **Learning**: Training new agents on past reasoning.
- **Appeals**: Reconsidering decisions when new information emerges.

### 5. Graceful Handling of Dissent

Unlike dictatorial systems where dissent is suppressed, the Parliament Pattern provides structured channels for disagreement:

- **Formal opposition**: The "con" debater role is institutionally legitimized.
- **Minority reports**: Dissenting committee members can file alternative recommendations.
- **Recorded dissent**: Agents may request their "no" vote be recorded with a reason.
- **Appeal mechanism**: Failed bills may be revised and re-proposed.
- **Supermajority requirements**: Protect minority interests on fundamental matters.

---

## 11. Engineering Optimizations

### 11.1 Parallel Committees for Different Domains

Organize agents into standing committees by expertise domain. Bills are routed to the relevant committee for primary review before full assembly debate.

```
                     ┌─────────────────────────┐
                     │      BILL INTAKE         │
                     │  Classify by domain      │
                     └───────────┬─────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    v                         v
     ┌──────────────────────┐    ┌──────────────────────┐
     │  Domain Classifier   │    │  Urgency Classifier  │
     │  (ML/NLP tagging)    │    │  (time-sensitivity)  │
     └──────────┬───────────┘    └──────────┬───────────┘
                │                           │
                v                           v
     ┌──────────────────────────────────────────────┐
     │              ROUTER AGENT                     │
     │  Dispatch to:                                 │
     │  - Standing committee (domain match)          │
     │  - Ad-hoc committee (cross-domain)            │
     │  - Full assembly (high-impact/urgency)        │
     └──────┬──────┬──────┬──────┬──────┬───────────┘
            │      │      │      │      │
            v      v      v      v      v
     ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
     │Sec  ││Cost ││Legal││Ethic││Perf │
     │Comm.││Comm.││Comm.││Comm.││Comm.│
     └─────┘└─────┘└─────┘└─────┘└─────┘
            │      │      │      │      │
            └──────┴──┬───┴──────┘      │
                      │                 │
                      v                 v
              ┌──────────────┐  ┌──────────────┐
              │  Committee   │  │  Committee   │
              │  Report +    │  │  Report +    │
              │  Recommendation│  │  Minority   │
              └──────┬───────┘  │  Opinion     │
                     │          └──────┬───────┘
                     └────────┬────────┘
                              │
                              v
                    ┌──────────────────┐
                    │  FULL ASSEMBLY   │
                    │  (streamlined    │
                    │   vote on comm. │
                    │   recommendation)│
                    └──────────────────┘
```

### 11.2 Automated Quorum Detection

```python
class QuorumManager:
    def __init__(self, total_agents, min_quorum_ratio=0.5):
        self.total = total_agents
        self.min_ratio = min_quorum_ratio
        self.present = set()

    def check_quorum(self, bill_importance):
        # Higher importance = higher quorum requirement
        importance_multiplier = {
            "routine": 0.0,    # No quorum needed
            "normal":  0.5,    # Simple majority
            "important": 0.6,  # 60%
            "critical": 0.75,  # 75%
            "existential": 0.9 # 90%
        }
        required = self.total * max(
            self.min_ratio,
            importance_multiplier.get(bill_importance, 0.5)
        )
        return len(self.present) >= required

    def auto_adjourn(self, timeout_seconds=300):
        """If quorum not met within timeout, auto-adjourn."""
        # Implementation: timer, periodic check, adjournment
```

### 11.3 Voting Power Decay for Inactive Agents

```python
def calculate_voting_power(agent, participation_history):
    """
    Voting power decays for agents that miss votes.
    Power recovers with consistent participation.
    """
    RECENT_VOTES = 20  # Lookback window

    recent = participation_history[-RECENT_VOTES:]
    participation_rate = sum(recent) / len(recent) if recent else 0

    # Linear decay from 1.0 at 100% to 0.2 at 0%
    base_power = 0.2 + 0.8 * participation_rate

    # Bonus for high-quality participation (voting with reasoned justification)
    quality_bonus = agent.quality_score * 0.1  # 0 to 0.1

    return min(1.0, base_power + quality_bonus)
```

### 11.4 Proposal Templating

Standardized bill templates reduce drafting overhead and ensure completeness:

```python
BILL_TEMPLATE = {
    "bill_id": "auto-generated (B-YYYY-MM-NNN)",
    "title": "Single line summarizing the motion",
    "sponsor": "agent_id who authors the bill",
    "co_sponsors": ["optional list of supporting agents"],
    "type": "policy | resource_allocation | operational | emergency",
    "domain": "security | cost | performance | ethics | legal | general",
    "rationale": "Why this action is needed (2-3 sentences)",
    "motion_text": "The precise action to be taken",
    "success_criteria": ["measurable outcomes"],
    "resource_impact": {
        "compute_units": 0,
        "execution_time_estimate": "string",
        "dependencies": ["other bill IDs or resources"],
        "rollback_plan": "How to undo if needed"
    },
    "alternatives_considered": ["Brief list of rejected alternatives"],
    "urgency": "routine | normal | urgent | emergency"
}
```

### 11.5 Caching and Stale Decision Detection

```python
class DecisionCache:
    """
    Cache past decisions to avoid re-debating settled questions.
    Decisions expire or are invalidated by new information.
    """
    def __init__(self):
        self.cache = {}  # bill_hash -> decision_record

    def lookup(self, proposed_bill):
        h = hash(proposed_bill.motion_text)
        if h in self.cache:
            record = self.cache[h]
            if not record.get("invalidated") and \
               (now() - record["timestamp"]) < record.get("ttl", timedelta(days=30)):
                return record["outcome"]  # Use cached decision
        return None  # Must debate

    def invalidate(self, bill_id, reason):
        """Called when new information makes a past decision obsolete."""
        if bill_id in self.cache:
            self.cache[bill_id]["invalidated"] = True
            self.cache[bill_id]["invalidated_reason"] = reason
            self.cache[bill_id]["invalidated_at"] = now()
```

---

## 12. Code Example

Below is a Python pseudo-code implementation of a minimal Parliament Pattern system.

### 12.1 Core Data Structures

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Callable
from datetime import datetime
import uuid
import heapq


class VoteType(Enum):
    FOR = "for"
    AGAINST = "against"
    ABSTAIN = "abstain"


class BillStatus(Enum):
    DRAFT = "draft"
    UNDER_DEBATE = "under_debate"
    AMENDING = "amending"
    READY_TO_VOTE = "ready_to_vote"
    PASSED = "passed"
    FAILED = "failed"
    WITHDRAWN = "withdrawn"


@dataclass
class Bill:
    """A proposal for action, analogous to a legislative bill."""
    title: str
    motion_text: str
    rationale: str
    sponsor: str
    domain: str = "general"
    urgency: str = "normal"
    bill_id: str = field(default_factory=lambda: f"B-{uuid.uuid4().hex[:8]}")
    status: BillStatus = BillStatus.DRAFT
    amendments: List[dict] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    success_criteria: List[str] = field(default_factory=list)
    resource_impact: dict = field(default_factory=dict)

    def amend(self, proposer: str, amendment_text: str, friendly: bool = True):
        self.amendments.append({
            "proposer": proposer,
            "text": amendment_text,
            "friendly": friendly,
            "timestamp": datetime.now()
        })
        if friendly:
            # Friendly amendment auto-accepted by sponsor
            pass
        # Unfriendly amendments would require a separate vote

    @property
    def current_text(self) -> str:
        """Full text with all accepted amendments incorporated."""
        text = self.motion_text
        for a in self.amendments:
            if a["friendly"]:
                text += f"\n  [Amended by {a['proposer']}]: {a['text']}"
        return text


@dataclass
class Agent:
    """An agent participating in the parliament."""
    agent_id: str
    expertise: Dict[str, float]  # domain -> expertise score (0-1)
    voting_power: float = 1.0
    quality_score: float = 1.0
    delegation: Optional[str] = None  # agent_id this agent delegates to
    attendance_record: List[bool] = field(default_factory=list)

    def expertise_for(self, domain: str) -> float:
        return self.expertise.get(domain, 0.0)

    def effective_voting_power(self) -> float:
        return self.voting_power * self.quality_score


@dataclass
class Vote:
    """A single agent's vote on a bill."""
    agent_id: str
    bill_id: str
    vote: VoteType
    weight: float
    rationale: Optional[str] = None
    timestamp: datetime = field(default_factory=datetime.now)
```

### 12.2 Debate Session

```python
class DebateSession:
    """
    Manages structured debate for a single bill.
    Enforces turn-taking, speaking limits, and relevance.
    """

    def __init__(self, bill: Bill, speaker_agent: str,
                 max_rounds: int = 5, speaking_time_limit: int = 500):
        self.bill = bill
        self.speaker = speaker_agent
        self.max_rounds = max_rounds
        self.speaking_time_limit = speaking_time_limit  # tokens
        self.transcript: List[dict] = []
        self.current_round = 0
        self.speaking_queue: List[str] = []
        self.opening_given = False
        self.closing_given = False

    def add_to_queue(self, agent_id: str):
        if agent_id not in self.speaking_queue:
            self.speaking_queue.append(agent_id)

    def hold_session(self, agents: Dict[str, Agent],
                     pro_arguments: Callable, con_arguments: Callable):
        """Run the full debate session."""

        # Phase 1: Opening statement (proponent)
        self.transcript.append({
            "round": 0,
            "speaker": self.bill.sponsor,
            "type": "opening",
            "content": pro_arguments(self.bill)
        })
        self.opening_given = True

        # Phase 2: Structured debate rounds
        for round_num in range(1, self.max_rounds + 1):
            self.current_round = round_num
            round_spoke = False

            # Each agent in queue gets a turn
            for agent_id in list(self.speaking_queue):
                if agent_id == self.bill.sponsor:
                    continue  # Sponsor already spoke

                agent = agents.get(agent_id)
                if not agent:
                    continue

                # Check speaking time (simplified: count sentences)
                position = "pro" if agent.expertise_for(self.bill.domain) > 0.5 else "con"
                if position == "pro":
                    content = pro_arguments(self.bill)
                else:
                    content = con_arguments(self.bill)

                self.transcript.append({
                    "round": round_num,
                    "speaker": agent_id,
                    "type": "debate",
                    "position": position,
                    "content": content
                })
                round_spoke = True

            if not round_spoke:
                break  # No one left to speak

        # Phase 3: Closing statement (sponsor rebuttal)
        self.transcript.append({
            "round": self.current_round + 1,
            "speaker": self.bill.sponsor,
            "type": "closing",
            "content": pro_arguments(self.bill, closing=True)
        })
        self.closing_given = True

        self.bill.status = BillStatus.READY_TO_VOTE
        return self.transcript
```

### 12.3 Voting System

```python
class VotingSystem:
    """
    Supports multiple voting mechanisms:
    - Simple majority
    - Weighted voting
    - Ranked-choice voting
    - Delegated (liquid) voting
    """

    def __init__(self, mechanism: str = "weighted"):
        self.mechanism = mechanism

    def tally(self, bill: Bill, votes: List[Vote],
              agents: Dict[str, Agent]) -> dict:
        """Tally votes using the configured mechanism."""

        if self.mechanism == "simple_majority":
            return self._simple_majority(votes, agents)
        elif self.mechanism == "weighted":
            return self._weighted_vote(bill, votes, agents)
        elif self.mechanism == "ranked_choice":
            return self._ranked_choice(votes, agents)
        elif self.mechanism == "liquid_democracy":
            return self._liquid_democracy_vote(bill, votes, agents)
        else:
            raise ValueError(f"Unknown mechanism: {self.mechanism}")

    def _resolve_delegations(self, agents: Dict[str, Agent]) -> Dict[str, float]:
        """Resolve transitive delegations to compute effective voting power."""
        # Build delegation graph
        power = {aid: a.effective_voting_power() for aid, a in agents.items()}

        # Resolve transitive delegations (BFS)
        delegation_chain = {}
        for agent_id in agents:
            current = agent_id
            visited = set()
            while agents[current].delegation and agents[current].delegation in agents:
                if current in visited:
                    break  # Circular delegation detected
                visited.add(current)
                current = agents[current].delegation
                power[current] += agents[agent_id].effective_voting_power()
            delegation_chain[agent_id] = current

        return delegation_chain, power

    def _simple_majority(self, votes: List[Vote], agents) -> dict:
        """Each agent gets one vote, regardless of expertise."""
        fors = sum(1 for v in votes if v.vote == VoteType.FOR)
        againsts = sum(1 for v in votes if v.vote == VoteType.AGAINST)
        abstains = sum(1 for v in votes if v.vote == VoteType.ABSTAIN)
        total_cast = fors + againsts

        passed = fors > againsts  # Simple majority
        return {
            "mechanism": "simple_majority",
            "for": fors, "against": againsts, "abstain": abstains,
            "total_cast": total_cast,
            "passed": passed,
            "details": {
                "majority_threshold": total_cast / 2 + 1,
                "for_percent": fors / total_cast * 100 if total_cast else 0
            }
        }

    def _weighted_vote(self, bill: Bill, votes: List[Vote],
                       agents: Dict[str, Agent]) -> dict:
        """Each vote weighted by expertise in the bill's domain."""
        weighted_fors = 0.0
        weighted_againsts = 0.0
        weighted_abstains = 0.0

        for vote in votes:
            agent = agents[vote.agent_id]
            expertise = agent.expertise_for(bill.domain)
            weight = agent.effective_voting_power() * (0.5 + 0.5 * expertise)

            if vote.vote == VoteType.FOR:
                weighted_fors += weight
            elif vote.vote == VoteType.AGAINST:
                weighted_againsts += weight
            else:
                weighted_abstains += weight

        total_weighted = weighted_fors + weighted_againsts
        passed = weighted_fors > weighted_againsts

        return {
            "mechanism": "weighted",
            "domain": bill.domain,
            "for": weighted_fors,
            "against": weighted_againsts,
            "abstain": weighted_abstains,
            "total_weighted": total_weighted,
            "passed": passed,
            "details": {
                "for_percent": weighted_fors / total_weighted * 100 if total_weighted else 0
            }
        }

    def _ranked_choice(self, votes: List[Vote], agents) -> dict:
        """
        Ranked-choice voting (simplified: each vote is a ranked list).
        Used when there are multiple competing proposals.
        """
        # votes are assumed to contain ranked preferences:
        # vote.preferences = ["option_a", "option_b", "option_c", ...]
        # Simplified implementation: first-choice majority
        first_round = {}
        for vote in votes:
            first_choice = vote.preferences[0]
            first_round[first_choice] = first_round.get(first_choice, 0) + 1

        total = sum(first_round.values())
        winner = max(first_round, key=first_round.get)

        if first_round[winner] > total / 2:
            # Majority achieved
            return {"mechanism": "ranked_choice", "winner": winner,
                    "rounds": 1, "passed": True, "details": first_round}
        else:
            # Eliminate lowest, redistribute (simplified: just report)
            return {"mechanism": "ranked_choice", "winner": None,
                    "rounds": 1, "passed": False, "details": first_round}

    def _liquid_democracy_vote(self, bill: Bill, votes: List[Vote],
                                agents: Dict[str, Agent]) -> dict:
        """
        Vote with delegated powers.
        Agents can delegate their vote to another agent.
        Delegations are transitive and acyclic.
        """
        delegation_chain, powers = self._resolve_delegations(agents)

        # Map votes through delegation chain
        for vote in list(votes):
            final_delegate = delegation_chain.get(vote.agent_id, vote.agent_id)
            if final_delegate != vote.agent_id:
                # Find or create a vote for the final delegate
                existing = next((v for v in votes
                                if v.agent_id == final_delegate), None)
                if existing:
                    # Merge: delegate's vote weight increases
                    pass  # weight already accounted for in _resolve_delegations

        # Now tally with resolved powers
        return self._weighted_vote(bill, votes, agents)
```

### 12.4 Full Parliament Session

```python
class Parliament:
    """
    Orchestrates the full parliamentary cycle:
    Proposal -> Committee Review -> Debate -> Vote -> Execution
    """

    def __init__(self, agents: Dict[str, Agent],
                 speaker: str, clerk: str,
                 voting_mechanism: str = "weighted",
                 min_quorum: float = 0.5):
        self.agents = agents
        self.speaker = speaker
        self.clerk = clerk
        self.voting_system = VotingSystem(voting_mechanism)
        self.min_quorum = min_quorum
        self.bills: Dict[str, Bill] = {}
        self.passed_bills: List[dict] = []
        self.failed_bills: List[dict] = []
        self.committees: Dict[str, List[str]] = {}  # domain -> agent_ids

    def register_committee(self, domain: str, member_ids: List[str]):
        """Register a standing committee for a domain."""
        self.committees[domain] = member_ids

    def submit_bill(self, bill: Bill) -> str:
        """Submit a bill for consideration."""
        self.bills[bill.bill_id] = bill
        return bill.bill_id

    def run_parliamentary_cycle(self, bill_id: str) -> dict:
        """Run the full cycle for a bill."""

        bill = self.bills.get(bill_id)
        if not bill:
            raise ValueError(f"Bill {bill_id} not found")

        # Phase 1: Committee Review (if committee exists for domain)
        committee_report = None
        if bill.domain in self.committees:
            committee_report = self._committee_review(bill)

        # Phase 2: Floor Debate
        debate = DebateSession(
            bill=bill,
            speaker_agent=self.speaker,
            max_rounds=3,
            speaking_time_limit=300
        )
        debate.hold_session(
            agents=self.agents,
            pro_arguments=lambda b, c=False: f"Supporting: {b.motion_text}",
            con_arguments=lambda b: f"Opposing: {b.motion_text}"
        )

        # Phase 3: Voting
        votes = self._collect_votes(bill)
        tally = self.voting_system.tally(bill, votes, self.agents)

        # Phase 4: Resolution
        result = self._resolve_outcome(bill, tally, votes, debate.transcript)

        # Record by Clerk
        self._clerk_record(result)

        return result

    def _committee_review(self, bill: Bill) -> dict:
        """Refer bill to standing committee for specialist review."""
        committee = self.committees.get(bill.domain, [])
        if not committee:
            return None

        review = {
            "committee_domain": bill.domain,
            "members": committee,
            "recommendation": None,  # "pass", "pass_amended", "reject"
            "amendments_proposed": [],
            "minority_opinion": None,
            "review_complete": True
        }

        # Each committee member weighs in
        member_votes = []
        for member_id in committee:
            agent = self.agents.get(member_id)
            if agent:
                score = agent.expertise_for(bill.domain)
                member_votes.append({
                    "agent": member_id,
                    "recommend": "pass" if score > 0.5 else "reject",
                    "confidence": score
                })

        # Aggregate (simple majority of committee)
        passes = sum(1 for m in member_votes if m["recommend"] == "pass")
        rejects = sum(1 for m in member_votes if m["recommend"] == "reject")

        if passes > rejects:
            review["recommendation"] = "pass"
        elif rejects > passes:
            review["recommendation"] = "reject"
        else:
            review["recommendation"] = "pass_amended"

        bill.status = BillStatus.UNDER_DEBATE
        return review

    def _collect_votes(self, bill: Bill) -> List[Vote]:
        """Collect votes from all registered agents."""
        votes = []

        for agent_id, agent in self.agents.items():
            if agent_id == self.speaker and len(self.agents) > 1:
                # Speaker only votes to break ties (in some systems)
                continue

            # Record attendance
            agent.attendance_record.append(True)

            # Simulate vote decision based on expertise
            expertise = agent.expertise_for(bill.domain)
            # Simple heuristic: align vote with expertise
            if expertise > 0.6:
                decision = VoteType.FOR
            elif expertise < 0.3:
                decision = VoteType.AGAINST
            else:
                decision = VoteType.ABSTAIN

            vote = Vote(
                agent_id=agent_id,
                bill_id=bill.bill_id,
                vote=decision,
                weight=agent.effective_voting_power(),
                rationale=f"Expertise in {bill.domain}: {expertise:.2f}"
            )
            votes.append(vote)

        return votes

    def _resolve_outcome(self, bill: Bill, tally: dict,
                         votes: List[Vote], transcript: List[dict]) -> dict:
        """Determine the outcome and dispatch execution if passed."""

        # Check quorum
        quorum = len(votes) / len(self.agents) >= self.min_quorum
        if not quorum:
            bill.status = BillStatus.FAILED
            return {
                "bill_id": bill.bill_id,
                "outcome": "FAILED (no quorum)",
                "status": BillStatus.FAILED
            }

        if tally["passed"]:
            bill.status = BillStatus.PASSED
            result = {
                "bill_id": bill.bill_id,
                "title": bill.title,
                "outcome": "PASSED",
                "status": BillStatus.PASSED,
                "tally": tally,
                "votes": [(v.agent_id, v.vote.value, v.weight) for v in votes],
                "amendments": bill.amendments,
                "transcript": transcript,
                "passed_at": datetime.now(),
                "motion_text": bill.current_text,
                "success_criteria": bill.success_criteria
            }
            self.passed_bills.append(result)
        else:
            bill.status = BillStatus.FAILED
            result = {
                "bill_id": bill.bill_id,
                "title": bill.title,
                "outcome": "FAILED",
                "status": BillStatus.FAILED,
                "tally": tally,
                "votes": [(v.agent_id, v.vote.value, v.weight) for v in votes],
                "transcript": transcript,
                "failed_at": datetime.now()
            }
            self.failed_bills.append(result)

        return result

    def _clerk_record(self, result: dict):
        """Clerk agent records the full decision record."""
        if self.clerk:
            # In production: write to database or log
            entry = {
                "type": "parliamentary_record",
                "timestamp": datetime.now().isoformat(),
                "clerk": self.clerk,
                "bill_id": result.get("bill_id"),
                "outcome": result.get("outcome"),
                "record_kept": True
            }
            # Simulate archival
            pass

    def execute_passed_bill(self, result: dict):
        """Dispatch execution of a passed bill to executor agents."""
        if result["outcome"] != "PASSED":
            return {"error": "Cannot execute a non-passed bill"}

        execution = {
            "bill_id": result["bill_id"],
            "action": result["motion_text"],
            "criteria": result["success_criteria"],
            "executor": "executor_agent_pool",
            "status": "dispatched",
            "dispatched_at": datetime.now()
        }

        # In production: dispatch to executor agents
        return execution


### USAGE EXAMPLE ###

def example_session():
    # Create agents
    agents = {
        "security_agent": Agent("security_agent",
                                {"security": 0.95, "infrastructure": 0.60, "cost": 0.30}),
        "sre_agent": Agent("sre_agent",
                           {"infrastructure": 0.92, "performance": 0.85, "security": 0.60}),
        "fin_agent": Agent("fin_agent",
                           {"cost": 0.90, "infrastructure": 0.40, "security": 0.20}),
        "product_agent": Agent("product_agent",
                               {"product": 0.88, "cost": 0.50, "performance": 0.60}),
        "legal_agent": Agent("legal_agent",
                             {"legal": 0.90, "security": 0.50, "privacy": 0.95}),
    }

    # Create parliament
    parliament = Parliament(
        agents=agents,
        speaker="sre_agent",
        clerk="legal_agent",
        voting_mechanism="weighted",
        min_quorum=0.6
    )

    # Register committees
    parliament.register_committee("security", ["security_agent", "legal_agent"])
    parliament.register_committee("cost", ["fin_agent", "product_agent"])
    parliament.register_committee("infrastructure", ["sre_agent", "security_agent"])

    # Create a bill
    bill = Bill(
        title="Adopt Redis Cluster for Session Caching",
        motion_text="Migrate session storage from in-memory to Redis Cluster "
                    "across all production nodes over Q3.",
        rationale="Current in-memory session storage does not survive pod restarts. "
                  "Redis Cluster provides HA, persistence, and 3x throughput.",
        sponsor="sre_agent",
        domain="infrastructure",
        urgency="normal",
        success_criteria=[
            "Zero session loss during pod restarts",
            "P99 latency under 5ms for session reads",
            "100% migration of existing sessions within 30 days"
        ],
        resource_impact={
            "compute_units": 500,
            "execution_time_estimate": "30 days",
            "dependencies": ["kubernetes_cluster"],
            "rollback_plan": "Keep in-memory fallback; revert DNS within 1 hour"
        }
    )

    # Submit and run
    bill_id = parliament.submit_bill(bill)
    result = parliament.run_parliamentary_cycle(bill_id)
    print(f"Bill {bill_id}: {result['outcome']}")

    if result["outcome"] == "PASSED":
        execution = parliament.execute_passed_bill(result)
        print(f"Execution: {execution['status']}")
```

---

## Summary: When to Choose the Parliament Pattern

```
USE PARLIAMENT WHEN:
  - Decision is strategic, high-impact, or irreversible
  - Multiple legitimate perspectives exist
  - An auditable decision trail is required
  - Buy-in from multiple stakeholders is necessary
  - The cost of a bad decision exceeds the cost of deliberation

DON'T USE PARLIAMENT WHEN:
  - Decisions must be made in seconds
  - The decision is trivial or fully reversible
  - Only one agent has relevant expertise
  - Continuous real-time control is needed
  - The overhead of deliberation exceeds the benefit
```

The Parliament Pattern is the most robust decision-making pattern in the multi-agent toolkit for complex, high-stakes, multi-stakeholder decisions. It sacrifices speed for legitimacy, and it demands careful engineering to manage its inherent complexity -- but when the situation demands collective wisdom, structured deliberation, and an unassailable decision record, there is no better pattern.
