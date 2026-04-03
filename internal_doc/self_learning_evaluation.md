# Self-Learning Evaluation: SKILL.md + Cost Curve Scoring

> Companion to `clawbench_v2_sandbox_architecture.md`. Defines how the v2 sandbox evaluates **self-learning** — the ability of an agent to improve across tasks.

---

## The Core Idea

Run the agent through a series of episodes in a persistent sandbox. Measure cost at each checkpoint. **A learning agent's cost goes down over time.** That's the signal.

```
Cost ($)
  |
  |  *
  |    *
  |      *  *
  |          *
  |            *  *
  |                 *
  +----------------------→ Episode
  1  2  3  4  5  ... N

Score = the downward slope. Steeper = faster learner.
```

A miner can hack a single scenario. A miner **cannot hack a downward trend** across N episodes with varying data. The trend IS the learning signal.

---

## SKILL.md: Agent-Harness-Agnostic Pack Format

Rename AGENTS.md → **SKILL.md**. A skill file is a plain markdown document that any agent framework can consume:

- Claude Code reads it as `CLAUDE.md`
- Cursor reads it as `.cursor/rules`
- OpenClaw reads it as `AGENTS.md`
- Any MCP-compatible agent reads it as context

The sandbox just places `SKILL.md` at `/workspace/SKILL.md`. The agent harness — whatever it is — reads it.

**What's in SKILL.md:**

```markdown
# SKILL.md (miner-authored)

## Instructions
[How to approach tasks, tool usage patterns, safety rules, etc.]

## Learned Patterns
[Initially empty. Agent appends here as it learns.]

## Project Context
[Accumulated knowledge about the workspace, codebase, team, etc.]
```

The miner authors the initial `Instructions` section. The `Learned Patterns` and `Project Context` sections start empty (or with seed content) and grow as the agent self-improves across episodes.

**Why SKILL.md works:**
- It's just a file. No special protocol, no API, no framework dependency.
- The agent reads it at episode start, writes to it at episode end.
- The validator doesn't need to understand the format — it just doesn't delete it between episodes.
- Miners compete on how well their SKILL.md teaches the agent to learn, not on memorizing scenarios.

---

## Evaluation Flow

```
1. Build sandbox (mock services + CLI tools)
2. Load miner's SKILL.md into /workspace/
3. For episode i = 1..N:
   a. Reset mock service data (new Tier 3 fixtures from epoch_seed + i)
   b. Deliver task prompt to agent
   c. Agent runs: reads SKILL.md → does task → updates SKILL.md
   d. Checkpoint:
      - quality: LLM judge PASS/FAIL
      - cost_i: total LLM tokens burned this episode
      - skill_md_size: file size of SKILL.md
4. Tear down sandbox
5. Score:
   - Gate: ALL episodes must PASS
   - Signal: cost curve across episodes
```

That's it. No complex per-episode scoring dimensions, no memory inspection, no transfer metrics. Just: **pass the gate, show a downward cost curve.**

---

## Why Cost Curve > Total Cost

**Total cost** can be gamed: make a minimal agent that's cheap from episode 1. No learning needed — just be terse.

**Cost curve slope** requires improvement over time:
- Flat line at low cost = cheap but not learning (no reward)
- Flat line at high cost = expensive and not learning (no reward)
- Downward slope = genuinely getting more efficient (reward)
- Steep downward slope = fast learner (maximum reward)

```
Score = -slope(cost_1, cost_2, ..., cost_N)

Higher is better. Positive score = cost going down = learning.
```

### Normalization

To prevent gaming via artificially inflated episode 1 cost:

```
learning_efficiency = (mean_cost_first_third - mean_cost_last_third) / mean_cost_first_third
```

This measures: **what fraction of initial cost did you eliminate through learning?**

A miner who starts at $0.05 and ends at $0.02 scores 0.60 (eliminated 60% of cost).
A miner who starts at $0.10 and ends at $0.02 scores 0.80 — but also burned more total.

**Combined score** (balances learning speed with absolute efficiency):

```
final_score = learning_efficiency * (1 / mean_total_cost)
```

Learn fast AND be cheap overall = win.

---

## Episode Sequence Design

Episodes should mix task types and repeat them, so the agent has opportunities to demonstrate learning on recurring patterns:

```
Episode sequence (example, N=12):

 E1:  morning_brief      (data_seed_1)    ← baseline
 E2:  inbox_triage        (data_seed_2)    ← baseline
 E3:  client_escalation   (data_seed_3)    ← baseline
 E4:  morning_brief       (data_seed_4)    ← should be cheaper than E1
 E5:  team_standup        (data_seed_5)    ← baseline
 E6:  inbox_triage        (data_seed_6)    ← should be cheaper than E2
 E7:  hiring_debrief      (data_seed_7)    ← baseline
 E8:  client_escalation   (data_seed_8)    ← should be cheaper than E3
 E9:  morning_brief       (data_seed_9)    ← should be cheapest yet
E10:  post_incident       (data_seed_10)   ← baseline
E11:  inbox_triage        (data_seed_11)   ← should be cheapest yet
E12:  client_escalation   (data_seed_12)   ← should be cheapest yet
```

**Key properties:**
- Each task type appears 2-3 times with different data
- The agent has a chance to learn from each encounter and improve on the next
- Interleaving prevents "warm cache" effects — learning must persist across different tasks
- Transfer learning is implicitly tested: learning from inbox_triage might help morning_brief

**Feedback injection (optional, between episodes):**
- After E1: "Your brief was too verbose. Bullet points, grouped by urgency."
- After E3: "You shared confidential info in Slack. Never do that."
- These corrections should show up in the cost curve as discontinuous improvements

---

## What the Validator Sees

```json
{
  "miner_uid": 42,
  "pack_hash": "abc123...",
  "episodes": [
    {"id": 1, "scenario": "morning_brief",    "pass": true, "cost": 0.052, "skill_md_lines": 45},
    {"id": 2, "scenario": "inbox_triage",      "pass": true, "cost": 0.048, "skill_md_lines": 52},
    {"id": 3, "scenario": "client_escalation", "pass": true, "cost": 0.061, "skill_md_lines": 58},
    {"id": 4, "scenario": "morning_brief",     "pass": true, "cost": 0.031, "skill_md_lines": 63},
    {"id": 5, "scenario": "team_standup",      "pass": true, "cost": 0.044, "skill_md_lines": 67},
    {"id": 6, "scenario": "inbox_triage",      "pass": true, "cost": 0.029, "skill_md_lines": 71}
  ],
  "learning_efficiency": 0.42,
  "mean_cost": 0.044,
  "final_score": 9.55,
  "qualified": true
}
```

The cost curve tells the whole story. No complex scoring rubrics needed.

---

## Sandbox Lifecycle (Minimal)

```
Container lifecycle:

  ┌─────────────────────────────────────────────┐
  │  Docker Sandbox (persistent across episodes) │
  │                                              │
  │  /workspace/SKILL.md  ← PERSISTS            │
  │  /workspace/learned/  ← PERSISTS (optional)  │
  │                                              │
  │  Mock services        ← DATA RESETS each ep  │
  │  Shell history        ← CAPTURED each ep     │
  │  Agent process        ← RESTARTS each ep     │
  └─────────────────────────────────────────────┘

Between episodes:
  1. Capture: shell transcript, cost, service state
  2. Score: LLM judge PASS/FAIL
  3. Reset: reload Tier 1 services with new fixtures
  4. Preserve: /workspace/SKILL.md, /workspace/learned/
  5. Optionally inject: user feedback file
  6. Start next episode
```

The container never stops. Only the "world" resets. The agent's brain (SKILL.md + any files it creates) persists.

---

## Why This Is Hard to Game

**Against memorization:**
- Data changes every episode (Tier 3 fixtures from different seeds)
- Same scenario type, completely different emails/tasks/people/urgency levels
- Can't hardcode "read msg_003" because msg_003 doesn't exist next time

**Against faking the cost curve:**
- Artificially inflating E1 cost then deflating E12 → caught by normalization (learning_efficiency uses thirds, not endpoints)
- Being cheap from the start → flat line, no learning signal
- Selectively learning on some episodes → interleaving makes this inconsistent
- The trend must be CONSISTENT across multiple task types with varying data

**Against over-fitting SKILL.md:**
- A SKILL.md that says "for morning_brief, always do X" will fail when the data changes
- Effective SKILL.md teaches PRINCIPLES ("prioritize by urgency", "check for confidential content") not ANSWERS

**The miner's real challenge:** Author a SKILL.md that teaches the agent to:
1. Reflect after each task
2. Identify what worked and what didn't
3. Store useful patterns (not raw data)
4. Retrieve and apply patterns in new contexts
5. All while minimizing token overhead

This is genuinely hard. It's agent engineering, not benchmark gaming.

---

## Agent Harness Agnostic

Because the interface is just "SSH into a box, read SKILL.md, do stuff," any agent framework works:

| Framework | How it reads SKILL.md | How it persists learning |
|-----------|----------------------|------------------------|
| Claude Code | Reads as CLAUDE.md (symlink or rename) | Writes to SKILL.md + files |
| Cursor | Reads as .cursor/rules | Writes to rules + files |
| OpenClaw | Reads as AGENTS.md | Writes via tool calls |
| Custom harness | `cat /workspace/SKILL.md` | Any file I/O |
| Raw LLM + bash | System prompt includes file | `echo >> SKILL.md` |

The validator doesn't care which harness runs. It only sees: cost per episode, PASS/FAIL, SKILL.md evolution.

This means miners can compete on **which agent framework learns best** — not just which prompt is cleverest. A miner running Claude Code with a well-crafted CLAUDE.md competes directly against a miner running a custom Python harness. Best learner wins.

---

## Minimal Viable Implementation

The entire self-learning evaluation system needs:

1. **Sandbox** (from v2 architecture doc): Docker + mock services + SKILL.md mount
2. **Episode runner**: Loop that resets data, delivers prompt, captures cost
3. **Cost tracker**: Count LLM tokens per episode (already tracked in v1)
4. **Quality gate**: LLM judge PASS/FAIL per episode (already built in v1)
5. **Scorer**: `learning_efficiency = (mean_first_third - mean_last_third) / mean_first_third`

That's five components. Three already exist. The new work is: sandbox + episode runner.

---

## Open Questions

1. **Number of episodes (N):** More episodes = more signal, but longer eval time. 12? 20? 8?
2. **Feedback injection:** Should user corrections be part of the standard eval, or a separate "feedback learning" scenario category?
3. **SKILL.md size limit:** Should there be a cap to prevent unbounded growth? 500 lines? 10KB?
4. **Cross-epoch learning:** Should SKILL.md persist across epochs (24h), or reset each epoch? Persisting rewards long-term learning but complicates pack versioning.
5. **Harness specification:** How does the validator know which agent harness to run? Miner specifies in pack metadata? Or we provide a standard harness and miners only control SKILL.md?
6. **Cost curve statistical significance:** With N=12 and noisy episode costs, is a linear regression slope robust enough? Or do we need more sophisticated trend detection?
