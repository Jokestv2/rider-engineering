# Rider Engineering

**Rider engineering is the practice of designing task-class-specific control layers that drive general-purpose agent harnesses through a bounded workflow, and that treat harness sessions — and therefore model context — as a designed resource rather than a runtime emergency.**

> *Status: v0.1. This is a working definition, not a finished one. It is published to be argued with.*

---

## The short version

A **harness** — Claude Code, Codex, Hermes — is general-purpose by design. It will attempt almost anything, which means it is tuned for nothing in particular. Give it a large, structured, multi-day task and it will make a plausible attempt, but it holds no opinion about what *that kind of task* requires: which step must precede which, where a human should be consulted, what has to be written down before it is safe to spend a week of GPU time.

A **rider** is that missing opinion, made explicit and executable. It is written for one bounded family of tasks and encodes what the family demands: the phases, the artifact each phase must produce, the gates that must clear before advancing, and how the work is partitioned across harness sessions so that model context stays clean as the task grows.

> **A rider does not do the work. It decides what work happens next, what must be true before it starts, and in which session it runs.**

The name is the metaphor. A harness is general-purpose equipment that makes a powerful animal steerable; it does not know where you are going. The rider supplies the destination, the pacing, and the judgment about when to push and when to rest. Same animal, same harness, different rider — very different journey.

**Why a new term?** [Harness engineering](https://github.com/ai-boost/awesome-harness-engineering) is already a named practice, and it is a good one — but its object is the harness itself: the loop, the tools, the permissions, the observability, all built to serve arbitrary tasks. Rider engineering has a different object and an inverted goal. It asks what you build *on top of* a finished harness when you already know what you are trying to do, and it buys reliability by surrendering generality inside a stated boundary. Those are different design problems, and conflating them is why "make the agent do research properly" usually decays into a longer system prompt.

---

## Why the harness alone is not enough

Take a concrete task: *get a computer-vision research idea to a CVPR submission.*

Hand it to a bare harness and three failure modes appear quickly.

**1. No enforced order of operations.** The harness will happily start writing training code before anyone has checked whether the idea is already published. Nothing in a general-purpose loop insists that literature review precede commitment. That ordering constraint is real, domain-specific knowledge — and it lives nowhere.

**2. Context collapse.** A serious literature review touches a hundred papers. If all of it lands in one session's context, the session that later designs the experiments is reasoning over a window that is mostly stale paper text. Modern harnesses do have cross-session mechanisms — memory files, resume, subagents with isolated windows — so this is not a capability gap. It is a *knowledge* gap: nothing in a general harness knows that literature review, for this task class, should be its own session whose only durable output is a six-page structured survey. Absent that knowledge, the mechanisms get configured ad hoc, per user, per project, and mostly after something has already gone wrong.

**3. Human input arrives at the wrong time.** In research, the highest-leverage human correction happens after the literature is known and before compute is committed. A general harness either asks constantly or not at all; it has no basis for knowing that *this particular moment* is when the advisor's opinion is worth the most.

Each failure is the same absence: a layer that knows the shape of this class of task. That layer is the rider.

---

## Where a rider sits

| Layer | Responsibility | Scope | Example |
|---|---|---|---|
| **Model** | Generate tokens | — | Claude, GPT, Gemini |
| **Harness** | Run the agentic loop: call the model, execute tool calls, manage context within a session, decide when to stop | General-purpose | Claude Code, Codex, Hermes |
| **Rider layer** | Drive a bounded task class to completion: sequence phases, enforce gates, place human checkpoints, allocate work across sessions, own durable state | One task class | A CS-research rider, a security-audit rider, a clinical-trial-protocol rider |

The rider layer has two parts, separated below: a **rider**, which is the specification, and a **rider runtime**, which executes it. Two boundaries keep the layering honest:

- **The harness owns context *within* a session. The rider owns context *across* sessions.**
- **The rider constrains the trajectory. The harness chooses the steps.**

*Two terminology notes.* Some writing separates the **scaffold** (system prompt, tool descriptions, output parsing, context strategy) from the **harness** (the execution loop proper); the table above folds them together, since a rider sits above both. And "harness" here means an *agent* harness, not an *evaluation* harness in the `lm-evaluation-harness` / SWE-bench sense.

---

## Formal definition

A **rider** is a specification

```text
R = ( T, I, Φ, Γ, Σ, Ω )
```

A **rider runtime** is the program that executes a rider against a harness *H*, written `run(R, H)`. The distinction matters and is easy to lose: the rider is the encoded opinion, the runtime is what enforces it. Every claim below about a rider "refusing," "gating," or "deciding" is discharged by the runtime; the rider merely says what must hold. A rider without a runtime is a design document; a runtime without a rider is a general workflow engine, which is very nearly what you should build it from (see [below](#what-a-rider-is-not)).

### T — Task class

The bounded family of tasks the rider covers, stated precisely enough that membership is decidable, together with the **terminal condition** that defines completion. *"Take a CS research idea to a submission at a target top-tier venue"* is a task class. *"Do research"* is not: no terminal condition, no structure shared across instances.

Note that the *specific* venue belongs to intake, not to T. If it were part of T, running the same rider for NeurIPS would be a different rider — which is the wrong answer.

Bounded scope is constitutive, not incidental. A rider that covers everything has stopped encoding anything.

### I — Intake

The typed inputs the rider requires before any work begins, plus their validation. Intake is a contract: the runtime refuses to start until it holds what it needs. For the CS-research rider — a venue target with deadline and reviewing norms, a research idea at some stated maturity, and an inventory of accessible compute.

Validated intake is written into Ω as the phase-0 artifact. This is where a rider absorbs the situational facts a general harness would otherwise rediscover in every session.

### Φ — Phase structure

A directed graph over phases:

```text
Φ = (P, →)     P = {phi_0, phi_1, ..., phi_n}     → ⊆ P × P
```

Each phase is a pair `(objective, exit artifact)`. The **objective** is delegated to the harness in natural language — it states *what to achieve*, never *how*. The **exit artifact** is the durable document, dataset, or code the phase must produce; a phase that produces no artifact has not run. `phi_0` is intake, whose exit artifact is the validated intake record.

`→` is the set of legal transitions, and it is a graph rather than a sequence on purpose. Research is not a DAG: experimental results routinely send you back to the literature. What a rider fixes is not a straight line but which moves are permitted, and each edge is guarded by a gate. Reflexive edges are legal — a phase whose artifact fails its own gate is sent back to itself.

### Γ — Gates

The decision function over the artifact store:

```text
γ : Ω × P → { advance(phi_j), revisit(phi_j), abort }
```

restricted to edges present in `→`. A gate is automated, human, or both.

Gates are where a rider encodes taste. *"An experiment plan may not be executed until the user has approved it"* is a gate. So is *"if the ablation fails to separate the proposed mechanism from the baseline, return to experiment design."* Γ includes a **terminal gate** that checks T's terminal condition.

Every gate resolution is itself written into Ω as a decision artifact — including human approvals. Otherwise a resumed run cannot tell which gates have already cleared.

Gate *placement* is the core design skill of rider engineering. Human attention is the scarcest resource in the loop; a rider is largely a theory of where to spend it.

### Σ — Session policy

For each phase, given current state:

```text
Σ : P × Ω → ( session count, seed, join spec )
```

The **seed** is a projection of Ω — precisely what the session starts from, and nothing else. The **session count** may depend on Ω, resolved at run time: one session per experiment group, where the groups are read from the plan artifact. The **join spec** says how the resulting artifacts are merged when a fan-out rejoins.

A phase's seed is the *only* thing its sessions begin from. There is no separate "what crosses out" — what leaves a phase is its exit artifact, and what reaches the next phase is whatever that phase's seed projects from Ω. Anything a session learned that is not in an artifact is gone, by construction.

Σ is stated in harness-independent terms — what must carry, what must stay isolated — and specialized by a **binding profile** per harness at `run(R, H)`, since window sizes and compaction behavior differ.

### Ω — Artifact store

The durable state that outlives any session: validated intake, the survey, the plan, result logs, gate decisions, the draft. Ω is the rider's memory and the only thing permitted to be.

Artifacts are append-only and versioned. Gates read the latest version; a revisit supersedes rather than deletes, so the history of a claim that was rejected twice remains inspectable.

---

## The central claim: sessions are the unit of context management

A harness manages the context window it is currently inside. It cannot manage the context of a session that has not started, because it has no model of how far the task extends.

A rider does. So the rider decides where sessions begin and end — and every session boundary *should be designed as* a **distillation boundary**: a point where a large volume of exploratory context is deliberately collapsed into a small durable artifact, and the rest is discarded on purpose.

This inverts the usual framing. Context management is normally damage control: the window is filling, so compact it. Under rider engineering it is a decision made in advance. The literature-review session is *supposed* to end. Its several hundred pages of paper content were never meant to survive; the six-page structured survey is the entire point, and the next session should start from the survey and nothing else.

Three consequences follow.

**Sessions get sized to their distillate, not to the window.** The right question is not "how much fits?" but "what should be left when this is over?" A phase whose artifact is a one-page decision need not hand forward the reasoning that produced it.

**The artifact store must be sufficient for restart.** If Ω genuinely captures what each phase produced, any session can be reconstructed from artifacts alone — after a crash, a week's pause, or a switch to a different harness. This is not merely a nice property; it is the sharpest available *test* of whether a phase decomposition is any good. If a fresh session cannot pick up the work from Ω, some phase is smuggling state through the context window instead of writing it down.

**Sessions become the natural unit of parallelism.** Once phases have typed artifact interfaces, independent phases — three baselines on three machines — run concurrently in separate sessions and rejoin at a gate, without polluting one another's context.

---

## Design principles

1. **Bound the task class.** If you cannot state the terminal condition, you are not writing a rider yet.
2. **Every phase produces an artifact.** Work whose value lives only in a context window is work that will be lost.
3. **Constrain trajectory, not steps.** Specify what must be true between phases; let the harness decide how to get there. That delegation is precisely what a capable harness is for.
4. **Gate before commitment, not after.** Put human checkpoints immediately upstream of expensive or irreversible steps, where correction is cheapest.
5. **Treat context as a designed resource.** Decide in advance what each session should leave behind.
6. **Make revisiting first-class.** Real workflows loop. A rider that can only move forward will either lie about progress or stall.
7. **Be resource-aware.** A rider that plans experiments it cannot run is not flexible, it is uninformed. Compute inventory belongs in intake and constrains the plan.
8. **Keep the rider harness-agnostic and push harness specifics into the binding profile.** A rider that only works on one harness has confused its own structure with an implementation detail.

---

## Worked example: a CVPR rider

**T** — take a computer-vision research idea to a submission at a target top-tier venue. Terminal condition: a submission package satisfying the venue's stated requirements, signed off by the user.

**I** — venue target (CVPR 2027; deadline, page limit, reviewing norms); research idea with stated maturity; compute inventory (nodes, GPUs, memory, walltime, storage).

**Ω** — intake record, subtopic list, structured survey, claim statement, experiment plan, code + smoke results, result logs, findings document, draft, rebuttal memo, and the decision record for every gate.

| Phase | Objective (delegated) | Exit artifact | Gate on exit | Session policy (Σ) |
|---|---|---|---|---|
| **0. Intake** | Validate and normalize the inputs | Intake record: venue profile, idea statement, compute inventory | *Automated:* all three present and well-formed; deadline is in the future. Otherwise **abort**. | 1 session. Seed = raw user inputs. |
| **1. Survey scoping** | Decompose the idea's neighborhood into searchable subtopics | Subtopic list with search strategy per subtopic | *Automated:* list is non-empty and covers the idea's stated components. | 1 session. Seed = intake record. |
| **2. Literature survey** | For each subtopic, establish what exists, what is closest, what is unclaimed | Structured survey: positioning map, closest prior work, standard benchmarks and baselines | *Automated:* every subtopic covered; venue's expected baselines present. *Human:* **if closest prior work subsumes the idea, abort or revisit phase 0** with a revised idea. | *n* sessions, *n* = length of the subtopic list. Seed = intake record + one subtopic. Join = merge per-subtopic findings into one survey, de-duplicating citations. |
| **3. Idea refinement** | Sharpen the idea against the survey into a defensible claim | Claim statement: contribution, falsifiable hypothesis, evidence the venue would require | *Human:* user approves the claim. Rejection revisits phase 3. | 1 session. Seed = survey + intake record. |
| **4. Experiment design** | Design the minimum experiment set that could establish the claim, within available compute | Experiment plan: per-experiment hypothesis, dataset, baselines, ablations, metrics, compute estimate, experiment groups | *Automated:* compute estimate within inventory and finishing before the deadline. *Human:* user approves before any compute is spent. | 1 session. Seed = claim + survey positioning map + compute inventory. |
| **5. Implementation** | Build the pipeline and validate it end to end at small scale | Working code, smoke-test results | *Automated:* smoke test reproduces a known baseline within tolerance on a subsample; failure revisits phase 5. | 1 session. Seed = plan + code interfaces. Debugging noise stays inside; the artifacts are the code and a status report. |
| **6. Execution** | Run the plan, monitor, report | Per-run result logs, checkpoints, tables, figures | *Automated:* every planned run has terminated or is explicitly recorded as failed. | *k* sessions, *k* = number of experiment groups in the plan. Seed = code + that group's plan rows. Join = concatenate per-group result tables into one results artifact. |
| **7. Analysis** | Test each hypothesis against results; state honestly what held | Findings document; explicit claim-vs-evidence verdict | *Human:* if the claim is unsupported, revisit phase 3 (claim was wrong) or 4 (test was wrong). | 1 session. Seed = plan + results tables, **not** run logs. |
| **8. Writing** | Produce a submission meeting the venue's standards | Paper draft, figures, supplementary | *Automated:* page limit, format, anonymization all satisfied. | 1 session. Seed = survey + claim + plan + findings. |
| **9. Review readiness** | Attack the paper as a hostile reviewer would; fix or acknowledge | Rebuttal-anticipation memo; revised draft | **Terminal gate:** package complete, user signs off. | 1 session, deliberately adversarial. Seed = draft + findings only. |

Note what the session policy is doing. Phase 7 does *not* inherit phase 6's context, because analysis should reason over results rather than over the several thousand lines of training log that produced them. Phase 9 starts fresh precisely so the model has not been anchored by having written the draft itself. Phases 1 and 2 are split so that the fan-out width is a function of an artifact that already exists when phase 2 begins, rather than something discovered mid-phase. None of these are context-window emergencies; they are decisions about where knowledge should be distilled, available only to a layer that can see the whole task.

---

## What a rider is not

**Not a prompt or prompt template.** The real distinction is not persistence — a system prompt is re-injected every session — but *where enforcement lives.* A prompt requests compliance from the model. A gate is control flow outside the model's discretion. The moment the model can talk itself past a checkpoint, you have a suggestion, not a gate.

**Not a harness.** A rider never calls the model or handles a tool call. It decides what session runs next and what goes into it.

**Not a rigid pipeline or DAG.** A pipeline scripts steps; a rider specifies conditions between phases and delegates the steps. If you find yourself writing the harness's reasoning for it, you have built a pipeline and given up the thing that made the harness worth using.

**Not a multi-agent framework.** A rider may spawn parallel sessions, but it is individuated by its *task class*, not by its topology.

**Not an orchestrator or a workflow engine — and this is the objection worth taking seriously.** A durable-execution workflow engine maps onto the tuple almost component for component: nodes are Φ, conditional edges and interrupts are Γ, a checkpointer is Ω, per-node state schemas are Σ. That mapping is real, and rather than deny it, the claim is that a rider is typically *implemented on* such an engine. The difference is content versus mechanism. A workflow engine is domain-empty: it will faithfully execute any graph you hand it, including a bad one. A rider *is* the domain opinion — this phase decomposition, these gates at these points, this distillation policy — and rider engineering is the practice of getting that content right. LangGraph gives you `interrupt`; it does not tell you that the interrupt belongs immediately before compute commitment and not after. That second question is the one rider engineering is about — see [Relationship to existing work](#relationship-to-existing-work) for what that content actually consists of.

---

## Relationship to existing work

The worked example deliberately resembles [The AI Scientist](https://sakana.ai/ai-scientist/) and [its successor](https://github.com/SakanaAI/AI-Scientist-v2), which automate a comparable idea-to-paper pipeline end to end, and the broader autonomous-research literature. The phase list is not the contribution and should not be read as one; that decomposition is roughly what any researcher would name.

**The contribution, stated once:** rider engineering claims that two things usually treated as implementation detail are the primary design variables, and belong in the specification.

- **Gate placement.** Where the human checkpoints go — immediately upstream of irreversible commitment rather than at the end as review — determines how much a wrong turn costs, and how much of the human's attention it costs to prevent one.
- **Session policy.** *Where you cut the sessions* determines what the model is reasoning over at each step, and therefore the quality of the output. This is the variable with no established vocabulary, which is most of why the practice is worth naming separately.

Everything else in the tuple exists to make those two expressible and enforceable. A rider is also not required to be autonomous: the CVPR rider above expects a human at five gates and is better for it.

---

## Evaluating a rider

Rider engineering is only a discipline if riders can be judged. Some candidate criteria:

**Restart soundness.** Kill the run at any phase boundary. Can a fresh session resume from Ω alone? Failures localize precisely to phases with leaky artifacts. (Long-running phases need intra-phase checkpoint artifacts to make this hold at finer grain — see the per-run checkpoints in phase 6 above.)

**Gate precision.** How often does a human gate change the outcome? A gate that is always rubber-stamped costs attention and buys nothing; one that always fires is placed too late.

**Context efficiency.** Total tokens to reach the terminal gate, rider versus bare harness on the same instance — and how that ratio moves as instances get larger.

**Trajectory conformance.** How often does execution need out-of-band human rescue that no gate anticipated? Each instance names a missing gate.

**Transfer within class.** Same rider, different idea, different venue. If it only works on the instance it was developed against, T was drawn wider than the structure actually encoded.

**Limitations.** These are proposed criteria, not results. No rider described here has been evaluated against a bare-harness baseline, and the framework's central empirical claim — that designed session boundaries measurably improve output quality — is currently an argument, not a finding.

---

## When a rider is overkill

Riders are not free. Writing one costs the effort of decomposing a domain, and running one costs the latency of gates and the overhead of re-seeding sessions from artifacts. That price is worth paying when the task is long enough to exceed a single session, expensive enough that a wrong turn is costly, repeated enough to amortize the design, and structured enough that the phases genuinely recur across instances.

When a task fits comfortably in one session, or its shape differs every time, a bare harness is the better tool and a rider is ceremony. The generality you give up has to buy something back.

---

## Glossary

| Term | Definition |
|---|---|
| **Harness** | The execution layer around a model: calls it, handles tool calls, manages context within a session, decides when to stop. General-purpose. Compare the [HuggingFace agent glossary](https://huggingface.co/blog/agent-glossary), whose usage this follows. |
| **Rider** | A task-class-specific specification that drives one or more harness sessions through a bounded workflow: phases, gates, session policy, durable state. |
| **Rider runtime** | The program that executes a rider against a harness, enforcing gates and constructing session seeds. |
| **Phase** | A unit of delegated work with a natural-language objective and a required exit artifact. |
| **Gate** | A decision function over artifacts — automated, human, or both — resolving to advance, revisit, or abort. |
| **Artifact** | Durable output of a phase; the only state permitted to cross a session boundary. Append-only and versioned. |
| **Session policy** | The rider's mapping from phases to sessions, seeds, and join specifications. |
| **Distillation boundary** | A session boundary at which exploratory context is deliberately collapsed into a small durable artifact and the remainder discarded. |
| **Binding profile** | The harness-specific specialization of a session policy applied at `run(R, H)`. |

---

## Contributing

This definition is a first pass and several parts of it are unsettled — the split between Γ and Σ, whether Φ's transition graph should be explicit or derived from gates, and whether "rider" survives contact with people who already have a workflow engine. Issues and pull requests that sharpen it, or that argue the framing is wrong, are welcome.
