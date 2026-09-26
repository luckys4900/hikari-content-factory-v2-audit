# HIKARI CONTENT FACTORY v2 — 監査・設計報告書

作成日: 2026-09-26  
監査方式: read-only（既存ファイルの変更なし）  
対象:

- `${NANASE_SOURCE_ROOT}`
- `${HIKARI_SOURCE_ROOT}`

## A. EXECUTIVE SUMMARY

### 結論

「七瀬なな」は成人向け生成そのものが拒否されて失敗したのではない。成人向けプロンプトとLoRAは作動し、画像生成も成立していた。失敗の本質は、次の条件を同時に満たせなかったことにある。

1. 顔と身体の同一性を維持する。
2. 元衣装・元latentの支配を破って衣装・身体表現を変更する。
3. ポーズを変えても腕・手・胴体を破綻させない。
4. start/endを同一人物のI2V入力として成立させる。
5. 12GB VRAM / 約32GB RAMで安全に完走する。

最終状態はクラッシュではなく、品質ゲートによる `NO_ACCEPTABLE_CANDIDATE` だった。AIO Q4経路はRAM安全停止、軽量Qwen 2511経路はsource latent / garment authority、body geometry、貼り付け型face composite、弱いanatomy gateで行き詰まった。I2V用10枚の目標に対し、後期レビューで正式採用は2枚、その後の画像SHA紐付け再監査では0/10となった。

ヒカリの44画像は単一人物の整ったidentity packではなく、複数の顔・衣装・体型クラスタ、別人に見える写真、グループ写真、コンタクトシートが混在する。現時点で最も一貫した候補はレース衣装クラスタ6枚で、顔埋め込みの組内平均0.780、最小0.682だった。これは選別の補助であり、成人向け用途の許諾や架空性を証明しない。したがってv2は、最初に出所・成人年齢・生成物または利用許諾・許可された変換を記録する `subject_provenance.json` を必須化し、未確認なら生成を開始しない。

推奨方針は、旧Qwen全身再生成を移植せず、モデル交換可能な縦切りPoCにすること。静止画1枚を合格させ、同じ静止画から15秒I2Vを3 seedだけ生成し、1本以上が合格して初めてGUI・大量バッチへ進む。現行ComfyUIにはI2Vモデル、IP-Adapter、CLIP Vision、専用ヒカリLoRAがなく、動画バックエンドは未導入である。

## B. REPOSITORY MAP

七瀬なな配下は約44,342ファイル、約9.82GB。中心の `glm_pipeline` は約9.1GB。`local_generation/tests` に70個のテストファイル、`scripts` に71個のPythonスクリプト、workflow JSON 5個、config 21個がある。Git作業ツリーには未コミット変更・未追跡物が多いため、v2は旧ツリーへ直接実装しない。

| 区分 | 主要項目 | 役割 / 監査所見 |
|---|---|---|
| GUI | `launcher/nanase_generator_gui.py` | 10,941行のモノリス。機能は多いがPoCの起点にはしない。 |
| ジョブ | `generation_job.py`, `generation_queue.py`, `job_router.py` | 状態・キュー・ルーティング。`GenerationJob`相当が重複しており統合が必要。 |
| 出力 | `output_path_manager.py` | run単位の出力・metadata分離。再利用価値が高い。 |
| 自然言語 | `natural_language_compiler.py`, `prompt_compiler.py`, `prompt_card.py` | 自然言語→構造化指示の土台。adult/SFW衝突検知を追加する。 |
| identity/body | `body_profile.py`, `body_anchor_manager.py`, `face_identity_gate.py`, `body_geometry_analysis.py` | 七瀬固有値を外し、複数reference ensembleへ置換する。 |
| anatomy/I2V | `i2v_anatomy_gate.py`, `i2v_reference_authority.py` | 骨格検出だけでは画像破綻を見逃す。画像SHAに結び付いた人間確認が必要。 |
| pose | `reference_pose_control.py` | pose mapだけをconditioningへ渡す設計は再利用可。元画像をgeneration sourceにしない。 |
| Qwen runtime | `direct_huihui_runtime.py`, `direct_source_independent_core.py` | current routeは凍結・棄却。whole-body主経路としては廃止。 |
| region edit | `generic_region_i2i.py`, `identity_preserving_masked_inpaint.py` | 前者は3,793行で分割が必要。後者は次設計の概念的起点。 |
| recipe/series | `good_recipe_store.py`, `series_manager.py` | seed/model/workflow/QC結果の追跡へ拡張して再利用。 |
| workflow/config | `workflows/*.json`, `config/*.json` | 旧設定は証拠として固定。ヒカリv2は別ディレクトリ・別schemaにする。 |
| reports | `output/metadata/**`, `output/nanase_direct_huihui/**` | RAM停止、authority、identity/body/anatomy判定の一次証拠。 |

### 実行フローの要約

旧系は、自然言語指示→prompt/pose/body設定→ComfyUI workflow→候補生成→顔・身体・anatomy判定→I2V用start/end選定→metadata/report保存という形だった。しかし、generation sourceとpose referenceの権限が混線し、全画面再生成・tight face composite・contact sheet利用が混入した。後期にhash pinningとpose-map-onlyへ改善したが、body変更とidentity維持の同時成立には届かなかった。

## C. FAILURE ROOT CAUSE TABLE

| 原因 | 症状 | 証拠 | 再発防止 |
|---|---|---|---|
| AIO Q4のRAM過負荷 | 416×624でもfree RAMが1.73GBまで低下し安全停止 | `final_recipe_*_lowram_retry_report.md`: max RAM 30,743.7MB / min free 1,734.4MB / VRAM 11,696MB | このPCではAIO経路をPoC候補から外す。preflightと3GB hard-stopを共通化。 |
| Qwen source latent / garment authority | 元衣装が残る、body変更が通らない | `failure_classification.json`: `SOURCE_LATENT_AUTHORITY_TOO_STRONG`; source garment ratio 0.130441 | whole-body img2imgを主経路にしない。identity conditioning + pose + masked inpaintを分離。 |
| latent中和の破綻 | ピンクの平坦な貼り付け領域、胴体崩壊 | `final_architecture_report.md`: `CURRENT_QWEN_ARCHITECTURE_REJECTED_LATENT_COMPATIBILITY` | `direct_source_independent_core`を廃止し、独立rendererへ切替。 |
| full-frame生成 | 顔・身体identityが低下 | reference authority監査、後期identity/body review | 生成範囲を明示し、顔・身体referenceを別roleでhash固定。 |
| tight face composite | 顔類似度は上がるが貼り付け感 | similarity 0.778まで上昇した一方、pose/body不成立 | compositeは診断用途のみ。最終候補を合格させる補修手段にしない。 |
| pose indexの誤用 | pose人物がgeneration sourceへ混入 | `nanase_reference_authority_fix_report.md` | pose画像は骨格mapだけを使用。reference roleをschemaと型で分離。 |
| contact sheetのstart/end利用 | 複数画像がI2V authorityになる | Pattern05の監査記録 | collage/contact sheet/multi-subjectは入力時にhard reject。 |
| SFW文言の混入 | adult targetと `natural coherent SFW candid photograph` が衝突 | reference authority監査 | prompt linterで矛盾語を検出しcompileを停止。 |
| anatomy gateが骨格偏重 | 腕・手の破綻を通常の2-arm skeletonとしてPASS | Pattern01 anatomy再監査 | skeleton gateは補助。画像SHAに結び付けたpixel reviewと手・腕crop確認を必須化。 |
| 量産ゲート未達 | 10枚目標に2枚、再監査で0枚 | `identity_body_nsfw_regeneration_report.md`, anatomy report | まず1静止画→3動画の縦切り。失敗を増産しないstop conditionを設ける。 |
| identity固定手段が未導入 | FaceID/IP-Adapter/専用LoRAは調査止まり | `docs/ip_adapter_faceid_investigation.md`; clip_vision/ipadapter不在 | backend選定と最小比較実験をPoCの明示タスクにする。 |
| 現行I2V backend不在 | 15秒動画を生成できるローカル経路がない | ComfyUI model/node inventory | adapter interfaceを先に作り、承認後に1モデルだけ導入して3s→5s→15sと検証。 |

## D. REUSE / REWRITE MATRIX

| 判定 | 資産 | 対応 |
|---|---|---|
| 再利用可 | `output_path_manager.py`のrun分離思想 | immutable run directoryとmanifestへ継承。 |
| 再利用可 | job/queue/routerの概念 | 重複型を解消して小さなstate machineへ。 |
| 再利用可 | prompt card / recipe store | schema version、model/ref hash、failure reasonを追加。 |
| 再利用可 | pose-map-only制御 | 参照画像そのものをrendererへ渡さない契約をテスト。 |
| 再利用可 | resource guard | 3GB RAM hard-stop、VRAM OOM、解放確認を追加。 |
| 再利用可 | face/body QCの基礎 | ヒカリensembleに再校正し、単一anchor依存を廃止。 |
| 部分修正 | natural language / prompt compiler | `content_plan.json` と `motion_plan.json` を出力し、矛盾lintを追加。 |
| 部分修正 | anatomy gate | pose skeleton + image crop + human SHA reviewの三層化。 |
| 部分修正 | `generic_region_i2i.py` | mask、workflow、QC、metadataへ分割。 |
| 部分修正 | GUI | 凍結。PoC成功後にbackend APIの薄いclientとして再設計。 |
| 部分修正 | Qwen 2511 | 顔・局所補正の比較候補に限定。whole-bodyの既定にしない。 |
| 廃止 | Qwen AIO v19 Q4経路 | 現行32GB RAM環境では既定routeにしない。 |
| 廃止 | direct source-independent / latent neutralization | 実証済みのbody geometry破綻経路。 |
| 廃止 | tight face compositeによる最終合格 | 診断画像以外には使用しない。 |
| 廃止 | pose indexをsourceにする旧script | schemaで表現不能にする。 |
| 廃止 | contact sheet start/end | hard reject。 |
| 新規 | Hikari identity backend adapter | 専用LoRAまたはFaceID/IP-Adapter系を比較できる境界。 |
| 新規 | I2V backend adapter | model固有workflowをcoreから分離。 |

## E. HIKARI IMAGE AUDIT

### 全体

44画像（PNG 40 / JPG 4、約95.3MB）を全件inventory化し、contact sheet目視、最大顔検出、顔埋め込み類似度、用途別分類を実施した。完全重複・近似重複は検出されなかった。背景の人物・機械の顔が検出される場合があるため、自動値だけでは採否を決めていない。

| 区分 | 枚数 | index |
|---|---:|---|
| canonical candidate | 5 | 5, 8, 25, 28, 29 |
| face-only | 6 | 4, 13, 14, 26, 27, 44 |
| body-only | 3 | 6, 38, 39 |
| reject | 30 | その他 |

### 推奨canonical pack

- primary full-body: index 5
- primary face: index 25
- supporting: index 8, 28, 29
- face support: index 44
- body-only support: index 6, 38, 39（顔identityには使わない）

レース衣装の6枚（5, 8, 25, 28, 29, 44）の顔類似度は平均0.779889、最小0.681574、p10=0.701426。PoCの仮ゲートは「canonical ensembleへのmedian ≥ 0.747819、かつ2参照以上とのminimum ≥ 0.701426」とする。ただし正例6枚だけから得た暫定値なので、意図的な別人negativeと人間合格例で検証するまでhard gateにしない。

### canonical body profile（言語化）

- 身長感: 平均〜やや高め。ヒール・広角の影響を分離して扱う。
- 肩幅: narrow-medium、ややなだらか。
- 胸部: full / high-setの印象。ただし画角依存が大きく、cup sizeを固定値化しない。
- 胴体: compact-medium、ウエストは明確。
- 腰・ヒップ: moderate。
- 太もも: medium〜moderately full。
- 脚長比: medium-long。生成側で過剰に伸ばさない。
- 髪: 長いストレートの暗色、薄めのfull bangs。
- 推奨画角: eye-to-chest level、50–85mm相当。高角度座位は補助referenceのみ。

### 入力禁止

- グループ写真、他人物が明確な画像、contact sheet、collage。
- 顔roleに指定されていないbody-only画像。
- 出所・成人年齢・許諾が確認できない画像。

詳細は `evidence/hikari_image_audit.csv`、`evidence/hikari_image_audit.json`、4枚のcontact sheet、類似度matrixを参照。

## F. HIKARI V2 ARCHITECTURE

### 原則

1. 旧プロジェクトと別rootに新設し、旧資産はimportせずadapter越しに参照する。
2. referenceは `face`, `body`, `pose`, `style`, `start_frame` のroleを型で分離し、hash pinningする。
3. 生成器、QC、I2Vを交換可能にし、モデル名をcore logicへ埋め込まない。
4. 自動スコアは採否補助。成人向け候補の最終anatomy passは画像SHA紐付け人間確認を要する。
5. すべての生成はimmutable manifestで再現可能にする。

### 状態遷移

```text
DRAFT
  → PROVENANCE_VERIFIED
  → ASSETS_LOCKED
  → STILL_CANDIDATES
  → STILL_ACCEPTED
  → I2V_PREFLIGHT
  → VIDEO_CANDIDATES
  → VIDEO_ACCEPTED
                 ↘ STOPPED(reason_code)
```

### コンポーネント

| 層 | 責務 |
|---|---|
| provenance gate | `subject_provenance.json`を検証。成人・synthetic/licensed・同意/許諾・許可変換が欠ければ停止。 |
| asset registry | canonical画像、role、SHA-256、寸法、cluster、使用禁止条件を固定。 |
| request compiler | 自然言語からscene/pose/camera/clothing/motion/durationを構造化。曖昧点はdefault化して記録。 |
| prompt linter | adult/SFW矛盾、未成年に見える語、複数人物、role違反を検出。 |
| still backend | identity conditioning + pose map + region/masked inpaint。Qwen全画面body routeは使用しない。 |
| still QC | identity ensemble、body profile、hair/camera、image anatomy、I2V suitability。 |
| I2V backend | accepted still 1枚を入力し、seed違い3本。coreはmodel非依存。 |
| video QC | duration/decode、first/mid/last＋定間隔frame、identity/body drift、anatomy、flicker、cutを評価。 |
| run store | model/workflow/ref hash/seed/prompt/QC/failure/manual reviewを追記不可のrun単位で保存。 |

### I2V候補

現環境には動画modelがない。最初の調査候補はLTX-Video 2B distilled/FP8とする。公式LTXリポジトリはI2V・multi-keyframe・video extensionを提供し、Diffusers公式ドキュメントにはmemory-optimized例が約10GB VRAMと記載されている。一方、Wan2.2 TI2V-5B公式model cardは720p/24fpsに少なくとも24GB VRAMを要求するため、RTX 4070 12GBの第一候補にはしない。

- LTX official: https://github.com/Lightricks/LTX-Video
- Diffusers LTX pipeline: https://huggingface.co/docs/diffusers/main/api/pipelines/ltx_video
- Wan2.2 TI2V-5B: https://huggingface.co/Wan-AI/Wan2.2-TI2V-5B

ただし「15秒を12GBで完走」は未検証であり保証しない。明示承認後に1モデルだけ導入し、3秒→5秒→15秒の順にベンチマークする。必要ならchunk生成＋extensionをbackend内部で行う。

### manifest最小項目

```json
{
  "schema_version": "2.0",
  "run_id": "...",
  "request": {"raw": "...", "content_plan": "...", "motion_plan": "..."},
  "models": [{"role": "still|i2v|identity|pose", "name": "...", "sha256": "..."}],
  "workflow_sha256": "...",
  "references": [{"path": "...", "role": "face|body|pose", "sha256": "..."}],
  "seeds": {"still": 0, "i2v": [0, 0, 0]},
  "resource_metrics": {},
  "qc": {},
  "manual_reviews": [],
  "status": "...",
  "failure_reasons": []
}
```

## G. FIRST POC PLAN

### Phase 0 — 凍結・出所確認

- 旧七瀬ツリーはread-onlyのまま維持。
- 新rootとschemaだけ作成。
- `subject_provenance.json`を人間が記入・承認。
- canonical registryをhash固定し、collage/group/別identityを除外。

### Phase 1 — 生成なしの基盤

- state machine、manifest、asset registry、prompt compiler/linter、resource guardを実装。
- fake backendでrun directoryとfailure reasonをテスト。
- GUI、バッチ、公開機能は作らない。

### Phase 2 — 静止画1枚

- identity backendを1〜2経路だけ比較する。専用ヒカリLoRAを長期本命、IP-Adapter/FaceID系をPoC比較候補とする。
- primary face/bodyを分離し、poseはpose mapのみ使用。
- 4候補以内を生成。1枚だけ合格させる。
- identity/body/anatomy/I2V suitabilityとhuman SHA reviewを通す。
- 同じ失敗分類が2回続いたらparameter sweepを停止し、設計見直し。

### Phase 3 — I2V 15秒を3本

- 動画backendの承認・導入後、accepted still 1枚で3秒、5秒を確認。
- RAM/VRAM/時間がgate内なら15秒をseed違いで3本だけ生成。
- start/endを別人物・contact sheetで補わない。
- 全3本を同じQCで比較し、1本以上の正式採用を目標とする。

### Phase 4 — 判定

- 1本以上合格: PoC成功。次フェーズでのみ薄いGUIと小規模batchを検討。
- 0本合格: `STOPPED`。失敗理由、sample frame、resource logを保存し、モデルまたはmotion planのどちらを変えるか1点だけ決める。

## H. ACCEPTANCE CRITERIA / STOP CONDITIONS

### 静止画合格

- provenanceがverified。
- canonical registryの全入力がhash一致、single-subject、正しいrole。
- 成人1名、要求したscene/pose/clothingが成立。
- 顔はensemble暫定gateを満たす、または人間レビューで明示承認。
- body profileがcanonical範囲。脚の過長、胸部だけの過大化、肩幅ずれがない。
- 見えている腕・手・脚・胴体にfatal anomalyがない。
- 顔の貼り付け境界・肌色段差・局所平坦化がない。
- I2V向けに手・顔が読め、極端なocclusionや不安定な背景がない。
- 合格画像のSHA-256に対してhuman reviewが記録される。

### 動画合格

- 同一accepted stillからseed違いで exactly 3候補。
- 各候補がdecode可能、実時間15秒、音声要否がmanifestに明記。
- sampled framesで顔・髪・肩・胸部・腰・脚比率が安定。
- fatalな手・腕・脚の増減、身体融合、急な衣装変化、scene cutがない。
- flickerとcamera motionがrequest範囲内。
- 少なくとも1本がhuman review込みでaccepted。
- model/workflow/ref hash/seed/prompt/resource/QC/failureが完全記録。

### 即時停止

- provenance未確認、未成年に見える指示、無断の実在人物加工。
- group/contact sheet/別identityをauthority入力として検出。
- promptにadult/SFWなどの意味衝突。
- preflightでfree RAM < 12GBまたは想定VRAM不足。
- 実行中free RAM < 3GB、VRAM OOM、pagefile飽和、process hang、model unload失敗。
- 同一failure classが2回連続。
- identityは通るがbody/anatomyが落ちる候補を補修合成で無理に昇格。
- 15秒以前の3秒/5秒smokeが失敗。

## I. CODEX IMPLEMENTATION HANDOFF

### 実装ロードマップ

1. 新規 `hikari_content_factory_v2` rootを作る。
2. schema、provenance、asset registry、state machine、run storeを先に実装。
3. fake backendとunit testsで「生成なし」縦切りを通す。
4. Hikari canonical registryを相対パス＋hashで作成する。ただし元画像はcopyしない。
5. prompt compiler/linterとrole validationを実装。
6. still backend interface、QC interface、I2V backend interfaceを定義する。
7. hardware inventory/preflightを実装し、既存modelを自動downloadしない。
8. Phase 1完了時点で停止し、差分・テスト・未解決事項を報告する。

### 推奨ディレクトリ

```text
hikari_content_factory_v2/
  pyproject.toml
  README.md
  config/
    resource_limits.yaml
    qc_thresholds.yaml
  schemas/
    subject_provenance.schema.json
    asset_registry.schema.json
    content_plan.schema.json
    motion_plan.schema.json
    run_manifest.schema.json
  src/hikari_factory/
    domain/
      states.py
      reason_codes.py
      models.py
    provenance/
      validator.py
    assets/
      registry.py
      hashing.py
    planning/
      request_compiler.py
      prompt_linter.py
    backends/
      still_base.py
      i2v_base.py
      fake.py
    qc/
      identity.py
      body.py
      anatomy.py
      i2v_suitability.py
      video.py
    runtime/
      preflight.py
      resource_guard.py
      run_store.py
      orchestrator.py
  canonical/
    hikari_registry.json
    body_profile.json
    face_similarity_calibration.json
  tests/
  runs/                 # gitignored
```

### 最初に必要なテスト

- provenance欠落・未成年・無許諾でhard stop。
- reference role違反、hash不一致、contact sheet/groupでhard stop。
- adult/SFW衝突をprompt linterが停止。
- pose reference pathがstill generation sourceへ渡らない。
- manifestにmodel/workflow/reference hash/seed/failure reasonが必ず残る。
- state transitionの不正順序を拒否。
- resource guardの境界テスト（12GB preflight、3GB runtime hard-stop）。
- fake backendでDRAFT→STILL_ACCEPTEDまで再現可能。
- accepted image SHAとhuman review SHAの不一致を拒否。

### Phase 1実行コマンドの期待形

```powershell
python -m pytest -q
python -m hikari_factory.cli audit-assets --registry canonical/hikari_registry.json
python -m hikari_factory.cli compile-request --text "..." --dry-run
python -m hikari_factory.cli run-still-poc --backend fake --dry-run
```

### Phase 1成果物

- 新rootのコードとテスト。
- `runs/<run_id>/run_manifest.json`
- `runs/<run_id>/content_plan.json`
- `runs/<run_id>/motion_plan.json`
- `runs/<run_id>/events.jsonl`
- preflight report。
- 実モデル未導入を示す `blocked_capabilities`。

### 未解決事項

1. ヒカリ44画像の出所・生成履歴・成人年齢・利用許諾。ユーザー申告だけでなくmanifest化が必要。
2. identity方式の比較: 専用LoRA、IP-Adapter/FaceID系、その他ローカル方式。
3. 12GB VRAMでの15秒I2V性能。LTX 2Bを第一ベンチ候補とするが実測前は未確定。
4. 15秒を単発生成するか、短いchunk＋extensionにするか。
5. body metricのcamera/perspective正規化。まずは人間レビュー併用が必要。

### 監査の境界

- 有料APIは使用していない。
- 既存の七瀬なな・ヒカリ配下は変更していない。
- 新モデルのdownload、生成実行、成人向け出力の作成は行っていない。
- 顔類似度は同一性の補助指標であり、本人確認・架空性・同意の証明ではない。





