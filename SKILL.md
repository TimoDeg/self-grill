---
name: self-grill
description: Adversarially grill CLAUDE'S OWN answer/plan/reasoning before it ships — make the reasoning explicit (explain harder), then spawn a fresh effort-escalating refute jury that KILLS by default and injects FALSE-PREMISE trap questions, looping until no new defects survive. Forces source citations on every claim (anti-hallucination), forbids hand-waving (anti-laziness), and lets isolated agents decide instead of the author self-approving. Use when an answer/plan must be right ("grill this", "are you sure", "stress-test your reasoning", "did you hallucinate that", "stop being lazy", "explain yourself harder", "review your thinking"), or before any high-stakes write/decision. NOT for grilling the USER's plan (that's a different interview pattern), NOT for routine conversational turns (too expensive), NOT for tasks with no checkable ground truth.
---

# self-grill → interrogate your own answer until it survives

The author of an answer is the worst judge of it — same context, sunk cost, confirmation bias.
This skill runs **adversarial self-critique against ground truth**: explain the reasoning out loud,
hand it to fresh agents with zero commitment to it, let them refute it to death, keep only what
survives sourced. It is an applied form of *recursive criticism-and-improvement* (RCI) with the one
ingredient RCI usually lacks — an **external objective** to improve against, not the model's own
narrative.

**Why fresh agents, not self-critique in one pass:**
- **Don't open a second chat session** — manual, no shared state, no gate, pure overhead.
- **One session + N spawned agents.** The author (the current session) is the *generator*; the
  spawned refute agents are the *decider*. A fresh agent has **no sunk cost in your wording**, so it
  actually refutes instead of rationalizing — self-critique in the same window just defends what it
  already wrote.
- Vehicle = the **Workflow** tool (deterministic fan-out + loop). No Workflow tool? Plain `Agent`
  (Task) calls in parallel work too — you lose the deterministic loop, not the core idea.

This is Karpathy's **verification asymmetry**, operationalized: finding a good answer is expensive,
*verifying* one is cheap — so spend compute on verification, not on the model narrating its own
confidence.

## Three load-bearing rules

- **Source-or-reject** (anti-hallucination). Every load-bearing claim carries a citation: a
  `file:line`, a quoted tool output, a URL, or an explicit `ASSUMPTION — unverified`. A claim with
  none is **fabricated until proven** and the jury kills it. This is the hallucination firewall.
- **KILL by default** (anti-laziness). Each refuter is prompted: *"assume this is wrong; default to
  refuted=true if uncertain."* Keep-rate is meant to be *tiny* — rejection IS the product. A grill
  that passes most claims first try is broken; hand-waving cannot survive a panel told to assume
  it's wrong.
- **Catch the false premise** (anti-sycophancy). A fraction of critics inject a **wrong question** —
  a challenge built on a planted false fact, or a leading *"you said X, so Y, right?"* where X was
  never said. The author MUST reject the false premise, not absorb it. Going along with it is a
  logged hallucination/sycophancy defect.

## When to fire
On request ("grill this", "are you sure", "stop being lazy / hallucinating", "explain harder",
"review your thinking") OR self-triggered before a high-stakes act: a decision you're about to
commit to, a number you're about to cite, a plan you're about to execute, a claim that will be hard
to walk back. Cheap conversational turns don't need it — this spends tokens on purpose.

## Procedure

**0. Scope.** One target: the answer/plan/claim-set under grill. Default = your last substantive
answer. If the user named something specific, grill that.

**1. Explain harder — build the assumption ledger (in THIS session).** Rewrite the answer as an
explicit chain: each step a numbered line `claim → why → source`. For every claim attach ONE of:
a citation (`file:line` / URL / quoted tool result) or `ASSUMPTION (unverified)`. No prose
hand-waves — if a step can't be made explicit, that's the first defect. This is the
"explain yourself harder" deliverable and the jury's input.

**2. Spawn the refute jury — effort-escalating, diverse lenses, KILL-by-default.** One lens per
refuter (diversity catches failure modes redundancy can't); cheap models first, expensive lens only
on survivors:
   - `unsourced` — which claims have no real citation? (hallucination hunt)
   - `false-premise` — **inject 1–2 wrong questions**; report whether the ledger's logic would have
     swallowed them.
   - `lazy-leap` — where did the author skip work / assert instead of check / pick the flimsy path?
   - `contradiction` — does any claim contradict another claim, a known fact, or the source it cites?
   - `overconfident` — what's stated as fact that's actually an unverified assumption?

   Workflow tool sketch (each `agent()` returns a structured `{refuted, reason, claim}` verdict):
   ```js
   phase('Verify')
   const votes = await parallel([
     () => agent(refutePrompt('unsourced'),     {effort:'low',    model:'haiku',  schema:VERDICT}),
     () => agent(refutePrompt('false-premise'),  {effort:'low',    model:'haiku',  schema:VERDICT}),
     () => agent(refutePrompt('lazy-leap'),      {effort:'medium', model:'sonnet', schema:VERDICT}),
     () => agent(refutePrompt('contradiction'),  {effort:'medium', model:'sonnet', schema:VERDICT}),
     () => agent(refutePrompt('overconfident'),  {effort:'high',   model:'opus',   schema:VERDICT}),
   ])
   // escalate ONLY survivors of the cheap+mid tiers — saves the expensive lens
   const survivesCheap = votes.filter(Boolean).filter(v => !v.refuted).length >= 3
   const judge = survivesCheap
     ? await agent(refutePrompt('final'), {effort:'xhigh', model:'opus', schema:VERDICT})
     : {refuted:true, reason:'failed cheap tier'}
   const PROMOTE = survivesCheap && !judge.refuted
   ```

**3. Answer every challenge — in THIS session.** For each refuter finding: fix the claim, add the
missing source, downgrade it to ASSUMPTION, or reject the challenge WITH a reason. For every
false-premise trap: state plainly *"premise false because …"* — never argue from the planted fact.

**4. Loop until dry.** Re-run the jury on the revised ledger. Stop when **2 consecutive rounds
surface no new real defect**, or escalate survivors to one `effort:'xhigh'` judge and promote only
if it doesn't refute. Cap rounds (default 3) — log if you hit the cap without converging; that
itself is a finding.

**5. Receipt.** Emit the result, not just the polished answer:
   - **SURVIVED** — claims now sourced + un-refuted.
   - **KILLED / REVISED** — what was wrong and why (the real output of the grill).
   - **TRAPS** — false premises injected, and whether the original reasoning caught each.
   - **STILL ASSUMPTION** — load-bearing things that remain unverified (name them; don't bury).
   - Round count + jury verdicts (e.g. `R1 1/5 → R2 4/5 → PROMOTE`).

## Anti-patterns
- Self-grilling in one head — no fresh agent = no real refutation, just defense. Spawn the panel.
- Refuters that praise — a lens that returns "looks good" did nothing; KILL-by-default or it's noise.
- Stopping at round 1 — convergence needs ≥2 quiet rounds; one pass is theatre.
- Grilling trivia — don't burn a jury on a one-line fact; this is for high-stakes answers.
- Letting the author cast the deciding vote — generator drafts, **jury decides**.
- Silent assumptions — an unsourced load-bearing claim that survives unflagged is the exact
  hallucination this skill exists to catch.

---
*Inspired by Andrej Karpathy's verification-asymmetry / autoresearch framing and the
recursive-criticism-and-improvement (RCI) literature. The core move: improve against an external
objective, not the model's own confidence.*
