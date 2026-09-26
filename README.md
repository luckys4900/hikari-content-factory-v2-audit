# HIKARI CONTENT FACTORY v2 — Audit & Architecture

> **Claude Code users:** use the clean, brand-safe engineering repository [hikari-influencer-content-factory](https://github.com/luckys4900/hikari-influencer-content-factory). This repository is a historical audit record and is not the implementation workspace for Claude Code.

Read-only audit and PoC-first architecture for a local, fictional adult AI character still-image → I2V pipeline.

This repository is intentionally **documentation-only**. It contains no source photographs, generated adult media, face embeddings, image hashes, model weights, or executable generation workflows.

## What was audited

- A legacy local generation project (“Nanase Nana”)
- 44 candidate reference images for a new fictional adult character (“Hikari”)
- Static-image identity/body consistency, anatomy QC, resource limits, and I2V readiness

The original local sources are not part of this repository. Evidence paths use placeholders:

- `${NANASE_SOURCE_ROOT}`
- `${HIKARI_SOURCE_ROOT}`
- `${HIKARI_V2_ROOT}`

## Main conclusion

The legacy pipeline did not fail because adult prompting was categorically blocked. It failed because several technical requirements could not be satisfied together:

- the Qwen AIO Q4 route exhausted safe system RAM;
- source latent / garment authority resisted whole-body changes;
- full-frame regeneration weakened face and body identity;
- tight face compositing improved similarity but created pasted-face artifacts;
- pose sources and contact sheets were incorrectly used as generation authority;
- the anatomy gate missed visible arm/hand failures;
- identity-specific conditioning remained an investigation rather than a production dependency;
- no local I2V backend was installed.

The replacement design is a strict vertical slice:

1. Verify provenance and adult/synthetic-or-licensed status.
2. Lock a hash-pinned, role-separated canonical asset registry locally.
3. Produce and accept one consistent still image.
4. Use that same accepted still to produce exactly three 15-second I2V candidates.
5. Accept at least one video or stop with explicit failure reasons.
6. Defer GUI, bulk generation, and publishing until the PoC passes.

## Repository map

- [`AUDIT_REPORT.md`](AUDIT_REPORT.md) — full A–I audit and architecture report
- [`CODEX_IMPLEMENTATION_HANDOFF.md`](CODEX_IMPLEMENTATION_HANDOFF.md) — Phase 1 implementation prompt
- [`LLM_AUDIT_PROMPT.md`](LLM_AUDIT_PROMPT.md) — prompt for an independent Codex/ChatGPT review
- [`AGENTS.md`](AGENTS.md) — repository-specific review rules for coding agents
- [`docs/EVIDENCE_INDEX.md`](docs/EVIDENCE_INDEX.md) — local evidence map with placeholder roots
- [`data/audit_manifest.json`](data/audit_manifest.json) — machine-readable findings and boundaries
- [`data/face_similarity_calibration.json`](data/face_similarity_calibration.json) — provisional aggregate calibration only

## How to ask an LLM to audit it

Give the repository URL to a browsing-capable model and paste the contents of `LLM_AUDIT_PROMPT.md`. If the model cannot access GitHub, upload the Markdown and JSON files directly.

The independent reviewer should distinguish:

- facts supported by recorded evidence;
- architectural recommendations;
- provisional thresholds;
- claims that still require local execution or human review.

## Safety and privacy boundary

- No real-person non-consensual sexual editing is in scope.
- No minors or subjects who appear underage are in scope.
- Local source images must not be copied into this repository.
- Face similarity is selection support, not proof of identity, age, fictionality, ownership, or consent.
- A verified `subject_provenance.json` is a hard prerequisite for any real generation backend.

## Status

Audit and design: complete.  
Implementation: not started in this repository.  
Model installation or generation: not performed.



