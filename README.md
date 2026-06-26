# 🔥 self-grill

**A [Claude Code](https://claude.com/claude-code) skill that makes Claude grill its *own* answers before it ships them.**

LLMs are bad judges of their own output — same context, sunk cost, confirmation bias. They defend what they just wrote instead of breaking it. `self-grill` fixes that: it makes Claude **explain its reasoning out loud**, then hands that reasoning to a panel of **fresh agents told to assume it's wrong** and prove it. Only claims that survive — *with sources* — make it through.

It's a practical anti-hallucination / anti-laziness pass: every claim needs a citation, hand-waving gets killed by default, and a few critics throw **deliberately false-premise questions** to catch sycophancy.

---

## What it does

```
                          YOU: "/self-grill <answer or claim>"
                                        │
                                        ▼
        ┌───────────────────────────────────────────────────────────┐
        │  1. EXPLAIN HARDER                                         │
        │     Rewrite the answer as an explicit ledger:             │
        │     claim → why → source   (no prose hand-waves)          │
        └───────────────────────────────┬───────────────────────────┘
                                         │ ledger
                                         ▼
        ┌───────────────────────────────────────────────────────────┐
        │  2. REFUTE JURY  (fresh agents, KILL by default)          │
        │                                                           │
        │   unsourced   false-premise   lazy-leap                   │
        │   contradiction   overconfident                          │
        │      haiku ── haiku ── sonnet ── sonnet ── opus           │
        │       (cheap lenses first, expensive lens on survivors)   │
        └───────────────────────────────┬───────────────────────────┘
                                         │ findings + planted traps
                                         ▼
        ┌───────────────────────────────────────────────────────────┐
        │  3. ANSWER BACK  → fix / cite / downgrade / reject-w-reason │
        │     (catch every false premise — don't absorb it)         │
        └───────────────────────────────┬───────────────────────────┘
                                         │
                          ┌──────────────┴──────────────┐
                          │  new defects found?         │
                          │    yes → loop to step 2     │  (max 3 rounds)
                          │    no (2x) → xhigh judge     │
                          └──────────────┬──────────────┘
                                         ▼
        ┌───────────────────────────────────────────────────────────┐
        │  RECEIPT:  SURVIVED · KILLED/REVISED · TRAPS · ASSUMPTIONS  │
        └───────────────────────────────────────────────────────────┘
```

**Three rules that make it work:**

| Rule | What it kills |
|------|---------------|
| **Source-or-reject** — every claim needs a `file:line` / URL / tool-output, or it's flagged `ASSUMPTION` | hallucination |
| **KILL by default** — refuters are told "assume it's wrong"; ~97% of hand-waves don't survive | laziness |
| **Catch the false premise** — critics plant wrong facts in their questions; Claude must reject them | sycophancy |

**Generator ≠ decider:** the session that wrote the answer never gets the deciding vote. Fresh agents with no stake in the wording do. That's the whole trick — and why it's *agents*, not a second chat session.

---

## Install

`self-grill` is a single `SKILL.md` file. Drop it where Claude Code looks for skills.

### Global (every project on your machine)

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/self-grill
curl -fsSL https://raw.githubusercontent.com/TimoDeg/self-grill/main/SKILL.md \
  -o ~/.claude/skills/self-grill/SKILL.md
```

```powershell
# Windows (PowerShell)
$dir = "$env:USERPROFILE\.claude\skills\self-grill"
New-Item -ItemType Directory -Force $dir | Out-Null
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/TimoDeg/self-grill/main/SKILL.md" `
  -OutFile "$dir\SKILL.md"
```

### Per-project (one repo only)

```bash
mkdir -p .claude/skills/self-grill
curl -fsSL https://raw.githubusercontent.com/TimoDeg/self-grill/main/SKILL.md \
  -o .claude/skills/self-grill/SKILL.md
```

Restart Claude Code (or start a new session) so it picks up the skill.

---

## Use

Type the slash command, optionally naming what to grill:

```
/self-grill
/self-grill the migration plan you just gave me
```

Or just say any of the trigger phrases mid-conversation — Claude will reach for it:

> "are you sure about that?" · "stop being lazy" · "did you hallucinate that number?"
> "stress-test your reasoning" · "explain yourself harder" · "review your thinking"

### Example receipt

```
RECEIPT — /self-grill on the deploy plan.  1 round (5 lenses) → xhigh judge.
Verdict: SURVIVES_REVISED (after 3 fixes).

KILLED / REVISED
  #1 major  "rollback is instant"  → wrong: migration is non-reversible, cited no source
  #2 minor  "p99 < 200ms"          → unverified ASSUMPTION, no benchmark behind it
  #3 minor  step 4 cited the wrong config file

TRAPS — both planted false premises CAUGHT (not swallowed)

STILL ASSUMPTION (named, not buried)
  - load test was run on staging, not prod-sized data
```

The point isn't the polished answer — it's the **KILLED list**. That's the bugs you'd have shipped.

---

## Requirements

- **[Claude Code](https://claude.com/claude-code)** — this is a Claude Code skill.
- The refute jury uses Claude Code's **Workflow** tool for deterministic fan-out + looping. If your
  setup doesn't have it, the skill falls back to parallel **Agent (Task)** calls — same idea, you
  just lose the deterministic loop.
- Spawning a multi-agent jury costs tokens on purpose. Fire it on answers that matter, not every turn.

---

## Why it works (the one idea)

Andrej Karpathy's **verification asymmetry**: *finding* a good answer is expensive, but *verifying*
one is cheap. So don't spend compute on the model narrating its own confidence — spend it on
adversarial verification against an external objective. Recursive self-critique only helps when it
improves against **ground truth** (sources, checkable facts), not against the model's own narrative.
`self-grill` is that idea wired into a repeatable pass.

## License

[MIT](LICENSE) — use it, fork it, ship it.
