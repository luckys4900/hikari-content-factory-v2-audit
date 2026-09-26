# Codexへ渡す「第一実装プロンプト」

```text
ROLE
You are the implementation engineer for HIKARI CONTENT FACTORY v2.

OBJECTIVE
監査済み設計に従い、実モデル生成をまだ行わず、Phase 1「生成なしの安全な最小縦切り」を実装してください。旧七瀬ななプロジェクトへ直接変更を入れず、新規rootにprovenance、canonical asset registry、request compiler/linter、state machine、immutable run manifest、resource preflight、backend interfaces、fake backend、unit testsを作ります。

READ FIRST
- HIKARI_CONTENT_FACTORY_V2_AUDIT.md
- evidence/hikari_image_audit.csv
- evidence/face_similarity_calibration.json

SOURCE DIRECTORIES — READ ONLY
- ${NANASE_SOURCE_ROOT}
- ${HIKARI_SOURCE_ROOT}

NEW IMPLEMENTATION ROOT
- ${HIKARI_V2_ROOT}

NON-NEGOTIABLE CONSTRAINTS
1. 既存ファイルを上書き・移動・削除しない。
2. 新モデルをdownloadしない。有料APIを使わない。ComfyUI生成をまだ実行しない。
3. 実在人物の無断性的加工、出所不明、未成年または未成年に見える対象を扱わない。
4. `subject_provenance.json` がverifiedでなければ、実backendは呼べない設計にする。
5. group photo、contact sheet、collage、別identity、hash不一致をauthority入力として拒否する。
6. pose referenceはpose mapのsourceにだけなり、still generation image sourceにはならないことを型とテストで保証する。
7. adult targetとSFW文言などの意味衝突をprompt linterで停止する。
8. Qwen AIO v19 Q4、direct source-independent latent neutralization、tight face composite、contact-sheet start/endを移植しない。
9. GUI、バッチ生成、公開機能を作らない。
10. Phase 1終了時に止まり、実モデル導入へ自動で進まない。

IMPLEMENTATION SCOPE
A. Project scaffold
- Python package `hikari_factory`
- pyproject.toml, README, .gitignore
- config, schemas, canonical, src, tests, runsの分離

B. Schemas
- subject_provenance.schema.json
- asset_registry.schema.json
- content_plan.schema.json
- motion_plan.schema.json
- run_manifest.schema.json
- schema_versionを全artifactへ入れる

C. Provenance gate
- adult age assertion
- synthetic_or_licensed_source
- consent_or_license_basis
- allowed_transformations
- reviewer, reviewed_at
- 不足、false、矛盾でreason code付きhard stop

D. Canonical registry
- 監査報告のindex 5, 8, 25, 28, 29をcanonical candidateとして登録
- index 44をface-only support
- index 6, 38, 39をbody-only support
- 元画像はcopyせず、絶対path、role、SHA-256、寸法、notesを記録
- reject画像をauthorityとして指定できない

E. Domain/state machine
- DRAFT → PROVENANCE_VERIFIED → ASSETS_LOCKED → STILL_CANDIDATES → STILL_ACCEPTED → I2V_PREFLIGHT → VIDEO_CANDIDATES → VIDEO_ACCEPTED
- 任意状態からreason code付きSTOPPEDへ
- 不正transitionは例外

F. Planning
- 自然言語を deterministic な `content_plan.json` と `motion_plan.json` へcompile
- scene, pose, camera, clothing, motion, duration_seconds=15, candidate_count=3を表現
- dry-runでは曖昧値をdefault化し、assumptionsへ記録
- adult/SFW、single/multiple subject、未成年表現などをlint

G. Backend boundaries
- StillBackendとI2VBackendのProtocol/ABC
- FakeStillBackendのみ実装
- 実backendは `blocked_capabilities` として報告
- reference roleをtyped objectで渡し、pose originalがgeneration sourceに入らない

H. Runtime/run store
- `runs/<run_id>/` を作成
- run_manifest.json, content_plan.json, motion_plan.json, events.jsonl, preflight.json
- model/workflow/reference SHA、seed、status、QC、manual review、failure reasonsを格納
- artifactは原則immutable。状態更新はevent append＋manifestの原子的置換

I. Resource preflight
- free RAM 12GB未満なら実backend不可
- runtime free RAM 3GB未満をhard stopとしてconfig化
- GPU/VRAM、ComfyUI path、I2V model/IP-Adapter/CLIP Vision有無をread-only inventory
- 自動download禁止

J. Tests
- provenance gateのpass/fail
- 未成年・無許諾のhard stop
- hash mismatch、wrong role、contact sheet/groupのreject
- prompt conflict lint
- pose reference isolation
- state transitions
- manifest completenessとevent append
- accepted image SHAとmanual review SHAの一致
- resource threshold boundaries
- fake backendでDRAFTからSTILL_ACCEPTEDまでのdry-run

EXPECTED COMMANDS
- python -m pytest -q
- python -m hikari_factory.cli audit-assets --registry canonical/hikari_registry.json
- python -m hikari_factory.cli compile-request --text "夜のホテル室内、自然な立ち姿、ゆっくり振り向く15秒動画" --dry-run
- python -m hikari_factory.cli run-still-poc --backend fake --dry-run

DEFINITION OF DONE
1. 全テストがpass。
2. 既存2フォルダのgit/file stateに変更がない。
3. dry-runが再現可能なrun directoryと完全なmanifestを作る。
4. 実モデル未導入は明示的blockとして報告され、自動downloadされない。
5. 変更ファイル一覧、テスト結果、dry-run成果物path、未解決事項を報告する。
6. Phase 2へ進まず停止する。

IMPLEMENTATION STYLE
- 小さく型の明確なmoduleに分割する。
- 旧10,941行GUIや3,793行region moduleのようなmonolithを作らない。
- model名やworkflow固有nodeをdomain層へ埋め込まない。
- reason codeを文字列の自由記述だけにせずenum化する。
- thresholdはconfigに置き、仮校正であることをmetadataへ記録する。
```




