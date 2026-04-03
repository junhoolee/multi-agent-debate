# Changelog

All notable changes to this project will be documented in this file.

## [0.1.0] - 2026-04-03

### Added
- Initial release of multi-agent-debate plugin
- 3-agent structured debate: Claude (Moderator), Codex CLI (Pragmatist), Gemini CLI (Explorer)
- 2-layer fact-checking system:
  - Layer 1: Evidence Grounding Rule (agents self-verify via web search)
  - Layer 2: Moderator Fact-Check Gate (brave-search preferred, WebSearch fallback)
- Fact-check status taxonomy: VERIFIED, REFUTED, INCONCLUSIVE, DISPUTED, SEARCH-UNAVAILABLE
- Revision Loop (1 attempt per REFUTED claim)
- Separate factcheck-log.md to prevent transcript bloat
- Iron Law: every turn must add new information
- Quality Gate: format validation with Claude substitution on failure
- Decision Memo output format with evidence hierarchy
- CLI fallback with degraded mode disclosure
- Privacy notice for search query data exposure
