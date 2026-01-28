---
title: "Part 2: What To Actually Do About It"
description: "Actionable approaches for making AI swarms productive within deterministic constraints"
---

# Part 2: What To Actually Do About It

*A follow-up to [Part 1: "Why AI Swarms Cannot Build Architecture"](https://jsulmont.github.io/swarms-ai/)*

---

## Introduction

Part 1 made the case: swarms are local, parallel, stateless. Architecture requires global, sequential, persistent. The structural mismatch is fundamental.

This part explores what we can actually do. Not theoretical fixes — actionable approaches that exist today.

---

## The Aviation Way

Aviation figured this out decades ago. The answer: **model-based development**.

You don't write code. You write models. The model is the source of truth. Code is generated — mechanically, deterministically, from a qualified generator.

Tools:
- **[SCADE](https://www.ansys.com/products/embedded-software/ansys-scade-suite)** — synchronous dataflow, generates certified C
- **[Simulink](https://www.mathworks.com/products/simulink.html)** — model-based design, Embedded Coder generates C/C++
- **[SPARK](https://www.adacore.com/about-spark)** — Ada subset with contracts, proven at compile time
- **[TLA+](https://lamport.azurewebsites.net/tla/tla.html)** — specification language for distributed systems, used at AWS for S3, DynamoDB, EBS

For the extreme end — verified compilers, verified kernels — there's [Coq](https://coq.inria.fr/), [Isabelle](https://isabelle.in.tum.de/), [Lean](https://leanprover.github.io/). These require hand-written proofs, but they've produced CompCert (a verified C compiler) and seL4 (a verified microkernel). The cost is high; so is the assurance.

In cryptography: [F\*](https://www.fstar-lang.org/) and [HACL\*](https://hacl-star.github.io/) — verified crypto now shipping in Firefox and the Linux kernel.

The code generator is qualified (DO-178C). "Qualified" means the tool is trusted not to introduce additional errors — if the model is correct and the generator is qualified, the generated code won't add new defects. You verify the model, not the generated code. This is the key insight: push verification upstream, let the machine handle the rest.

But qualified codegen is not a closed system. You still need:
- Requirements validation (did we specify the right thing?)
- Model review (does the model match the requirements?)
- Integration testing (do the components work together?)
- System-level verification (does the whole thing behave correctly?)

Qualification reduces error surface. It does not eliminate requirement errors or model errors. If your specification is wrong, the generator will faithfully produce wrong code.

This works anyway. Airbus flight controls. NASA rovers. Nuclear plant controllers. It's not research — it's deployed, trusted, tested by decades of flight hours. But it works because the entire pipeline is disciplined, not because codegen is magic.

The insight: **raise the abstraction level**. Verify intent, generate implementation.

---

## Our Constraints Aren't Theirs

Aviation operates under DAL-A: failure means loss of aircraft. The cost of formal verification is justified.

Most software isn't DAL-A. It's not even close.

| Level | Consequence | Reality |
|-------|-------------|---------|
| A | Catastrophic | Flight controls, nuclear |
| B | Hazardous | Medical devices, rail |
| C | Major | Payment systems, maybe |
| D | Minor | Most SaaS |
| E | No effect | Internal tools, prototypes |

Full formal methods for a CRUD app? Overkill. The economics don't work.

But "overkill for CRUD" doesn't mean "useless." It means we need to pick the right tool for the consequence level.

---

## Triangulation

For DAL-C through E, we need something lighter than proofs but stronger than "the LLM said it works."

The idea: [property-based testing](https://en.wikipedia.org/wiki/Property_testing). Instead of example-based tests ("given input X, expect output Y"), you define invariants ("for all inputs, this property holds") and the framework generates thousands of random inputs to falsify them.

The problem: if the LLM generates both code and properties, it tests its own assumptions. Properties pass, but prove nothing.

The fix: **triangulate from a fixed point**.

```
            [Human-verified Contract]
                   (frozen)
                  /        \
                 v          v
          [Code Gen]    [Property Gen]
          (Model A)      (Model B)
               |              |
               v              v
            [SUT]  <--test--> [Properties]
                      |
                      v
              [Adjudicator]
```

The contract is human-approved and versioned. Code and properties are generated from that anchor using *diverse* witnesses — different model families, different prompting strategies, different formalisms. If they disagree, adjudicate against the contract.

True independence is impossible — models share training data. The goal is *diversity*, not independence. More diverse witnesses → harder for a single failure mode to slip through.

Diversity tactics:
- Different model families (Claude vs. GPT vs. Gemini)
- Different prompting strategies (chain-of-thought vs. direct)
- Different testing strategies (property-based, unit tests, coverage-guided fuzzing)

This isn't proof. It's statistical confidence from diverse witnesses. The strength scales with effort. 100 random inputs? Weak signal. 1M inputs with coverage-guided fuzzing? Stronger. You're buying confidence, not certainty (and the cost is measurable in tokens).

Good enough? Depends on your DAL level. For most software, yes.

Tooling exists: Rust has [`proptest`](https://github.com/proptest-rs/proptest) for property-based testing and [`cargo-fuzz`](https://github.com/rust-fuzz/cargo-fuzz) for coverage-guided fuzzing. Other ecosystems have equivalents.

---

## The Contract Bottleneck

Triangulation assumes you can write a good contract. This is not free.

**Contracts that work:**
- Type signatures (cheap, mechanical)
- API schemas (OpenAPI, protobuf)
- State machine diagrams (finite, enumerable)
- Algebraic properties (`for all x, decode(encode(x)) == x`)
- Database constraints (unique, foreign key, check)

**Contracts that don't work:**
- Natural language specs ("should be fast")
- Implicit requirements ("users expect...")
- Emergent behavior ("the system should feel responsive")
- Aesthetic judgments ("clean code")

The scope of triangulation is limited to what you can formalize. For most CRUD apps, that's actually a lot — schemas, types, invariants, state machines. For UX, product feel, "it just works"? You're back to human judgment.

This isn't a weakness to hide. It's a boundary to respect.

---

## Risk Transfer, Not Risk Elimination

Triangulation shifts risk. It does not remove it.

| Before | After |
|--------|-------|
| Implementation risk | Specification risk |
| "Did we build it right?" | "Did we specify the right thing?" |
| Bugs in code | Bugs in contracts |

If the contract is wrong, the entire system is *perfectly* wrong. Code and properties will agree. Tests will pass. The system will do exactly what the contract says — which isn't what you needed.

In safety engineering, this is called **specification risk**, and it dominates once implementation risk is reduced. The more you trust your generators and testers, the more your risk concentrates in the spec.

This is not a reason to avoid triangulation. It's a reason to:
- Treat contracts as critical artifacts (review, version, test the tests)
- Invest in contract validation (domain experts, formal review, prototype testing)
- Accept that some errors will be specification errors, not implementation errors

Worse: **correlated spec-generator failure**. The human writes a contract that aligns with the LLM's blind spot. Generated code satisfies the contract. Generated properties pass. The system is wrong, but no signal fires.

This failure mode is well-documented. [Knight & Leveson (1986)](https://ieeexplore.ieee.org/document/1702161) showed that independently developed software versions failed on the same inputs far more often than statistical independence would predict — developers made similar mistakes at similar decision points. LLMs trained on similar corpora have the same problem, possibly worse.

The mitigation is adversarial review — someone tasked with challenging the spec itself, not just verifying implementation against it.

Triangulation makes implementation cheap and specification expensive. Cost moves; it doesn't disappear.

---

## Adjudication

When code and properties disagree, who decides?

**Not an LLM.** That's the judge fallacy.

Options:

1. **Type checker / schema validator** — if the contract is machine-checkable, use it. This is the best case: disagreement is a compile error.

2. **Human escalation** — flag the disagreement, human decides, decision gets logged. Yes, this reintroduces humans. That's the point. Humans verify contracts and adjudicate disputes; machines generate and test. The ratio shifts, not the roles.

3. **Fail closed** — disagreement = rejection, regenerate both. Expensive but safe. Use for high-consequence code paths.

The adjudicator must be deterministic or human. Never stochastic.

---

## Contract Evolution

Contracts change. How do you update without invalidating everything?

**Versioned contracts.** Each contract has an ID and version. Generated code and properties reference the version they were built against.

**Migration protocol:**
1. Draft new contract version
2. Generate code and properties against new version
3. Run old properties against new code (regression check)
4. Run new properties against old code (gap analysis)
5. Human approves migration
6. Freeze new version, deprecate old

This is heavyweight. Use it for core invariants (auth, payments, data integrity), not every helper function. Most code doesn't need versioned contracts — it needs types and tests.

---

## Semantic Drift

Contract versioning handles explicit changes. But there's a subtler failure mode: **the contract stays the same while its meaning shifts**.

Over months or years, teams slowly reinterpret:
- "User" (account? person? session?)
- "Valid" (syntactically? semantically? contextually?)
- "Authorized" (authenticated? permitted? rate-limited?)
- "Consistent" (eventually? strongly? causally?)

The words don't change. The types don't change. The tests still pass. But the system rots because humans, LLMs, and new team members interpret the same terms differently.

This is a known failure mode in long-lived formal systems. The spec is correct; the understanding diverges.

**Mitigations:**

1. **Canonical definitions** — a glossary of terms with precise meanings, versioned alongside contracts. Not prose; structured definitions that can be referenced.

```yaml
terms:
  user:
    definition: "An authenticated account with a verified email"
    not: "A session, a person, or an anonymous visitor"
    see_also: [account, session, visitor]
```

2. **Semantic locking** — key terms get formal definitions (types, schemas, ontologies) that constrain interpretation. If "User" is a type with specific fields, drift is bounded by the type system.

3. **Drift detection** — periodic audits where you ask: "What does X mean?" to humans, LLMs, and documentation. If answers diverge, you have drift. Catch it before it causes bugs.

Semantic drift is slow and invisible. By the time you notice, the damage is distributed across the codebase. The only defense is active maintenance of meaning.

---

## Example: User Registration

**Contract (human-verified):**
```
register(email: Email, password: String) -> Result<UserId, RegistrationError>

Invariants:
- Emails are unique (enforced by DB constraint)
- Passwords are hashed with bcrypt, cost >= 12
- Returns RegistrationError::EmailTaken if email exists
- Returns RegistrationError::WeakPassword if password < 8 chars
```

**Generated code:** (Claude)
```rust
pub fn register(email: Email, password: &str) -> Result<UserId, RegistrationError> {
    if password.len() < 8 {
        return Err(RegistrationError::WeakPassword);
    }
    let hash = bcrypt::hash(password, 12)?;
    let id = db::insert_user(&email, &hash)?;
    Ok(id)
}
```

**Generated properties:** (GPT-4)
```rust
proptest! {
    #[test]
    fn prop_email_uniqueness(email in valid_email(), pw in valid_password()) {
        let _ = register(email.clone(), &pw);
        let result = register(email, &pw);
        prop_assert_eq!(result, Err(RegistrationError::EmailTaken));
    }

    #[test]
    fn prop_password_not_stored(email in valid_email(), pw in valid_password()) {
        let id = register(email, &pw).unwrap();
        let user = db::get_user(id).unwrap();
        prop_assert!(!user.password_hash.contains(&pw));
    }

    #[test]
    fn prop_hash_verifiable(email in valid_email(), pw in valid_password()) {
        let id = register(email, &pw).unwrap();
        let user = db::get_user(id).unwrap();
        prop_assert!(bcrypt::verify(&pw, &user.password_hash).unwrap());
    }

    #[test]
    fn prop_weak_password_rejected(email in valid_email(), pw in ".{0,7}") {
        let result = register(email, &pw);
        prop_assert_eq!(result, Err(RegistrationError::WeakPassword));
    }
}
```

**Test run:** 10,000 random emails/passwords. One failure: code returned generic `DbError` instead of `EmailTaken` on duplicate insert. Fixed by matching on the DB error code. Properties passed.

**Cost:** 2 hours human time (contract + review). Machine time: negligible.

---

## Formalizability Spectrum

The registration example is the easy case. Discrete domain, crisp invariants, small state space, simple oracles. Not everything is this clean.

**Fully formalizable** — triangulation works well:
- CRUD operations
- Parsers, serializers
- Authentication flows
- Schema migrations
- Pure algorithms

Contracts are crisp. Properties are checkable. Disagreements are adjudicable.

**Partially formalizable** — triangulation helps but degrades:
- Trading systems (some rules formal, some heuristic)
- Caches (correctness is formal, eviction policy is statistical)
- Distributed schedulers (safety properties formal, performance emergent)
- Consensus implementations (protocol formal, timeout tuning empirical)
- Control loops (stability provable, tuning experimental)

Here, contracts become leaky. You can formalize the safety envelope but not the optimization target. Triangulation catches violations of hard constraints; it won't tell you if your heuristics are good.

**Essentially informal** — triangulation doesn't apply:
- UX feel
- "Responsive" systems
- Aesthetic judgments
- Product-market fit

No contract captures these. You're back to human judgment, user testing, iteration.

Most real systems are mixed. The auth module is fully formalizable; the recommendation engine is partially; the onboarding flow is essentially informal. Apply triangulation where it fits. Don't pretend it covers everything.

---

## Architectural Memory

Even with perfect contracts, swarms fail. Why?

Multiple agents make incompatible local decisions. One picks `reqwest`, another picks `hyper`. Same contract, different HTTP clients. The code doesn't integrate.

This isn't optional. It's structural. Without shared architectural memory, no swarm converges — even with perfect contracts, perfect triangulation, perfect adjudication.

**Decision Registry**: a versioned log of architectural choices.

```
HTTP_CLIENT=reqwest
DATE_LIBRARY=chrono
ERROR_HANDLING=thiserror
ASYNC_RUNTIME=tokio
ID_FORMAT=uuid_v7
```

Agents query before choosing. If no entry exists, they escalate (to human, not to another LLM). Human decides, registry updates, all future agents inherit the decision.

This is logically equivalent to an [Architecture Decision Record](https://adr.github.io/) (ADR) system — but machine-readable and enforceable.

The registry captures:
- **Incidental choices** (which library, which format)
- **Architectural patterns** (how we do error handling, how we structure modules)
- **Constraints** (what we've decided *not* to do)

Without this:
- Agents re-litigate the same choices
- Coherence erodes over time
- Integration becomes the bottleneck

Contracts define *what*. The registry defines *how*. Both are required.

Implementation: could be as simple as a TOML file in the repo, validated by a pre-commit hook. The mechanism matters less than the discipline.

---

## What Doesn't Work

Some ideas sound good but fail structurally.

**The Self-Grading Fallacy**

LLM generates code. Same LLM generates tests. Tests pass. Ship it?

No. The tests encode the LLM's understanding, including its misconceptions. Passing tests prove consistency with the model's beliefs, not correctness.

**The Smarter Model Fallacy**

"GPT-4 makes mistakes, but GPT-5 will be better."

Better at what? Whether smarter models converge or diverge is an open question. They might converge on canonical solutions more often — or they might have more ways to solve problems, more stylistic range, more "creativity."

But even if they converge, they converge on *their* answer, not *your* requirements. A model that confidently produces the same wrong answer every time is not better than one that varies. The problem isn't intelligence. It's coordination. Smarter agents with no shared state still can't maintain invariants across a codebase.

**The Judge Fallacy**

"Have one LLM review another LLM's output."

Escalating stochastic to stochastic doesn't help. The judge has the same structural limitations: local context, no persistent memory of past decisions, no enforcement authority.

A judge only works if it's deterministic: a type checker, a schema validator, a constraint solver. But then it's not a "judge" — it's a compiler. We already have those.

---

## Summary

| Approach | Mechanism | Good For |
|----------|-----------|----------|
| Formal (aviation) | Proofs, qualified codegen | DAL-A/B, catastrophic failure domains |
| Triangulation | Diverse witnesses, properties | DAL-C/D/E, fully/partially formalizable domains |
| Adjudication | Deterministic checkers, human escalation | Resolving disagreements |
| Architectural memory | Decision registry, ADRs | Maintaining coherence across agents |
| Contract evolution | Versioned migration | Handling change over time |
| Semantic maintenance | Canonical definitions, drift detection | Preserving meaning over time |

The through-line: **stochastic generators need deterministic shells**.

What this actually means:

> AI explores the implementation space inside human-defined, machine-enforced constraint surfaces.

This is not "AI writes software." This is constraint-oriented engineering.

The engineer becomes:
- A **spec designer** — defining what, not how
- A **boundary setter** — constraining the solution space
- A **risk allocator** — choosing where to invest verification effort
- A **semantic authority** — maintaining meaning over time

Let swarms generate. But anchor them to human-verified contracts, enforce with proofs or properties, adjudicate with deterministic checkers, coordinate with registries, and actively maintain the meaning of terms.

The constraint surface is smaller than the code surface. That's the leverage.

The economic condition is explicit: **this only pays off when the invariant surface is smaller than the code surface**. If specifying the contract is as hard as writing the code, you've gained nothing. The approach works because most software has high code-to-contract ratios — many lines of implementation per line of specification. Where that ratio inverts (novel algorithms, research code, one-off scripts), write the code directly.

What triangulation buys you: machines do the generation and testing; humans do the contracts and adjudication. The ratio of human to machine effort shifts dramatically. But humans don't leave the loop — they move upstream.

This won't make swarms "architects." But it can make them productive, bounded, and subordinate. Which is the only form in which they will ever be safe.

