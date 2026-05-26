# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Academic Research Skills (ARS) is a source-available academic research copilot framework for Claude Code. It provides a 10-stage pipeline (research → write → integrity → review → revise → final integrity → finalize) with mandatory human checkpoints. The code is agent prompts (markdown), validation scripts (Python 3), schemas (JSON), and slash command definitions. This is not traditional code—most logic lives in detailed agent prompts and validation contracts.

**License:** CC-BY-NC 4.0 (Academic/non-commercial use only)

## Essential Commands

All commands run from repository root. Use Python 3.x.

```bash
# Full spec consistency check (runs ~30 sub-checks, 3-5 minutes)
# Entry point for all CI checks. This is what CI runs on every PR.
pip install -r requirements-dev.txt
python3 scripts/check_spec_consistency.py

# Run individual lint checks (when working on a specific feature)
python3 scripts/check_version_consistency.py              # Sync versions across files
python3 scripts/check_data_access_level.py                 # Validate data_access_level frontmatter
python3 scripts/check_task_type.py                         # Validate task_type frontmatter
python3 scripts/check_sprint_contract.py <json-file>       # Reviewer/writer/evaluator contracts
python3 scripts/check_collaboration_depth_rubric.py        # Observer agent rubric
python3 scripts/check_literature_corpus_schema.py          # Passport corpus schemas
python3 scripts/check_corpus_consumer_protocol.py          # v3.6.5 consumer protocol
python3 scripts/check_passport_reset_contract.py           # v3.6.3 reset boundary
python3 scripts/check_v3_6_7_pattern_protection.py         # v3.6.7 pattern protection
python3 scripts/check_v3_7_3_three_layer_citation.py       # v3.7.3 citation anchors
python3 scripts/check_claim_audit_consistency.py           # v3.8 claim audit invariants
```

```bash
# Run unit tests (required when modifying lint scripts)
python3 -m unittest scripts.test_check_sprint_contract -v
python3 -m unittest scripts.test_check_collaboration_depth_rubric -v
python3 -m unittest scripts.test_check_passport_reset_contract -v
python3 -m unittest scripts.test_check_v3_6_7_pattern_protection -v
python3 -m unittest scripts.test_claim_audit_schema scripts.test_claim_audit_pipeline -v

# Or run all pytest tests
pip install pytest pyyaml jsonschema
pytest scripts/test_check_*.py -v
```

```bash
# Validate a sprint contract JSON against schema
python3 scripts/check_sprint_contract.py shared/contracts/reviewer/full.json --ars-version v3.8.2
```

```bash
# Run a specific claim audit calibration test (when modifying audit logic)
python3 -m unittest scripts.test_claim_audit_calibration -v
```

## Architecture Overview

### Four Skills

- **`deep-research/`** — 7-mode research engine (socratic/full/systematic-review/etc). 13 agents. Produces RQ Brief, Annotated Bibliography, Synthesis Report.
- **`academic-paper/`** — 10-mode paper writing engine (full/plan/revision/etc). 12 agents. Produces Draft, Bilingual Abstract, Citation List.
- **`academic-paper-reviewer/`** — 6-mode review engine (full/re-review/methodology-focus/etc). 7 agents. Produces EIC + 3 Reviewers + Devil's Advocate reports.
- **`academic-pipeline/`** — Orchestrator coordinating all above. 10-stage pipeline with checkpoints and integrity gates.

Mode details: `MODE_REGISTRY.md` (single source of truth for all 25 modes, triggers, spectrum classification).

### Plugin Packaging (v3.7.0+)

- **`.claude-plugin/`** — Plugin manifest for Claude Code marketplace
- **`commands/ars-*.md`** — 10 slash command definitions mapping modes to `/ars-<mode>` triggers
- **`agents/*.md`** — 3 plugin-shipped agents (symlinks to deep-research/agents/ with `model: inherit` frontmatter)
- **`skills/`** — Symlink dir used by plugin loader

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `shared/` | Cross-cutting schemas (`handoff_schemas.md`), contracts (`contracts/`), patterns, rubrics |
| `shared/contracts/reviewer/` | Sprint contract templates (Schema 13.1) |
| `shared/contracts/writer/` | Writer contracts (Schema 13.1) |
| `shared/contracts/evaluator/` | Evaluator contracts (Schema 13.1) |
| `shared/contracts/passport/` | Material Passport schemas (Schema 9) |
| `scripts/` | ~60 Python lint/validation scripts. Most important: `check_spec_consistency.py`, `check_sprint_contract.py`, `check_claim_audit_consistency.py` |
| `scripts/adapters/` | Reference Python adapters for literature corpus ingestion (Zotero, Obsidian, folder scan) |
| `docs/ARCHITECTURE.md` | Full 10-stage pipeline matrix, flow diagrams, data access levels, quality gates |
| `docs/design/` | Design specs for each versioned feature (v3.7.3, v3.6.8, etc) |
| `docs/PERFORMANCE.md` | Token budgets, cost estimates, session management guidance |
| `tests/fixtures/` | Test fixtures for claim audit, pattern eval, A/B manifests |

### Data Access Level System (v3.3.2+)

Every skill declares `data_access_level`:
- `raw` — deep-research (ground truth isolation, operates on unverified sources)
- `redacted` — academic-paper (operates on sanitized materials, can make new claims)
- `verified_only` — academic-paper-reviewer, academic-pipeline (reads only verified/integrity-gated materials)

The gate enforcement points are Stage 2.5 and 4.5 (integrity verification). See `shared/ground_truth_isolation_pattern.md`.

### Important Cross-Cutting Files

| File | What it does |
|------|--------------|
| `shared/handoff_schemas.md` | Defines Material Passport (Schema 9), stage handoff formats, history arrays |
| `shared/sprint_contract.schema.json` | Schema 13.1 — reviewer/writer/evaluator pre-commitment contracts |
| `shared/collaboration_depth_rubric.md` | 4-dimension rubric for `collaboration_depth_agent` (advisory only) |
| `MODE_REGISTRY.md` | 25 modes × 4 skills with triggers and oversight levels |
| `CHANGELOG.md` | Versioned release notes (runs 100+ KB; extremely detailed) |

### Agent File Structure

Agent prompts are markdown files with YAML frontmatter:

```yaml
---
name: research_question_agent
description: Generates research questions from user input
model: sonnet  # or opus, or 'inherit' for plugin agents
---
```

Agents live in `<skill>/agents/<agent_name>_agent.md`. Agent prompts follow a cognitive framework pattern — they teach "how to think" not just procedures. Look for `PATTERN PROTECTION` sections in hardened agents (v3.6.7+).

### Environment Variables

```bash
# Pipeline control
ARS_PASSPORT_RESET=1           # Opt-in: promote every FULL checkpoint to reset boundary
ARS_CLAIM_AUDIT=1              # Opt-in: enable v3.8 claim-faithfulness audit gate (default OFF)
ARS_SOCRATIC_READING_PROBE=1   # Opt-in: enable v3.5.1 reading-check probe
ARS_CROSS_MODEL=1              # Enable cross-model verification (GPT/Gemini)

# API keys (user must set)
ANTHROPIC_API_KEY              # Required for Claude Code
S2_API_KEY                     # Optional: Semantic Scholar API (higher rate limits)
```

### CI/CD

`.github/workflows/spec-consistency.yml` runs on push/PR. It executes:
1. `check_spec_consistency.py` — cross-file version/sync checks
2. All sprint contract validation (reviewer + writer + evaluator)
3. All v3.6.7, v3.6.8, v3.7.3, v3.8 lints and their unit tests
4. Collaboration depth and benchmark report checks

### Testing Approach

No traditional unit tests for "business logic" — logic lives in LLM prompts. Instead:
- **Lint scripts** enforce structural invariants (schema, naming, line budgets)
- **Mutation tests** verify lints catch prompt modifications
- **Unit tests** verify the lint scripts themselves
- **E2E tests** verify claim audit pipeline on synthetic papers

When modifying agent prompts: check if there's a matching `test_check_v3_X_Y_pattern_protection.py` or similar. The lint byte-equivalence gates (v3.6.7+) prevent drift in hardened sections.

### Version-Gated Features

ARS evolves through feature contracts tracked by version:
- **v3.6.7** — Pattern protection (A1-A5, B1-B5, C1-C3) in downstream agents
- **v3.6.8** — Generator-evaluator contract (Schema 13.1) with two-phase paper-blind/paper-visible protocol
- **v3.7.3** — Three-layer citation emission (`<!--anchor:kind:value-->`) and contamination signals
- **v3.8.0** — Claim-faithfulness audit (retrieval against locator anchors) with HIGH-WARN gate refusal

Each version has a design spec in `docs/design/` and a matching lint in `scripts/`.

## Routing Rules for Users

From `README.md` and current CLAUDE.md. Apply these when a user asks about ARS:

1. **Individual skill vs pipeline**: If user needs single function (just research, just write, just review) — trigger the skill directly without pipeline. If end-to-end — use pipeline.

2. **deep-research socratic vs full**: If research question is unclear or user says "guide me" — `socratic` mode. If defined question — `full` mode.

3. **academic-paper plan vs full**: If user wants to think through structure — `plan` mode. If ready to write — `full` mode.

4. **academic-paper-reviewer guided vs full**: If user wants to learn from review — `guided` mode. If standard assessment — `full` mode.

5. **Language**: Default output language matches user input (Traditional Chinese or English).

## Handoff Materials

| Stage transition | Key materials passed |
|------------------|---------------------|
| deep-research → academic-paper | RQ Brief, Methodology Blueprint, Annotated Bibliography, Synthesis Report, INSIGHT Collection |
| academic-paper → academic-paper-reviewer | Complete paper text (field_analyst_agent auto-detects domain) |
| academic-paper-reviewer → academic-paper (revision) | Editorial Decision Letter, Revision Roadmap, Per-reviewer detailed comments |

