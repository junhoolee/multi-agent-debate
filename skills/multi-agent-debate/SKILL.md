---
name: multi-agent-debate
description: "Use when making decisions through structured multi-agent debate — 'debate this', 'multi-perspective analysis', 'argue both sides', 'help me decide'. Claude/Codex/Gemini take turns with counter-arguments, expansions, and modifications to reach an optimal conclusion."
---

# Multi-Agent Debate Protocol with 2-Layer Fact-Checking

Three agents conduct a structured debate on any technical or strategic decision:

| Role | Agent | Lens |
|------|-------|------|
| **Moderator** | Claude Code | Balances positions, enforces structure, runs fact-check gates |
| **Pragmatist** | Codex CLI | Cost, schedule, operational risk, proven patterns |
| **Explorer** | Gemini CLI | Research frontiers, long-term scalability, unconventional options |

Each turn must **Refute**, **Expand**, or **Modify** the previous position. All factual claims are verified through a 2-layer fact-checking system: the Evidence Grounding Rule (agents self-check before claiming) and the Moderator Fact-Check Gate (independent verification after each turn).

---

## The Iron Law

```
EVERY TURN MUST ADD NEW INFORMATION.
AGREEMENT WITHOUT NEW EVIDENCE IS A WASTED TURN.
```

If an agent agrees with a prior position, it MUST bring new data, a new angle, or a new risk that was not previously mentioned. Mere rephrasing or enthusiastic agreement is forbidden.

---

## When to Use

- Technical architecture decisions with genuine trade-offs
- Strategic choices where multiple valid paths exist
- Technology selection (framework, database, cloud provider, etc.)
- Process or workflow design with competing priorities
- **Especially when you feel certain** — certainty is precisely when blind spots hide

## When NOT to Use

- Pure fact-checking (use brave-search or WebSearch directly)
- Clear-cut implementation with no ambiguity
- Subjective preference with no measurable impact ("tabs vs spaces")
- Urgent hotfixes where speed trumps deliberation

## Privacy Notice

**The Fact-Check Gate sends debate claims as search queries to Brave Search, Google, and other external services.** If the debate topic involves confidential business decisions, unreleased product names, or proprietary strategies, these details will be exposed to third-party search providers. For sensitive topics, the user should include "no fact-check" in the arguments to skip the Fact-Check Gate.

---

## Phase 0: Setup

**Objective**: Parse the debate topic, configure rounds, and initialize tracking files.

### Steps

1. **Parse arguments** from `$ARGUMENTS`:
   - Extract the debate topic (required)
   - Extract round count (optional, default: 3)
   - Example: `"Should we migrate to PostgreSQL? 3 rounds"` → topic = "Should we migrate to PostgreSQL?", rounds = 3

2. **Create debate workspace**:
   ```bash
   DEBATE_DIR="/tmp/debate-$(date +%s)"
   mkdir -p "$DEBATE_DIR"
   ```

3. **Initialize transcript.md**:
   ```markdown
   # Debate: {topic}
   - Date: {YYYY-MM-DD}
   - Rounds: {n}
   - Mode: {full | degraded-codex | degraded-gemini | self-debate}
   ---
   ```

4. **Initialize factcheck-log.md**:
   ```markdown
   # Fact-Check Log: {topic}
   - Date: {YYYY-MM-DD}
   ---
   ```

5. **Moderator writes initial position**: Claude Code writes a balanced 3-5 sentence framing of the topic, identifying the core tension and key dimensions to explore.

### Gate Condition
- [ ] `$DEBATE_DIR/transcript.md` exists and contains the header
- [ ] `$DEBATE_DIR/factcheck-log.md` exists and contains the header
- [ ] Initial moderator position is written to transcript.md

**Do NOT proceed to Phase 1 until all three conditions are met.**

---

## Phase 1: Debate Rounds

Each round consists of exactly two turns executed **sequentially** (never in parallel):

```
Round N: Codex Turn → Fact-Check Gate → Gemini Turn → Fact-Check Gate
```

### Codex Turn (Pragmatist)

Execute via Bash tool (timeout: 200000ms). The transcript is piped via stdin; output written to a file:

```bash
cat "$DEBATE_DIR/transcript.md" | codex exec \
  --sandbox read-only \
  --ephemeral \
  --skip-git-repo-check \
  -C "$DEBATE_DIR" \
  -o "$DEBATE_DIR/codex-out.md" \
  "$(cat <<'CODEX_PROMPT'
You are the PRAGMATIST in a structured debate.
Your perspective is pragmatism and engineering reality.

## Your Lens
- Prioritize implementation complexity, maintenance cost, and team capability
- Skeptical of theoretically elegant solutions that fail in practice
- Always ask: "Can we ship this today?"
- Wary of over-abstraction and speculative future-proofing

## Evidence Grounding Rule
Before making ANY factual claim (performance numbers, adoption statistics, framework behavior, pricing, benchmark results), you MUST:
1. Search the web to verify the claim
2. Cite the source URL or document name
3. If you cannot verify a claim, prefix it with [UNVERIFIED]

Claims without sources or [UNVERIFIED] tags are treated as opinion, not evidence.

## Action Rules
Read the full debate transcript from <stdin> and perform EXACTLY ONE of:

1. **Refute**: Identify logical flaws, overlooked risks, or incorrect assumptions. Provide counter-evidence.
2. **Expand**: Add a new perspective, case study, data point, or scenario not yet discussed.
3. **Modify**: Partially accept a prior proposal but add specific conditions, changes, or improvements.

## Hard Rules
- "I agree" alone is NEVER acceptable. You MUST add new information.
- No vague language ("it might", "it depends"). Be specific and evidence-based.
- Do not repeat prior arguments. Provide fresh insight.
- Text response only — do not modify files or execute code.

## Output Format (follow exactly)
[Action: Refute/Expand/Modify]

Reason: {why you chose this action, 1 sentence}

{Body — specific, evidence-based argument through your Pragmatist lens. Cite sources.}

**Core claim**: {1-2 sentence summary}
CODEX_PROMPT
)"
```

Append Codex output to transcript:
```bash
echo -e "\n## [Round {round}] Codex (Pragmatist)\n" >> "$DEBATE_DIR/transcript.md"
cat "$DEBATE_DIR/codex-out.md" >> "$DEBATE_DIR/transcript.md"
echo -e "\n\n---" >> "$DEBATE_DIR/transcript.md"
```

### Quality Gate (after each agent turn)

The Moderator checks the agent output for:

1. **Empty or too short** (fewer than 100 characters of substance) → INVALID
2. **Missing format** (no `[Action:]` tag or no `**Core claim**`) → INVALID
3. **Repeated core claim** (same core claim as a previous turn by any agent) → INVALID

If INVALID: Log the failure reason, then **Claude substitutes** for the failing agent using the same persona lens (Pragmatist or Explorer). Append a notice to transcript: `[SUBSTITUTION: Claude acting as {Persona} — {reason}]`.

### Moderator Fact-Check Gate

**This gate runs after EVERY agent turn.** It is the heart of debate integrity.

#### Step 1: Extract Claims

From the agent's turn, extract 2-3 verifiable factual claims. Prioritize by verifiability:

1. **Numbers** (performance benchmarks, costs, adoption percentages) — highest priority
2. **Framework/tool behavior** ("React re-renders on every state change")
3. **Citations** (verify the cited source actually says what was claimed)
4. **General technical claims** ("microservices increase deployment complexity") — lowest priority

#### Step 2: Search Verification

For each extracted claim, search using **brave-search MCP tools first** (e.g., `brave_web_search`). Fall back to WebSearch only if brave-search is unavailable:

```
Claim: "{exact claim text}"
Search query: {targeted search query}
```

#### Step 3: Classify Results

Each claim receives one of five verdicts:

| Verdict | Symbol | Meaning |
|---------|--------|---------|
| VERIFIED | ✅ | Search confirms the claim with supporting evidence |
| REFUTED | ❌ | Search directly contradicts the claim |
| INCONCLUSIVE | 🔍 | Search returns mixed or insufficient results |
| DISPUTED | ⚠️ | Agent failed to correct a REFUTED claim after Revision Loop |
| SEARCH-UNAVAILABLE | ⛔ | brave-search and WebSearch both unavailable or errored |

#### Step 4: Revision Loop (on REFUTED claims)

When a claim is classified as REFUTED:

1. **One revision attempt only** — make a new CLI call to the same agent with the refutation evidence:
   ```
   "Your claim '{claim}' has been REFUTED. Evidence: {search results summary}.
   Please revise your position on this specific point. Keep your [Action] and core argument,
   but correct or withdraw the refuted claim."
   ```
2. Re-verify the revised claim via brave-search (or WebSearch fallback)
3. If still REFUTED after revision → classify as ⚠️ DISPUTED and proceed
4. If corrected → classify as ✅ VERIFIED (revised)

#### Step 5: Log to factcheck-log.md (FULL details)

Append the complete fact-check record:

```markdown
## Round {n} — {Agent Name} Turn

### Claim 1: "{claim text}"
- Search query: {query}
- Search results summary: {2-3 sentence summary}
- Verdict: ✅ VERIFIED / ❌ REFUTED / 🔍 INCONCLUSIVE / ⚠️ DISPUTED / ⛔ SEARCH-UNAVAILABLE
- Evidence: {key supporting or contradicting evidence}
- Revision: {if applicable, note revision attempt and outcome}

### Claim 2: "{claim text}"
...
```

#### Step 6: Append COMPACT summary to transcript.md

To prevent transcript bloat, only append a brief summary:

```markdown
> **Fact-Check [{Agent}]**: 2/3 VERIFIED, 0 REFUTED, 1 INCONCLUSIVE
> - ✅ "{claim 1 short}" — confirmed
> - ✅ "{claim 2 short}" — confirmed
> - 🔍 "{claim 3 short}" — insufficient data
```

#### Search Unavailable Fallback

If both brave-search MCP and WebSearch are unavailable or return errors:
- Log: `⛔ SEARCH-UNAVAILABLE: No search tools accessible. Fact-check skipped for this turn.`
- Append warning to transcript: `> ⚠️ Fact-check skipped (search unavailable) — claims unverified`
- **Continue the debate** — do not block on unavailable search
- Note the gap in the final synthesis

### Gemini Turn (Explorer)

Execute via Bash tool (timeout: 200000ms). The transcript is piped via stdin:

```bash
GEMINI_PROMPT=$(cat <<'GEMINI_PROMPT'
You are the EXPLORER in a structured debate.
Your perspective is exploratory thinking and research.

## Your Lens
- Explore possibilities, alternatives, and long-term impacts not yet discussed
- Use "What if...?" scenarios to reveal blind spots
- Reference academic research, industry case studies, and lessons from similar projects
- Ask whether the direction is correct, not just whether it is practical

## Evidence Grounding Rule
Before making ANY factual claim (performance numbers, adoption statistics, framework behavior, pricing, benchmark results), you MUST:
1. Search the web to verify the claim
2. Cite the source URL or document name
3. If you cannot verify a claim, prefix it with [UNVERIFIED]

Claims without sources or [UNVERIFIED] tags are treated as opinion, not evidence.

## Action Rules
Read the full debate transcript below and perform EXACTLY ONE of:

1. **Refute**: Identify logical flaws, overlooked risks, or incorrect assumptions. Provide counter-evidence.
2. **Expand**: Add a new perspective, case study, data point, or scenario not yet discussed.
3. **Modify**: Partially accept a prior proposal but add specific conditions, changes, or improvements.

## Hard Rules
- "I agree" alone is NEVER acceptable. You MUST add new information.
- No vague language. Be specific and evidence-based.
- Do not repeat prior arguments. Provide fresh insight.
- Text response only — do not modify files or execute code.

## Output Format (follow exactly)
[Action: Refute/Expand/Modify]

Reason: {why you chose this action, 1 sentence}

{Body — specific, evidence-based argument through your Explorer lens. Cite sources.}

**Core claim**: {1-2 sentence summary}
GEMINI_PROMPT
)

# Try default model, fall back to gemini-2.5-flash
cat "$DEBATE_DIR/transcript.md" | gemini -p "$GEMINI_PROMPT" > "$DEBATE_DIR/gemini-out.md" 2>"$DEBATE_DIR/gemini-err.md"
if [ ! -s "$DEBATE_DIR/gemini-out.md" ]; then
  cat "$DEBATE_DIR/transcript.md" | gemini -m gemini-2.5-flash -p "$GEMINI_PROMPT" > "$DEBATE_DIR/gemini-out.md" 2>"$DEBATE_DIR/gemini-err.md"
fi
```

Append Gemini output to transcript:
```bash
echo -e "\n## [Round {round}] Gemini (Explorer)\n" >> "$DEBATE_DIR/transcript.md"
cat "$DEBATE_DIR/gemini-out.md" >> "$DEBATE_DIR/transcript.md"
echo -e "\n\n---" >> "$DEBATE_DIR/transcript.md"
```

**Gemini CLI Fallback**: If both models fail, Claude substitutes as Explorer.

### Round Gate Condition

- [ ] All {n} rounds are complete (each with Codex turn + Fact-Check + Gemini turn + Fact-Check)
- [ ] All Fact-Check Gates have been executed (even if some resulted in SEARCH-UNAVAILABLE)

**Do NOT proceed to Phase 2 until the gate is satisfied.**

---

## Phase 2: Synthesis

The Moderator reads the **full transcript.md** AND **factcheck-log.md**, then produces a Decision Memo.

### Decision Memo Format

```markdown
# Decision: {topic}

## Recommendation

{Definitive recommendation in 2-3 sentences. No hedging. No "it depends." Pick a side.}

## Why

{3-5 bullet points explaining the rationale. Reference fact-check results where relevant:}
- Point backed by ✅ VERIFIED claims carries full weight
- Point with ⚠️ DISPUTED claims noted with caveat
- Point with only 🔍 INCONCLUSIVE evidence flagged as lower confidence

## Residual Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| {risk 1} | {High/Medium/Low} | {mitigation strategy} |
| {risk 2} | {High/Medium/Low} | {mitigation strategy} |
| {risk 3} | {High/Medium/Low} | {mitigation strategy} |

## Next Actions

1. {Immediate concrete action}
2. {Follow-up action}
3. {Validation step}

## Debate Summary

| Agent | Rounds | Core Position | Fact-Check Score |
|-------|--------|---------------|------------------|
| Codex (Pragmatist) | {n} | {1-sentence summary} | {x}✅ {y}❌ {z}🔍 {w}⚠️ {v}⛔ |
| Gemini (Explorer) | {n} | {1-sentence summary} | {x}✅ {y}❌ {z}🔍 {w}⚠️ {v}⛔ |

## Economics

- **Mode**: {full | degraded-codex | degraded-gemini | self-debate}
- **Rounds**: {n}
- **Estimated tokens**: {approximate total across all agents}
- **Estimated latency**: {approximate wall-clock time}
- **Searches performed**: {count}
- **Revision Loop invocations**: {count}
```

### Synthesis Rules

1. **No "it depends"** — you must commit to a recommendation
2. **No new arguments** — synthesis uses only what emerged during debate
3. **Fact-checked claims carry more weight** — ✅ VERIFIED > 🔍 INCONCLUSIVE > ⚠️ DISPUTED
4. **Acknowledge the strongest counter-argument** — show you understood the losing side
5. **Residual risks must be concrete** — not vague hand-waving

---

## Phase 3: Save & Report

### Save

1. Write the Decision Memo to `$DEBATE_DIR/result.md`
2. If `docs/superpowers/` directory exists in the project, copy to:
   ```
   docs/superpowers/{YYYY-MM-DD}-{topic-slug}-debate.md
   ```
3. If `docs/superpowers/` does not exist, keep in `$DEBATE_DIR` only

### Report

Output to the user in this order:

1. **3-line summary FIRST** (before the full result):
   ```
   Debate: {topic}
   Recommendation: {one-sentence recommendation}
   Confidence: {High|Medium|Low} (based on fact-check scores)
   ```

2. **Full Decision Memo** (the complete result.md content)

---

## CLI Fallback Table

| Failure | Fallback | Warning |
|---------|----------|---------|
| `codex exec` fails or times out | Claude acts as Pragmatist | `[DEGRADED MODE: Claude substituting for Codex as Pragmatist]` |
| `gemini -p` fails (both models) | Claude acts as Explorer | `[DEGRADED MODE: Claude substituting for Gemini as Explorer]` |
| Both Codex and Gemini fail | Claude self-debates both roles | `[SELF-DEBATE MODE: Claude performing all roles]` |
| brave-search + WebSearch both unavailable | Skip fact-check for that turn | `[WARNING: Fact-check skipped — no search tools available]` |
| Search intermittent | Retry once, then skip | `[WARNING: Fact-check partial — some searches failed]` |

**All Bash tool calls use timeout: 200000ms.**

When substituting for a failed agent, Claude MUST:
- Explicitly adopt the persona's lens (Pragmatist = cost/schedule focus, Explorer = research/innovation focus)
- Not simply agree with its own moderator position
- Still follow the Iron Law: add new information

---

## Red Flags

If you catch yourself thinking any of these, **STOP immediately** and re-read the Iron Law:

- "I think they're both making good points, let me just summarize..."
- "This is getting repetitive, let me wrap up early..."
- "I agree with the previous speaker..."
- "The agents are converging, no need for more rounds..."
- "Let me skip the format requirements to save tokens..."
- "The claim is probably right, no need to search..."
- "The fact-check is slowing things down, let's skip it..."
- "One REFUTED claim isn't a big deal..."
- "The agent revised their claim, no need to re-verify..."
- "Search didn't return much, I'll just mark it VERIFIED based on my knowledge..."

---

## Common Rationalizations

| # | Excuse | Reality Check |
|---|--------|---------------|
| 1 | "The agents agree, so we have consensus" | Agreement without new evidence means the debate failed. Add a contrarian round. |
| 2 | "Three rounds is enough" | If no strong counter-argument has emerged, the debate was too shallow, not too long. |
| 3 | "I'll add the Fact-Check Gate results in the synthesis" | Fact-checks must happen after EACH turn. Batching defeats the purpose — later arguments may build on refuted claims. |
| 4 | "The topic is too simple for a full debate" | Then why invoke this skill? Use direct implementation instead. |
| 5 | "Codex/Gemini output was close enough to the format" | Missing `[Action:]` or `**Core claim**` means the output is INVALID. No exceptions. |
| 6 | "The REFUTED claim was a minor detail" | Minor details compound. A REFUTED benchmark number can invalidate an entire cost argument. Always run the Revision Loop. |
| 7 | "Self-debate mode is basically the same quality" | It is not. Self-debate lacks genuine model diversity. Always disclose degraded mode and acknowledge reduced confidence. |

---

## Agent Persona Reference

| Attribute | Claude (Moderator) | Codex (Pragmatist) | Gemini (Explorer) |
|-----------|-------------------|--------------------|--------------------|
| **Primary lens** | Balance, synthesis, fact-checking | Cost, schedule, operational risk | Research, innovation, long-term scalability |
| **Tendency** | Seeks convergence | Favors proven, battle-tested solutions | Favors cutting-edge, novel approaches |
| **Strength** | Sees both sides, verifies claims | Grounds discussion in reality | Expands possibility space |
| **Blind spot** | May compromise too early | May dismiss innovation as risky | May underestimate practical constraints |
| **Fact-check role** | Runs the Fact-Check Gate (brave-search preferred, WebSearch fallback) | Provides claims to verify | Provides claims to verify |

---

## Quick Reference

| Phase | Key Activities | Gate Condition |
|-------|---------------|----------------|
| **Phase 0: Setup** | Parse topic, create workspace, init transcript.md + factcheck-log.md, write initial position | Both files exist, initial position written |
| **Phase 1: Debate** | Codex turn → Fact-Check Gate → Gemini turn → Fact-Check Gate (repeat n rounds) | All n rounds complete with all Fact-Check Gates executed |
| **Phase 2: Synthesis** | Read transcript + factcheck-log, produce Decision Memo with fact-check-weighted reasoning | Decision Memo follows format, no "it depends", no new arguments |
| **Phase 3: Save & Report** | Save result.md, copy to docs/superpowers/ if exists, output 3-line summary then full memo | Files saved, user sees summary + full result |

---

## Key Principles

1. **The Iron Law is non-negotiable** — every turn must add new information or be invalidated
2. **Sequential execution only** — Codex before Gemini, never parallel, to enable genuine response to prior arguments
3. **Format compliance is binary** — output either meets the format or is INVALID, no partial credit
4. **The Fact-Check Gate runs after every turn** — not batched, not skipped, not deferred
5. **The Evidence Grounding Rule is preventive** — agents self-verify before claiming; the Fact-Check Gate is detective verification after
6. **REFUTED claims trigger exactly one Revision Loop** — not zero, not two, exactly one chance to correct
7. **Degraded mode must be disclosed honestly** — substitutions and skipped fact-checks are always visible in the transcript and final report
8. **Synthesis follows evidence hierarchy** — VERIFIED claims outweigh INCONCLUSIVE which outweigh DISPUTED
9. **The debate serves the decision** — if after all rounds the answer is genuinely unclear, say so with specific reasons, but still make a recommendation
