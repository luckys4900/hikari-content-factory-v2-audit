# Repository instructions for coding agents

## Default task

Review this repository as documentation and architecture. Do not start Phase 1 implementation unless the user explicitly requests implementation.

## Evidence rules

- Separate recorded evidence, inference, recommendation, and unverified assumption.
- Later final-review reports supersede earlier technical-pass reports when they evaluate the same candidates.
- `data/face_similarity_calibration.json` is provisional and must not become a hard identity gate without positive and negative validation.
- The omitted local code, images, logs, and hardware cannot be treated as independently verified from this repository alone.

## Safety and scope

- Never add source photographs, face embeddings, image hashes, local usernames, secrets, model weights, or adult generated media to this repository.
- Never support real-person non-consensual sexual editing or any subject who is or appears underage.
- Require verified provenance before enabling a real generation backend.
- Do not download models, call paid APIs, or execute image/video generation as part of an audit.

## If implementation is explicitly requested

- Follow `CODEX_IMPLEMENTATION_HANDOFF.md`.
- Implement Phase 1 only and stop.
- Keep legacy source trees read-only.
- Use fake backends and dry runs.
- Do not add GUI, batch generation, public sharing, or model downloads.



