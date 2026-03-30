# agents-crossfire

Make your AI models argue with each other so you get a better answer.

Instead of asking one model and hoping it's right, crossfire puts Claude, Codex, Gemini, and Ollama in the same room with conflicting personas. They research independently, read each other's arguments, and clash until positions stop moving. You get the debate transcript and a verdict.

## Install

```bash
npx skills add sderosiaux/agents-crossfire
```

Or manually: copy `SKILL.md` to `~/.claude/commands/agents/debate.md`

## Use

```
/agents:debate should we fine-tune or just use RAG for our domain? ask codex and gemini
/agents:debate are AI coding agents ready to replace junior devs? use codex, gemini and sonnet
/agents:debate MCP vs tool-use for agent integrations, ask ollama and codex
```

No flags. Claude parses the topic and participants from your sentence. If you don't name anyone, it defaults to Claude + Codex.

## What happens

1. Claude creates a persona for each model (a cautious ops engineer, an aggressive strategist, a skeptical architect) designed to conflict
2. All models research and stake a position in parallel, each writing its own file
3. Each model reads the others' files and reacts directly, minimum 3 rounds
4. Claude writes a verdict with a coalition map showing who aligned with whom

Every response is written to disk by the model itself. Unfiltered, unedited. The debate folder is your audit trail.

## What the output looks like

From a "fine-tune vs RAG" debate (Codex vs Gemini vs Sonnet):

```
COALITION MAP:
Codex + Sonnet aligned: "RAG first, fine-tune only when retrieval
hits a wall." Gemini isolated: "Fine-tuning is underrated because
teams stop at naive RAG and never push past the retrieval ceiling."

KEY EXCHANGE:
Sonnet found that OpenAI's own fine-tuning docs recommend RAG as
the first approach. Codex verified: fine-tuning GPT-4 costs $25/M
tokens training, requires 50-100 quality examples minimum, and the
model can still hallucinate. Gemini countered: "You're comparing
bad fine-tuning to good RAG. A domain-tuned model with 500 examples
eliminates the retrieval latency and context window tax entirely."

VERDICT: Start with RAG. Fine-tune when you've hit a measurable
retrieval ceiling and have the labeled data to prove it. The 2-vs-1
coalition held: fine-tuning is a second-stage optimization, not a
starting point.
```

Codex checked pricing docs and fine-tuning guides. Sonnet found API deprecation timelines. Gemini argued from ML research papers. Three models, three kinds of evidence.

## Why different models, not different personas

Multi-agent setups like think-tank or SME review spin up multiple Claude instances with different system prompts. Same weights, same training, same blind spots. You get diversity of framing, not diversity of thought.

Crossfire uses models with different architectures, training data, and RLHF. When they disagree, it means something. When they converge despite all that, it means even more.

In our test, Codex (OpenAI) consistently verified claims against actual documentation. Gemini (Google) reasoned from ecosystem trends with zero web searches. Claude as a participant found architectural limitations nobody else spotted. Three different cognitive styles producing three different kinds of evidence.

## How models are called

| Model | Command |
|---|---|
| codex | `npx @openai/codex exec --ephemeral --skip-git-repo-check --full-auto -m gpt-5.4` |
| gemini | `npx @google/gemini-cli -m gemini-2.5-pro --yolo -p` |
| ollama | `ollama run --experimental {model}` |
| claude | `claude --no-session-persistence --dangerously-skip-permissions --model sonnet -p --output-format text` |

All configured for autonomous tool use (web search, bash, file reads, git clone). Models are encouraged to verify claims, not just reason from training data.

## File output

Each debate creates a folder:

```
crossfire-2026-03-29-finetune-vs-rag/
  00-setup.md              # personas (orchestrator)
  01-scatter-codex.md      # written by Codex
  01-scatter-gemini.md     # written by Gemini
  01-scatter-sonnet.md     # written by Sonnet
  02-clash-1-codex.md      # written by Codex
  02-clash-1-gemini.md     # written by Gemini
  02-clash-1-sonnet.md     # written by Sonnet
  03-assessment-1.md       # tension tracking (orchestrator)
  verdict.md               # synthesis + coalition map
  transcript.md            # full log
```

Each model writes its own markdown file. The orchestrator never rewrites participant output.

## Learnings from testing

A few things we found running real debates:

- Gemini with `--yolo` will simulate the entire debate by itself if you don't explicitly say "write ONLY your own file." The skill now includes negative constraints in every prompt.
- `--bare` on Claude blocks OAuth auth. Don't use it.
- Codex consistently does the most research (8 web searches in one debate). Gemini reasons from training data. Claude finds architectural edge cases. The model diversity is real.
- With 3 participants, one round often produces a 2-vs-1 coalition that is stronger signal than any single model's opinion.
- Minimum 3 rounds matters. Early agreement is shallow.

## Inspired by

- [discuss-skill](https://github.com/Restuta/discuss-skill) for the file-based protocol and structured turn-taking
- [Du et al. (2023)](https://arxiv.org/abs/2305.14325) showing multi-agent debate improves accuracy (ChatGPT-3.5 on GSM8K: 77% -> 85%)
- [Khan et al. (ICML 2024 Best Paper)](https://arxiv.org/abs/2402.06782) showing debating LLMs + human judge: accuracy 48% -> 76%

## License

MIT
