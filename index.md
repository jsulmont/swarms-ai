# Why AI Swarms Cannot Build Architecture

*2,000 AI agents. 1.7 million lines of Rust. A working browser engine in a week.*

*Cursor's [FastRender](https://cursor.com/blog/scaling-agents) ([GitHub](https://github.com/wilsonzlin/fastrender)) is not a failure. It's a success — and the success reveals something important.*

*The code runs. Tests pass. Features work. But look closer: two HTTP clients, two MP4 parsers, two JSON libraries. Agents solving the same problems independently, differently. Each made locally reasonable choices. The aggregate works, but it doesn't cohere.*

*This is not a bug in the experiment. It's the natural attractor state of swarms. That raises a sharper question: can verification tame swarm divergence? Gensyn's [Verde paper](https://arxiv.org/abs/2502.19405) is a useful test case.*

---

## When Agents Diverge

Consider a concrete scenario: two agents independently need an HTTP client.

Agent A asks: "What Rust crate should I use for HTTP requests?"  
Agent B asks the same question, moments later.

They get different answers. Maybe `reqwest` vs `ureq`. Maybe even the same model gave different responses due to temperature sampling.

**Is this a problem?**

For FastRender's goals: no. Both crates work. The code compiles. Tests pass. The browser renders pages. Mission accomplished.

But even if they'd happened to pick the *same* client — could we be certain they'd pick it again? Run the experiment tomorrow with different temperature samples, different agent-to-task assignments, maybe a model update overnight. You might get `reqwest` everywhere this time, `hyper` next time, a mix the third time.

The experiment is not reproducible. The output is a sample from a distribution, not a deterministic function.

And what if the divergence was more fundamental?

- One agent picks async, another picks sync
- One expects errors as `Result`, another expects panics
- One assumes UTF-8, another assumes bytes

Now agents' outputs are *incompatible*. The swarm produces code that doesn't compose.

Or does it? In Rust, the mighty `cargo check` would catch the mismatch. One function returns `Result`, another expects a panic — compilation fails. And Rust's crystal-clear error messages might give the agent enough information to self-correct. The compiler becomes an implicit verifier, forcing convergence through feedback.

This is one reason Rust works well for AI-assisted development: the type system acts as a safety net, catching a class of coordination failures automatically. In Python or JavaScript, these incompatibilities would surface at runtime — or never.

## Scaling the Problem: N Models

FastRender hit a single model (or model family). What happens when a swarm queries heterogeneous models?

- Some agents hit GPT-5.2
- Some hit Claude Opus 4.2
- Some hit Llama 4 running locally
- Some hit Mistral Large via API

For factual queries with clear answers (2+2=?), models agree. But for judgment calls — architectural decisions, library choices, error handling philosophy — models diverge. Each has different training data, different RLHF tuning, different implicit preferences.

The swarm's coordination problem compounds: not just "agents didn't share state" but "agents consulted different oracles with different opinions."

## Why Does the Same Model Give Different Answers?

Even hitting the *same* model with the *same* prompt can yield different outputs. Why?

**Sampling randomness.** Temperature > 0 means probabilistic token selection. Same logits, different dice rolls, different text.

**But even with temperature = 0?**

Floating-point arithmetic is non-associative:

```
(a + b) + c ≠ a + (b + c)
```

Each addition involves rounding. The order of operations affects the result.

GPUs are massively parallel. Matrix multiplications split across thousands of threads. The order those partial sums accumulate? Non-deterministic. Depends on thread scheduling, memory layout, hardware architecture.

Run the same model, same weights, same prompt, same temperature on:

- An NVIDIA H100 vs an RTX 5090
- CUDA 13.1 vs CUDA 12.8
- Different batch sizes
- Different memory strides

Even at temperature = 0, hardware-level nondeterminism can perturb logits enough to change outputs. The model isn't "wrong." Floating-point math on parallel hardware is inherently non-deterministic unless you force it to be.

## Local Intelligence, Global Incoherence

Here's the core problem: agents can be individually excellent and collectively incoherent.

Each agent has the same training data, the same priors, the same general knowledge. You might expect this to produce convergence. It doesn't.

**Correlation is not coordination.**

Two agents with identical training who independently choose an HTTP client are not coordinating — they're sampling from the same distribution. If the distribution has a strong mode, you'll often get the same answer. But "often" is not "always." Run the swarm enough times, or scale it large enough, and divergence becomes certain.

When a swarm produces working code, it's tempting to say coordination "emerged." But this is survivorship bias. The redundant dependencies, the inconsistent error handling — these are also emergent. They just don't break the build.

LLM priors are a shared bias, not a coordination mechanism. This is why the problem is structural, not incidental. The next section makes this precise.

## Why Swarms Cannot Produce Architecture

This isn't a limitation of current models. It's structural. Let me make this precise.

**What is architecture?**

Architecture is a system of **global invariants** — properties that must hold across the entire codebase, not just within individual components.

- "All error handling uses Result, not panics" (consistency invariant)
- "Module A may depend on Module B, but not vice versa" (dependency invariant)
- "Exactly one HTTP client library" (uniqueness invariant)
- "All public APIs use these types" (interface invariant)

These invariants create **long-range coupling**. A decision in one part of the system constrains decisions everywhere else. That's the point — architecture is *constraint propagation*. You make a choice once, and it ripples outward, ruling out entire classes of future choices.

Enforcing invariants requires:
1. **Global visibility** — seeing all the places where the invariant applies
2. **Temporal persistence** — remembering that the invariant exists across all future decisions
3. **Authority** — the ability to reject work that violates the invariant

**What is a swarm?**

A swarm is a collection of agents with the opposite properties:

- **Local optimization.** Each agent sees its task, not the whole system. It optimizes for "complete my task," not "maintain global coherence."
- **Weak coupling.** Agents don't share state. At best they share a repo, but the repo is data, not a coordination mechanism. There's no "lock" on architectural decisions.
- **Non-persistent intent.** No agent remembers *why* a prior decision was made. Context is per-task. When the task ends, the reasoning vanishes.
- **No authority.** No agent can reject another agent's work for violating an invariant it doesn't know about.

**The mismatch is mathematical.**

Architecture requires constraint propagation: a decision at time T must constrain decisions at time T+1, T+2, ... T+N.

Swarms have no propagation mechanism. Each agent starts fresh. Even if agent A makes a perfect architectural decision, agent B — running concurrently or later — has no channel through which A's constraint reaches B.

This is not a bug. It's the definition of "swarm." Swarms are parallel, independent, stateless. Architecture is sequential, coupled, stateful. You cannot get the second from the first without adding exactly the machinery that makes it no longer a swarm.

**Scaling makes it worse, not better.**

With 10 agents, you might get lucky. The probability that all 10 happen to make compatible choices is low but nonzero.

With 1,000 agents, the probability of accidental coherence drops to near zero. More agents means more independent decisions, more opportunities for divergence, more surfaces for contradiction.

This is not "early tooling" that will be engineered away. It's combinatorics. The number of possible inconsistencies grows faster than any coordination mechanism that preserves the "swarm" property.

**Better models don't fix this.**

Give each agent GPT-10. Make the prompts perfect. The structure of the problem remains:

- No shared decision register
- No persistent architectural memory
- No mechanism to say "this decision was already made; you must comply"

You'll get better local code. You won't get architecture. The agents will make smarter individual choices that still don't cohere globally.

Duplication, drift, and contradiction are not bugs in the swarm — they're the equilibrium state. The attractor. The place the system settles when no force pushes it toward coherence.

**"But what if agents wrote down their decisions?"**

You might think: have each agent write an ADR (Architecture Decision Record) when it makes a choice. Collective memory, built from the bottom up.

But a log is not a coordination mechanism. Who queries the ADRs before each decision? Who interprets them (another LLM, reintroducing non-determinism)? Who enforces compliance? Who resolves conflicts when two agents write contradictory ADRs concurrently?

You can't instruct individual agents to *contribute to* global knowledge. You can only build infrastructure that *imposes* global knowledge on them. The agents don't coordinate — the infrastructure does.

**The conclusion is definitional.**

If you add global visibility, temporal persistence, and enforcement authority to a swarm, you no longer have a swarm. You have a hierarchical system with stochastic workers.

That's fine. That's probably what you want. But call it what it is: the swarm generates candidates; the architecture comes from somewhere else.

## Enter Gensyn: Verified Machine Learning

Gensyn tackles exactly this problem for *distributed ML training*. Their approach is detailed in the [Verde paper](https://arxiv.org/abs/2502.19405).

Their scenario: you want to train a model across untrusted nodes worldwide. How do you know they actually did the computation correctly, rather than returning garbage to collect rewards?

**The naive solutions don't scale:**

- Full replication (re-run everything): prohibitively expensive
- Cryptographic proofs (zkML): 10,000x overhead, minutes per inference step
- Trust and whitelisting: centralizes the system

**Gensyn's approach: Verde + RepOps**

1. **RepOps (Reproducible Operators):** Re-implement ML primitives (matmul, exp, tanh, etc.) to enforce deterministic operation ordering across all hardware. Same inputs → bitwise identical outputs, regardless of GPU architecture.

2. **Verde (Verification via Refereed Delegation):** Optimistic verification. Assume nodes are honest. Verifiers spot-check by re-running portions of computation. If outputs match (thanks to RepOps determinism) → verified. If they diverge → dispute resolution game pinpoints the exact operation that differs, referee re-executes just that step.

3. **Incentive game:** Solvers stake tokens, verifiers check work, whistleblowers can challenge verifiers. Classical blockchain mechanism design.

The key insight: don't verify *semantic correctness* (is this a good model?). Verify *execution fidelity* (did you run the specified computation correctly?).

## The Hidden Constraint: Everyone Runs the Same Library

There's a catch buried in the Verde design: RepOps is a library. Every participating node must run *through RepOps* for verification to work.

Gensyn isn't verifying "did you correctly run PyTorch on your hardware?" They're verifying "did you correctly run *our library* on your hardware?"

**What this means in practice:**

- **Closed ecosystem.** You can't join the network with vanilla PyTorch and get verified. Everyone must use the RepOps runtime.
- **Performance cost.** RepOps forces deterministic operation ordering, sacrificing some GPU parallelism. The paper benchmarks this — overhead varies by operator, but it's non-zero.
- **FP32 only (currently).** IEEE-754 compliance is more reliable at full precision. FP16/BF16 support is harder to make deterministic.

**The blockchain analogy:**

This is like saying "we verify EVM execution" — but everyone must run the same EVM implementation. You're not verifying arbitrary computation. You're verifying execution within a constrained, agreed-upon runtime.

For Gensyn's use case (coordinated distributed training where they control the stack), this works. But you can't retrofit Verde verification onto an existing heterogeneous swarm running stock inference servers with whatever CUDA version they happen to have installed.

The verification is real, but the scope is narrow: a walled garden of compatible nodes, not the open internet of GPUs.

## The Limitation: Fidelity ≠ Equivalence

Verde answers: "Did node X correctly execute Model M with weights W on data D?"

Verde cannot answer:

- "Is GPT-5.2's output equivalent to Claude Opus 4.2's output?"
- "Is this response semantically correct?"
- "Did two different models give compatible answers?"

The protocol requires everyone to agree upfront on *exactly* what computation to run. Verification is against a specification, not against reasonableness or semantic equivalence.

For a homogeneous training swarm (everyone trains the same model), this works. For a heterogeneous inference swarm (agents query different models, expect compatible outputs), Verde is irrelevant.

## Could Swarms Have Verifiers?

**Gensyn's model:**
- Executors (Solvers) — do the computation
- Verifiers — spot-check against *a specification*
- The spec is explicit: Model M, Weights W, Data D, Optimizer O

**FastRender's model:**
- Agents — do the work
- Tests — verify "does it compile? do tests pass?"
- No architectural specification to verify against

The tests are a weak verifier. They check *functional correctness* (output matches expected), not *architectural consistency* (did you make decisions compatible with the rest of the system?).

What's missing is a decision authority that enforces architectural invariants. In "architect in control" mode, the human plays this role — reviewing decisions, catching inconsistencies, enforcing a coherent vision. But that doesn't scale to 2,000 concurrent agents.

**The fundamental asymmetry:**

Gensyn verifies against a *static specification* — the model architecture is fixed before training starts.

A swarm building software has an *evolving specification* — architectural decisions emerge during the work. There's no ground truth to verify against until someone creates it.

## A Swarm Improvement: The Judge Pattern

Perhaps the answer isn't full verification, but *scoped autonomy with escalation*.

Cursor learned this the hard way. Their first attempt at FastRender used flat coordination with file locking — agents would acquire locks before editing, release when done. It failed badly:

> "Twenty agents would slow down to the effective throughput of two or three, with most time spent waiting."

Agents held locks too long, forgot to release them, or failed while holding them. The system was brittle.

Their second attempt: optimistic concurrency control. Simpler, more robust. But a deeper problem emerged:

> "With no hierarchy, agents became risk-averse. They avoided difficult tasks and made small, safe changes instead. This led to work churning for long periods of time without progress."

No single agent would take ownership of hard problems. Everyone made safe local edits. The swarm was busy but not productive.

**The solution: explicit hierarchy**

Cursor landed on a three-role architecture:

- **Planners** — continuously explore the codebase and generate tasks, with ability to spawn sub-planners for specific areas
- **Workers** — accept assigned tasks and focus entirely on completion, without coordinating with other workers
- **Judge** — determines at cycle end whether to continue; the next iteration starts fresh

The key insight from their experience:

> "Too little structure and agents conflict, duplicate work, and drift. Too much structure creates fragility."

**This pattern keeps appearing**

Cursor isn't alone. Similar hierarchical structures show up across the industry — [MetaGPT](https://github.com/FoundationAgents/MetaGPT), [AWS Bedrock multi-agent](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-multi-agent-collaboration.html), [HuggingFace smolagents](https://huggingface.co/docs/smolagents/examples/multiagents), among others. The convergent evolution suggests this isn't arbitrary — it's load-bearing structure.

**What qualifies as needing escalation?**

- Adding a new dependency
- Choosing between architectural alternatives
- Defining interfaces between modules
- Anything that constrains future agents' choices

**The judge could be:**

- A human architect (doesn't scale, but highest fidelity)
- A dedicated LLM with full project context and explicit architectural principles
- A consensus mechanism (multiple agents vote, majority wins)
- A constraint system (rules encoded upfront: "only reqwest for HTTP", "errors as Result not panic")

**The deeper point:**

FastRender's initial agents didn't distinguish between "local implementation detail" and "global architectural decision." Everything was treated the same — pure autonomy, no escalation.

A smarter swarm recognizes: "I need an HTTP client. This affects the whole project. Let me check if one is already chosen, or escalate if not."

Not verification after the fact. Coordination before the act.

**What does this mean in practice?**

Judges reintroduce hierarchy. Planners reintroduce design authority. Constraint systems reintroduce compilers.

At scale, a "swarm with judges" is not a swarm. It's a pipeline with stochastic components. The LLMs generate; the deterministic infrastructure decides, routes, validates, enforces. The more architectural consistency you want, the more infrastructure you need — until the "swarm" is really a compiler with LLM-powered code generation passes.

This isn't a defect in swarms; it's the expected outcome.

The swarm was never going to produce architecture. The swarm produces *candidates*. Something else — hierarchical, stateful, deterministic — must impose coherence.

---

## What About Gensyn?

This exploration started by asking whether Gensyn-style verification could help with swarm coordination. The answer: not directly.

Gensyn solves a narrower problem than it first appears — execution fidelity within a walled garden where everyone runs the same library. That's valuable for distributed training. But it doesn't address semantic equivalence across heterogeneous models, or the coordination problem of swarms making architectural decisions.

Deterministic computation is hashable, disputable, verifiable. Judgment calls are not.

## What Remains Open

The structural argument is settled: swarms cannot produce architecture. But practical questions remain.

**Can we verify semantic equivalence across models?** Probably not — but we can verify behavioral equivalence through testing.

**How much verification do we need?** Depends on stakes. Disposable generation is fine for demos. Infrastructure, finance, medical systems need the same rigor we apply to any critical software. What matters is the artifact and its checks, not whether it came from a human or a swarm.

---

## The Architectural Implication

If you accept the arguments above, one design principle follows:

**Stochastic generators need deterministic shells.**

Let swarms generate. But wrap them in systems that enforce invariants — type checkers, schema validators, constraint solvers, deterministic orchestrators. The creative work can be probabilistic. The structural guarantees cannot. Trust attaches to the shell, not the swarm.

This is not a new idea. It's how we already think about compilers, databases, and distributed systems. The only thing that's new is applying it to AI-generated artifacts — and admitting that generation and verification are fundamentally different problems, requiring different tools.

**What this looks like in practice:**

- **Typed artifacts.** Use languages with strong type systems. Let `cargo check` or `tsc` reject code that doesn't cohere with existing interfaces.
- **Schemas.** Define the shape of valid outputs. Agents generate; schemas reject malformed results.
- **Checkers.** Linters, validators, property tests. Anything that can say "this output is invalid" without understanding how it was produced.
- **Proof-carrying results.** Where possible, require agents to produce not just answers but evidence: test cases, type derivations, formal proofs.
- **Deterministic reducers.** When multiple agents produce candidates, use deterministic rules (not another LLM) to select or merge. The reducer is the architecture; the agents are the search.

None of this is exotic. It's the same machinery we use to make distributed systems reliable. The only novelty is recognizing that LLM outputs need the same discipline.

## What This Means for Software Engineering

Aviation figured this out decades ago.

Cockpit software doesn't get written and shipped. It gets specified, implemented, and then *verified against the specification* — with rigor that scales with consequence. [DO-178C](https://en.wikipedia.org/wiki/DO-178C) defines five levels: from Level A (catastrophic failure, formal methods required) to Level E (no safety effect, minimal oversight). The more critical the system, the more evidence you must produce that it satisfies its invariants.

The engineers writing flight control software aren't just coders. They're specifiers, verifiers, reviewers. They define what "correct" means before anyone writes a line of code. They design the verification regime. They review the evidence that the implementation satisfies the spec.

**This is where software engineering is heading.**

If swarms can generate code — and they can — then the scarce skill is no longer "writing code." It's defining what correct code must satisfy, designing the verification that enforces it, and reviewing failures when candidates don't pass.

The engineer's role shifts from *author* to *architect of constraints*.

You don't write the HTTP client — you specify what a valid HTTP client must do, and let the swarm search for implementations that pass your checks. You don't review pull requests line by line — you design the type system, the schema, the property tests that reject invalid contributions automatically.

**This isn't diminishment. It's leverage.**

A single engineer who can precisely specify correctness criteria can oversee the output of a thousand agents. But only if the criteria are precise, and only if the verification is rigorous.

Generation is cheap. Correctness is expensive. Our job is to make that expense tractable.

---

**Continue to [Part 2: What To Actually Do About It →](part2)**
