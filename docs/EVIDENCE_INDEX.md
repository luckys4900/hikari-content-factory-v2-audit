# Evidence index

監査対象はread-only。以下は結論に直接使った主要一次証拠です。

## Resource / AIO

- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\metadata\local_qwen_aio_nsfw\final_recipe_aio_regenerate_lowram_retry\final_recipe_20260709_193128_708f98_aio_regenerate_lowram_retry_report.md`
  - 416×624、max RAM 30,743.7MB、min free RAM 1,734.4MB、safety stop。
- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\metadata\local_qwen_aio_nsfw\identity_body_validation\identity_body_validation_report.md`
  - AIO Q4がcanonical body anchorに不適との比較。

## Latest Qwen body architecture

- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\nanase_direct_huihui\2026-08-12\20260812_154905_30defbed\final_architecture_report.md`
  - `CURRENT_QWEN_ARCHITECTURE_REJECTED_LATENT_COMPATIBILITY`、masked inpaintへのpivot。
- 同runの `runtime_status.json`, `failure_classification.json`, `candidate_gate.json`
  - `NO_ACCEPTABLE_CANDIDATE`、source latent authority、body/anatomy failure。

## I2V identity/body/anatomy

- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\metadata\local_qwen\i2v_identity_body_natural_refine_final_review_20260802\identity_body_nsfw_regeneration_report.md`
  - 正式採用2/10、Pattern03〜05未実行、full-frameとtight compositeの問題。
- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\metadata\local_qwen\i2v_pattern01_anatomy_regen_20260802\final_review\pattern01_anatomy_regeneration_report.md`
  - 画像SHA紐付け再監査で0/10、骨格gateの取りこぼし。

## Reference authority

- `${NANASE_SOURCE_ROOT}\glm_pipeline\output\metadata\local_qwen\nanase_reference_authority_audit_20260802_115100\nanase_reference_authority_fix_report.md`
  - pose indexのsource誤用、Pattern05 contact sheet、SFW語混入、full-frame identity低下。
- 同runの `nanase_reference_authority_fix_results.json`
  - raw similarity 0.550、tight composite 0.778だがpose不成立。

## Identity tooling / pose

- `${NANASE_SOURCE_ROOT}\glm_pipeline\docs\ip_adapter_faceid_investigation.md`
  - IP-Adapter/FaceID/InsightFace/CLIP Vision、専用LoRAが本導入されていない。
- `${NANASE_SOURCE_ROOT}\glm_pipeline\docs\REFERENCE_POSE_CONTROL_AUDIT.md`
  - SDPose＋Qwen ControlNet、pose-map-only経路。
- `${NANASE_SOURCE_ROOT}\glm_pipeline\docs\KNOWN_LIMITATIONS.md`
  - 32GB RAM / RTX 4070 12GBの既知制約。

## Configuration

- `${NANASE_SOURCE_ROOT}\glm_pipeline\config\direct_huihui_runtime.json`
  - current architecture frozen、replacement masked inpaint。
- `${NANASE_SOURCE_ROOT}\glm_pipeline\config\nanase_i2v_reference_authority_v1.json`
  - hash-pinned authority、contact sheet / pose identity / low-similarity禁止。

## Hikari audit artifacts

- Local-only inventory — 44画像の寸法・hash・ファイル情報。プライバシー保護のため非公開。
- Local-only classification — canonical / face-only / body-only / reject。公開版は `data/audit_manifest.json` のindex集計のみ。
- Local-only face audit / pairwise matrix — 顔特徴量を含むため非公開。
- `data/face_similarity_calibration.json` — 個別埋め込みを含まない暫定PoC集計値。
- Contact sheets — 全件目視に使用したが、プライバシー保護のため非公開。




