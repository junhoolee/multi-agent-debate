# Multi-Agent Debate

Structured multi-model debate for technical decisions: Claude (Moderator) x Codex (Pragmatist) x Gemini (Explorer) with 2-layer web-search fact-checking.

## Key Features

- **Real multi-model debate** -- not persona splitting within a single model. Claude moderates while Codex CLI and Gemini CLI provide genuinely independent perspectives with different training data and reasoning patterns.
- **2-layer fact-checking** -- Layer 1: Evidence Grounding Rule requires agents to self-verify claims via web search before asserting them. Layer 2: Moderator Fact-Check Gate independently verifies 2-3 claims after every agent turn using brave-search (preferred) or WebSearch (fallback).
- **Iron Law** -- every turn must add new information. Agreement without new evidence is a wasted turn and gets flagged as INVALID.
- **Quality Gate** -- agent output is validated for format compliance, minimum substance, and claim uniqueness. Invalid output triggers Claude substitution with the same persona lens.
- **Decision Memo output** -- structured recommendation with residual risks, next actions, debate summary, fact-check scores, and economics breakdown.

## Prerequisites

| Dependency | Required | Install |
|------------|----------|---------|
| Claude Code | Yes | Already running if you see this |
| Codex CLI (`@openai/codex`) | Optional | `npm i -g @openai/codex` |
| Gemini CLI (`@google/gemini-cli`) | Optional | `npm i -g @google/gemini-cli` |

Without Codex CLI or Gemini CLI installed, the skill runs in **degraded mode** where Claude substitutes for the missing agent. The debate still works but loses genuine model diversity. Self-debate mode (both CLIs missing) provides the lowest confidence results.

## Installation

```bash
# Add the marketplace (one-time)
claude plugin marketplace add github:junhoolee/multi-agent-debate

# Install the plugin
claude plugin install multi-agent-debate@junhoolee-multi-agent-debate
```

Or install directly from GitHub:

```bash
claude plugin install github:junhoolee/multi-agent-debate
```

## Privacy Notice

**The Fact-Check Gate sends debate claims as search queries to external services** (Brave Search API, Google via WebSearch, or Google Search via Gemini CLI). If your debate topic involves sensitive or proprietary information (e.g., acquisition targets, unreleased product names, strategic pivots), these details may be exposed to third-party search providers.

**Mitigations:**
- For sensitive topics, invoke with `no fact-check` in the arguments: `/multi-agent-debate "sensitive topic, no fact-check"`
- The Evidence Grounding Rule also causes Codex and Gemini to perform their own web searches. This cannot be disabled per-agent.
- Review your organization's data handling policies before debating confidential topics.

## Quick Start

Basic usage:

```
/multi-agent-debate "Should we migrate from SQLite to PostgreSQL?"
```

With custom round count:

```
/multi-agent-debate "Should we adopt microservices or stay monolithic? 5 rounds"
```

## How It Works

```
Phase 0: Setup
  Parse topic -> Create workspace -> Init transcript.md + factcheck-log.md -> Moderator frames the debate

Phase 1: Debate Rounds (repeat N times)
  Codex Turn (Pragmatist) -> Fact-Check Gate -> Gemini Turn (Explorer) -> Fact-Check Gate

Phase 2: Synthesis
  Read transcript + factcheck-log -> Produce Decision Memo (no "it depends")

Phase 3: Save & Report
  Save result.md -> Copy to docs/superpowers/ if exists -> Output 3-line summary + full memo
```

### Fact-Check Gate (runs after every agent turn)

1. **Extract claims** -- identify 2-3 verifiable factual claims from the agent's turn, prioritizing numbers and benchmarks.
2. **Search verification** -- search for each claim using brave-search MCP (preferred) or WebSearch (fallback).
3. **Classify** -- each claim receives a verdict:
   - VERIFIED -- search confirms the claim
   - REFUTED -- search contradicts the claim
   - INCONCLUSIVE -- mixed or insufficient results
   - SEARCH-UNAVAILABLE -- both brave-search and WebSearch unavailable
4. **Revision Loop** (on REFUTED) -- the agent gets exactly one chance to correct the refuted claim with the contradicting evidence provided. If still wrong after revision, the claim is marked DISPUTED.
5. **Log** -- full details go to factcheck-log.md; a compact summary is appended to the transcript.

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Round count | 3 | Number of Codex+Gemini debate rounds. Specify in the topic string (e.g., "topic 5 rounds"). |
| Fact-check | Enabled | The Fact-Check Gate runs after every agent turn using brave-search (preferred) or WebSearch (fallback). Skipped only if both are unavailable. |
| Output directory | `docs/superpowers/` if exists | Decision memo is saved to `docs/superpowers/{date}-{topic-slug}-debate.md` when the directory exists; otherwise kept in `/tmp/debate-*`. |

## Benchmarks

Measured from an initial smoke test: "SQLite vs PostgreSQL for a SaaS with <100 users" (2 rounds, full mode).

| Metric | Value | Notes |
|--------|-------|-------|
| Latency (2 rounds, full mode) | ~8 min | Sequential: 4 agent turns + 4 fact-check gates + synthesis |
| Latency (3 rounds, estimated) | ~12 min | Extrapolated from 2-round data |
| Token usage (2 rounds, full mode) | ~45K | Across Claude + Codex + Gemini + fact-check searches |
| Token usage (3 rounds, estimated) | ~60-65K | Transcript growth is cumulative per round |
| Fact-check searches performed | 10 | 2-3 claims verified per turn × 4 turns |
| Fact-check results | 8✅ 0❌ 2🔍 | No REFUTED claims in this run; 2 INCONCLUSIVE (unconfirmed source URLs) |
| Revision loops triggered | 0 | All claims were VERIFIED or INCONCLUSIVE |

> These numbers are from a single run. Latency and token usage vary with topic complexity, agent response length, and revision loop triggers. More benchmarks will be added as the plugin sees broader usage.

## Limitations

- **Latency**: A full 3-round debate takes approximately 5-15 minutes due to sequential CLI calls and WebSearch verification.
- **Cost**: Expect ~40K+ tokens total across all models for a 3-round debate. Each additional round adds ~10K tokens.
- **CLI availability**: Codex CLI and Gemini CLI must be installed and authenticated separately. API keys or login sessions must be active.
- **Degraded mode**: When one or both external CLIs are unavailable, Claude substitutes for the missing agent. This loses genuine model diversity -- the debate still runs but confidence should be discounted.
- **Fact-check accuracy**: WebSearch results depend on query quality and source availability. INCONCLUSIVE verdicts are common for niche or very recent topics.

## License

MIT -- see [LICENSE](LICENSE) for details.
