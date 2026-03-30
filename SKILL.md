---
description: >
  Cross-LLM debate. Orchestrates real discussions between different AI models
  (Claude, Codex, Gemini, Ollama, etc.) — each with its own training, biases,
  and reasoning style. The disagreements ARE the insight.
  Use when asked to "debate", "get opinions from different models",
  "what do codex/gemini/ollama think", "cross-model discussion".
---

# Crossfire: Cross-LLM Debate

Different models, different weights, different blind spots. Put them in a room and see what happens.

---

## Input

**$ARGUMENTS:** Topic + participants in natural language.

Examples:
- `Kafka vs Redpanda for event streaming, ask codex and gemini`
- `best RAG architecture, get opinions from ollama llama3 and codex`
- `is event sourcing worth it? use gemini and claude opus`

If no participants mentioned, default to: claude:sonnet + codex.

---

## CLI Reference

Optimized for debate: non-persistent sessions, no sandbox overhead, clean text output, no hooks/plugins. Claude picks the right flags automatically.

| CLI | Command | Why these flags |
|---|---|---|
| `codex` | `npx @openai/codex exec --ephemeral --skip-git-repo-check --full-auto -m gpt-5.4 '{prompt}'` | `--ephemeral`: no session on disk. `--skip-git-repo-check`: runs anywhere. `--full-auto`: auto-approve tools (bash, web, files) in sandboxed mode. `-m gpt-5.4`: best reasoning model available |
| `gemini` | `npx @google/gemini-cli -m gemini-2.5-pro --yolo -p '{prompt}'` | `-p`: headless. `-m gemini-2.5-pro`: best Gemini model. `--yolo`: auto-approve all tool calls (web search, bash, file ops) |
| `ollama {model}` | `ollama run --experimental {model} '{prompt}'` | `--experimental`: enables agent loop with tools (bash, web search). Default to `qwen3:4b` if user says just "ollama". Already non-persistent |
| `claude` (external) | `claude --no-session-persistence --dangerously-skip-permissions --model sonnet -p '{prompt}' --output-format text` | `--no-session-persistence`: no disk writes. `--dangerously-skip-permissions`: auto-approve all tools. `--model sonnet`: best speed/quality. Use `--model opus` for max depth. NOTE: don't use `--bare` — it blocks OAuth/keychain auth |
| anything else | `npx {name} '{prompt}'` | best guess fallback |

**Availability check:** Before launching, verify each CLI exists. If missing via npx, it auto-installs. If auth fails (no API key), drop that participant and continue. Minimum 2 participants required.

**Tool access:** All CLIs are configured for autonomous tool use. Agents can and should use any tool available to them: web search to verify claims, bash to run commands, git clone to study repos, file reads to check docs. The debate is richer when agents ground their arguments in evidence they found themselves, not just their training data.

---

## Discussion File

Every debate produces an append-only markdown file. This is the artifact — auditable, shareable, git-committable.

### File structure

```
agora-YYYY-MM-DD-HHMMSS-{topic-slug}/
├── 00-setup.md                    # Topic, participants, personas (orchestrator)
├── 01-scatter-{participant}.md    # Each agent writes its own SCATTER response
├── 02-clash-{N}-{participant}.md  # Each agent writes its own CLASH response per round
├── 03-assessment-{N}.md          # Orchestrator's tension tracking per round
├── verdict.md                     # Final synthesis (orchestrator)
└── transcript.md                  # Full concatenated log (generated at end)
```

**Each agent writes its own file.** The orchestrator never writes on behalf of a participant. This means:
- Responses are unfiltered and unmodified — what the model wrote is what you read
- Each file is attributed to a specific model — provenance is clear
- The orchestrator only writes setup, assessments, and verdict

Write each file as you complete each phase.

### 00-setup.md format

```markdown
# Crossfire: {topic}

Created: {ISO 8601}
Participants: {list}
Status: in-progress

## Personas

### {Participant} — {Role}
- Domain: ...
- Worldview: ...
- Methods: ...
- Blind spots: ...
- Style: ...
```

### Append-only rule

Never delete or rewrite earlier files. If a position changes, the new file documents the change. The history IS the value.

---

## Phase 0: PARSE

Extract from natural language:

1. **Topic**: the question or subject to debate
2. **Participants**: model names mentioned

Show the parsed setup before launching:

```
TOPIC: {extracted topic}
PARTICIPANTS: {list with routing}
PERSONAS: {assigned — see Phase 0.5}
```

---

## Phase 0.5: PERSONA CRAFTING

Before any participant speaks, Claude the orchestrator creates a distinct persona for each one. The persona anchors a cognitive mode — it's not decoration, it shapes how the agent reasons.

### Why personas matter

Same prompt + same topic → same-ish answers. A persona forces a different entry point into the problem. When Codex thinks as a "battle-scarred SRE who's been paged at 3am by bad architectural decisions", it generates different arguments than when it thinks as a generic assistant.

### Persona design

For each participant, Claude crafts:

```
PERSONA: {name} — {role}
├── DOMAIN: area of expertise, what they know cold
├── WORLDVIEW: beliefs, assumptions, non-negotiables
│   (e.g. "simplicity beats correctness until proven otherwise")
├── METHODS: how they reason, what counts as evidence for them
│   (e.g. "trusts production metrics over theoretical arguments")
├── BLIND SPOTS: what they'll naturally miss or undervalue
│   (e.g. "dismisses long-term maintenance cost")
├── POSITION SEED: initial lean (FOR / AGAINST / CONDITIONAL / SKEPTICAL)
└── STYLE: how they communicate
    (e.g. "terse, example-heavy, allergic to hand-waving")
```

### Persona assignment principles

1. **Complementary, not variations.** A "cautious architect" and a "careful engineer" are the same person. A "cautious architect" and a "move-fast founder" create friction.
2. **Exploit model strengths.** Codex trained on code → give it the practitioner role. Gemini with broad web training → give it the ecosystem/trends role. Ollama local model → give it the contrarian/skeptic role (less RLHF, rawer opinions).
3. **At least one adversarial.** Every panel needs someone whose instinct is to poke holes.
4. **Worldviews must conflict.** If all personas value the same things, the debate collapses. Ensure at least one fundamental tension in worldviews.
5. **Grounded in the topic.** A debate about database choice gets a DBA, a product engineer, and an ops person — not generic "expert A" and "expert B".

### Injection

Personas are written to `00-setup.md` once. Agents read their persona from this file — no need to paste the full persona into every prompt. Since all CLIs have file read access, the prompt just points them to the file:

```
Read your persona from {debate_dir}/00-setup.md — you are {participant name}.
Stay in character. Your worldview and methods shape HOW you argue,
not just WHAT you argue. If your persona would dismiss a point,
dismiss it. If your persona would demand evidence, demand it.

You have full tool access: web search, bash, file reads, git clone.
You MUST use web search or file reads at least twice to verify or
support your claims. Arguments without evidence will be challenged.
Do NOT rely solely on your training data — check the actual docs,
repos, or data before making factual claims.

IMPORTANT: You are ONLY {participant}. Do NOT write responses for
other participants. Write ONLY your own file.

Topic: {topic}
Round: {N}

{round-specific instructions}
```

Write personas to `00-setup.md` before launching SCATTER.

---

## Phase 1: SCATTER (parallel)

All participants answer simultaneously. Each gets the topic + their persona + the instruction to stake an initial position AND write it to a file.

Claude sub-agents via Agent tool. External LLMs via Bash. All in parallel.

### Prompt structure (per participant)

Each agent is instructed to read its persona from `00-setup.md` and write its response to its own file:

```
Read your persona from {debate_dir}/00-setup.md — you are {participant name}.
Stay in character. Use any tools you want (web search, bash, git clone, file reads)
to ground your arguments in evidence.

IMPORTANT: You are ONLY {participant}. Do NOT write files for other participants.

This is the SCATTER phase. You're staking your initial position
independently — you haven't seen anyone else's take yet.

YOUR TASK: Write your response to the file {debate_dir}/01-scatter-{participant}.md
After writing, confirm the file exists. If you cannot write it, output your response to stdout.

Use this exact format:

# SCATTER — {participant} ({persona role})

## Position
[one sentence — your stance]

## Arguments
[3-5 key points, grounded in your persona's methods]

## Assumptions
[what you're taking for granted]

## Weakness
[strongest argument against your own position — be honest]
```

### Self-authored files

Each agent writes its own markdown file. For external CLIs, this works because all CLIs are configured with tool access (file writes enabled via `--full-auto`, `--yolo`, `--dangerously-skip-permissions`). For Claude sub-agents, they use the Write tool natively.

If an external CLI fails to write the file (tool access issue, path error), fall back to capturing stdout and writing the file on the agent's behalf — but note this in the file header.

**After SCATTER:** Read all `01-scatter-*.md` files. Display positions side by side to user.

---

## Phase 2: CLASH (sequential, iterative)

### 2.1 Tension identification

Claude reads all SCATTER responses and identifies **tensions**: genuine disagreements in reasoning or conclusions. Not wording differences — substantive conflicts rooted in different worldviews or evidence.

```
TENSIONS IDENTIFIED:
1. {tension}: {participant A's persona} says X because ..., {participant B's persona} says Y because ...
2. ...
```

If zero tensions → skip to Phase 3 (note the unusual consensus).

### 2.2 Targeted rounds

Each participant reads the other's files directly and reacts. The orchestrator provides minimal framing — the agent does its own analysis.

Prompt per participant:

```
Read your persona from {debate_dir}/00-setup.md — you are {participant name}.

This is CLASH round {N}. Read the other participant's previous response(s):
{list of files to read, e.g. "01-scatter-codex.md" or "02-clash-1-codex.md"}

React to what you read. Challenge, concede, or refine. Stay in character.
Use any tools (web search, bash, git clone) to verify or counter their claims.

You are ONLY {participant}. Write ONLY your file:
{debate_dir}/02-clash-{N}-{participant}.md

Use this format:

# CLASH Round {N} — {participant} ({persona role})

## Response to {other participant}
{react to their actual arguments — challenge, concede, or refine}

## Updated position
{where you stand now, after considering the other side}

## New evidence or angle
{something not yet discussed, or "none — positions are stabilizing"}
```

The orchestrator does NOT summarize tensions or pre-digest positions. The agent reads the raw files and forms its own view. This produces more authentic disagreement — the agent reacts to what was actually written, not to an orchestrator's interpretation.

### 2.3 Position tracking

After each round, the **orchestrator** reads all `02-clash-N-*.md` files and writes `03-assessment-N.md`:

```markdown
# Assessment — Round {N}

- Position shifts: {who moved, how}
- New arguments: {any?}
- Remaining tensions: {list}
- Convergence: CONVERGING | DIVERGING | DEADLOCKED
```

### 2.4 Convergence detection

**Minimum 3 rounds.** Always. Even if positions seem aligned after round 1, run at least 3 rounds — early agreement is often shallow, and forced continued debate surfaces hidden assumptions and cracks.

After round 3, stop when ALL true:
- No participant changed position since last round
- No new arguments surfaced
- Remaining tensions produce diminishing returns

If not converging after round 3, keep going until convergence or deadlock. Safety cap: 7 rounds.

### 2.5 Mode switch: adversarial

If a tension survives 2+ rounds without movement:

- Assign one participant as ATTACKER (persona shifts to "find every flaw")
- Assign another as DEFENDER (persona shifts to "address every attack")
- One round, then Claude judges

---

## Phase 3: VERDICT (adaptive)

Output depth scales with debate richness. Write `verdict.md`.

### Fast convergence (1-2 CLASH rounds or consensus from SCATTER)

```markdown
# Crossfire Verdict: {topic}
Participants: {list w/ personas} | Rounds: {N} | Converged quickly

## Consensus
{what everyone landed on}

## Minor differences
{if any}

## Verdict
{recommendation}
```

### Rich debate (3+ rounds or surviving tensions)

```markdown
# Crossfire Verdict: {topic}
Participants: {list w/ personas} | Rounds: {N} | Status: {converged | max-rounds}

## Final positions
### {Participant} ({persona role}) — {one-line evolved stance}
{summary}

## Resolved tensions
| Tension | Resolution | Round |

## Surviving tensions
| Tension | Side A | Side B | Why unresolved |

## Key exchanges
{2-3 most revealing moments — quotes or paraphrases}

## Coalition map (3+ participants)
{Who aligned with whom, and why? A 2-vs-1 pattern where two models
converge independently is stronger signal than any single opinion.
Name the coalition, what united them, and what isolated the dissenter.}

## What the disagreement reveals
{Meta-insight: what does it mean that THESE specific models,
with THESE personas, disagreed on THIS? The inter-model divergence
is signal — name what it reveals about the problem space.}

## Verdict
{recommendation, acknowledging surviving tensions}
```

### Transcript generation

After writing `verdict.md`, concatenate all files into `transcript.md` for a single-file audit trail.

---

## Edge cases

| Situation | Response |
|---|---|
| CLI not installed | Drop participant, note it, continue if 2+ remain |
| Immediate consensus | Short verdict, flag the unusual agreement |
| Garbage response | Re-prompt once. If still bad, exclude with note |
| One model dominates | Flag it — might mean stronger training on this topic, or a weak persona assignment |
| External CLI error | Retry once, then drop with note |
| Only 1 participant available | Abort. Suggest alternatives |
| Persona not anchoring | If a model ignores its persona and responds generically, re-prompt with stronger framing |
| Agent plays both roles | Some models (notably Gemini) may simulate the entire debate solo — writing files for both sides. Add explicit instruction: "You are ONLY {participant}. Do NOT write responses for other participants. Write ONLY your own file." Delete any files written for other roles |

---

## Rules

1. **Parse, don't flag-parse** — extract participants from natural language, no `--flags`
2. **Persona-first** — every participant gets a crafted persona before speaking. No generic "Assistant" voices
3. **File-based** — every phase writes to disk. The discussion folder is the artifact
4. **Parallel SCATTER** — single message, all Agent + Bash calls at once
5. **Agents read each other's files** — CLASH agents read raw files from other participants, no pre-digested summaries
6. **Orchestrator stays out** — the orchestrator identifies tensions in assessments but does NOT rewrite or filter participant output
7. **Convergence stops the loop** — stop when positions freeze, don't run rounds for ceremony
8. **Adaptive output** — short debates get short reports, rich debates get rich reports
9. **Meta-insight required** — every VERDICT explains what the inter-model divergence means
10. **Persona travels with every call** — stateless CLIs get the full persona re-injected each round
11. **Append-only** — never rewrite earlier files. History is the value
12. **Minimum 2 participants** — a debate needs at least two voices
