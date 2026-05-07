# ComfyUI + Wan 2.2 + Qwen-Image クラウド先行/ローカル移行 方針書

| 項目 | 内容 |
| --- | --- |
| 文書ID | VIL-W1-02 |
| バージョン | 0.1 (Draft) |
| 作成日 | 2026-05-11 |
| 作業ブランチ | `claude/valdia-ip-lab-docs-ZvZtP` |
| 出力先 | `docs/valdia-ip-lab/comfyui-cloud-to-local-policy.md` |
| 対象フェーズ | Phase 0(クラウド検証) → Phase 1(ローカルRTX 4090移行) |
| 関連文書 | `cloud-gpu-evaluation-criteria.md`(指示書1)、`environment-manifest-template.md`(本指示書で同時作成) |

> **本書の位置づけ**: 指示書1で定めた選定基準のうち「環境再現性」を、ComfyUI を中心とした実装層で具体化する。Phase 0 でクラウド上に作った環境を、Phase 1 でローカル RTX 4090 24GB へ **同一ワークフローのまま** 移植できる運用ルールを定義する。
>
> **ライセンスについての注意**: 本書に記載のライセンス区分(Apache 2.0 等)はすべて **要公式確認**。実際の採用前に各モデルの公式リポジトリ・モデルカード・LICENSE ファイル(またはモデル配布元の利用規約)を確認し、本書の「ライセンス確認チェックリスト」(§4)に記入すること。本書は「採用候補と確認手順」を示すもので、ライセンス保証ではない。

---

## 1. 目的とスコープ

### 1.1 目的

VALDIA IP Lab の蒼龍検証(画像・動画・LoRA学習)を、Phase 0 ではクラウド GPU 上の ComfyUI で回し、Phase 1 でローカル RTX 4090 24GB へ移行する。両フェーズで **同一の workflow JSON / モデル / 依存関係 / プロンプト条件** が再現できる運用を定める。

### 1.2 環境再現性のゴール

> 「同じ workflow JSON + 同じモデル(同一ハッシュ) + 同じプロンプト + 同じ seed を、クラウドでもローカルでも、3か月後でも、別の作業者でも、再実行して **同等出力** が得られる」状態を作る。

これを支えるのが本書の運用ルールと `environment-manifest.md`(§13)である。

### 1.3 スコープ

- 対象: ComfyUI / Wan 2.2 / Qwen-Image / Z-Image / Ovi / LoRA学習(kohya_ss / AI-Toolkit)
- 対象外:
  - クラウドプロバイダ選定そのもの(指示書1)
  - 蒼龍のキャラクター設定(指示書3)
  - 蒼龍のプロンプト・監修基準(指示書4)

### 1.4 GPU 制約(逸脱禁止)

- VRAM **24GB で動作する構成のみ採用**(Phase 1 ローカル環境が RTX 4090 24GB のため)
- 48GB に最適化された構成を Phase 0 に持ち込むと、ローカル移行時に動かない構成が紛れ込む

---

## 2. 採用モデル候補一覧

> 全モデル **要公式ライセンス確認**。商用利用可(できれば Apache 2.0)を優先。ライセンス未確認のまま検証着手しないこと。確認手順は §4。

| 用途 | モデル | ライセンス想定(要公式確認) | VRAM 要件想定 | 公式URL確認方法 | 採否 |
| --- | --- | --- | --- | --- | --- |
| 動画生成 | **Wan 2.2** | Apache 2.0(要公式確認) | 24GB で動作する構成のみ採用 | 公式リポジトリ(GitHub or HuggingFace) の LICENSE / モデルカード | 候補(要ライセンス確認) |
| 画像生成 | **Qwen-Image** | Apache 2.0(要公式確認) | 24GB | 公式 HuggingFace モデルカード | 候補(要ライセンス確認) |
| 画像生成 | **Z-Image** | Apache 2.0(要公式確認) | 24GB | 公式リポジトリ | 候補(要ライセンス確認) |
| 音声付動画 | **Ovi** | 要公式確認 | 24GB で動作する構成のみ | 公式リポジトリ | 候補(ライセンス未確定の場合は採用保留) |
| LoRA 学習 | **kohya_ss** | Apache 2.0(要公式確認、本体はOSS) | 学習対象モデルに依存 | 公式リポジトリ LICENSE | 候補 |
| LoRA 学習 | **AI-Toolkit** | 要公式確認 | 学習対象モデルに依存 | 公式リポジトリ LICENSE | 候補(ライセンスにより採否決定) |

### 2.1 除外モデル(明示)

| モデル | 除外理由 |
| --- | --- |
| **FLUX.2 9B 系**(dev / pro 等) | 非商用ライセンス。VALDIA の商用検証では採用不可 |
| FLUX.2 4B 系 | 採用候補から除外しないが、本 Phase 0 では Wan 2.2 / Qwen-Image / Z-Image を優先し、必要時のみ追加検証 |

### 2.2 LoRA 学習対象モデルとライセンス連鎖

LoRA は「ベースモデル + 学習スクリプト + 学習素材」の3層からなる。

- **ベースモデル** のライセンスが LoRA の利用範囲を実質的に縛る(派生物条項に注意)
- **学習スクリプト** のライセンス(kohya_ss / AI-Toolkit)
- **学習素材** の権利(VALDIA 自社IP/天機-OS のみを Phase 0 では使用)

→ ベースモデルが商用 NG であれば、その LoRA は商用利用できない。**ベースモデル選定時に商用可を確認** することが第一義。

---

## 3. ComfyUI 配布形態の選択

### 3.1 候補

| 候補 | 説明 | 環境再現性 | クラウド適性 | ローカル適性 |
| --- | --- | --- | --- | --- |
| **ComfyUI 公式リポジトリ + 自前 venv / Docker** | 公式 GitHub から直接 clone | ◎(commit ピン留め可) | ◎(Docker image 化が直球) | ◎ |
| **ComfyUI Desktop**(公式デスクトップアプリ) | デスクトップアプリ版 | ○(バージョン記録は可だが内部更新がアプリ更新と連動) | △(クラウドGPU上で使う前提ではない) | ○(ローカルでGUI操作が楽) |
| **StabilityMatrix** | マルチアプリ管理ツール | △(StabilityMatrix 自体の更新がComfyUI環境にも影響) | △ | ○ |

### 3.2 採用方針

- **クラウド(Phase 0) = ComfyUI 公式リポジトリ + Docker image**(commit ハッシュ固定)
- **ローカル(Phase 1) = ComfyUI 公式リポジトリ + 自前 venv または同一 Docker image**
  - 補助ツールとして StabilityMatrix / ComfyUI Desktop を入れるのは可。ただし **検証用本体は自前リポジトリ管理側を正** とする
- 理由: Desktop / StabilityMatrix を「本体」にすると、アプリ更新で内部 ComfyUI バージョンが意図せず変わるリスクがある。再現性を最優先するため、**直接管理形態を本体とする**。

### 3.3 例外

- 個人作業者が手元検討で使う分には Desktop / StabilityMatrix を併用してよい
- ただし **VALDIA の納品・LoRA学習・Phase 移行検証** に使うワークフローは、必ず公式リポジトリ管理の本体側で実行する

---

## 4. ライセンス確認チェックリスト(モデル採用前に実施)

各モデルについて、採用前に以下を埋めて `environment-manifest.md` の「モデル」セクションに記録する。

- [ ] モデル名・バージョン・配布元(公式 GitHub or HuggingFace の URL)を記録した
- [ ] LICENSE ファイル または モデルカードに記載のライセンス名(Apache 2.0 / MIT / 独自 / 非商用 等)を記録した
- [ ] 商用利用の可否を確認した
- [ ] 派生物(LoRA / Fine-tune / 出力)に対するライセンス条項を確認した
- [ ] 帰属表示(Attribution)の要否と表記方法を確認した
- [ ] モデルファイルの SHA256 ハッシュを取得した
- [ ] 学習データに関する利用規約上の制約(例: 出力の特定用途禁止)を確認した
- [ ] アップデート時に旧バージョンが入手可能かを確認した(再現性のため)

### 4.1 ライセンス採用基準

| ライセンス | 採用可否 | 備考 |
| --- | --- | --- |
| Apache 2.0 / MIT / BSD | ◎ 第1優先 | 帰属表示を遵守すれば商用可 |
| CreativeML OpenRAIL-M | ○ 条件付き | 用途制限条項を読み、VALDIA の使い方が違反しないか確認 |
| 独自ライセンス(商用可) | ○ 条件付き | 全文を読み、契約レビュー |
| 非商用(NC) | × 不採用 | 商用検証では使わない |
| ライセンス不明 | × 不採用(確認後再評価) | 不明のまま使わない |

---

## 5. Phase 0: RunPod等クラウドGPU 環境構築手順

### 5.1 前提

- 指示書1の選定基準を満たすプロバイダ・インスタンスを確保済み
- データ取扱区分(指示書1 §3)を遵守(自社IP/天機-OS のみ)
- 永続ボリュームを暗号化前提で確保済み

### 5.2 構築手順(概要)

1. **Docker image 選定**
   - ベース: `nvidia/cuda:<X.Y.Z>-runtime-ubuntu<XX.XX>`(具体バージョンは検証時点のドライバ整合で決定し、`environment-manifest.md` に記録)
   - Python バージョンを image 内で固定(例: 3.11)
2. **ComfyUI 取得**
   - 公式リポジトリを clone
   - **commit ハッシュをピン留め** して checkout
   - `ComfyUI commit` を `environment-manifest.md` に記録
3. **依存関係インストール**
   - `requirements.txt` のバージョンをピン留め(`==`)
   - `pip freeze` で実環境のバージョン一覧を取得し `requirements.lock` として保存
4. **custom_nodes 取得**
   - 各 custom_node を **commit ハッシュでピン留め** して clone
   - `custom_nodes` 一覧(リポジトリURL + commit + ライセンス)を `environment-manifest.md` に記録
5. **モデル配置**
   - 永続ボリューム上の `models/` ディレクトリへ配置(後述 §9 のディレクトリ構造に従う)
   - 各モデルの SHA256 を取得し `environment-manifest.md` に記録
6. **workflow JSON 配置**
   - `workflows/` を VALDIA リポジトリ管理(永続ボリュームのみに依存しない)
7. **疎通確認**
   - 最小ワークフローで生成 1 枚を実行し、エラーなく完了することを確認

### 5.3 Docker 化の方針

- Dockerfile を VALDIA リポジトリ内 `docs/valdia-ip-lab/docker/Dockerfile.comfyui`(将来作成)に置き、image を **タグ付きでビルド・保管**
- image タグは `valdia-comfyui:<YYYYMMDD>-<commit短縮ハッシュ>` 形式
- 検証で使う image は `:latest` ではなく **このタグ指定で必ず固定**
- クラウドのレジストリに push するか、tarball をローカルに保存するかは秘匿性要件で判断(顧客IP取扱なら tarball を VALDIA 管理ストレージへ)

---

## 6. Phase 1 移行: ローカル RTX 4090 環境への切り替え手順

### 6.1 移行判定

指示書1 §11 の Phase 移行判定条件を **すべて満たす** ことを前提とする。

### 6.2 移行手順(概要)

1. **物理機調達 / セットアップ**
   - RTX 4090 24GB 中古ワークステーション
   - OS / ドライバ / CUDA を `environment-manifest.md` のクラウド側記録と合わせる
2. **Docker image の取り込み**
   - クラウドで使っていた image を tarball で持ち込み、`docker load`(または同レジストリから pull)
3. **永続データの取り込み**
   - `models/` / `custom_nodes/` / `workflows/` / LoRA / 生成ログ を VALDIA 管理ストレージから取得
   - 取得後、**SHA256 照合** で破損・改変なしを確認
4. **再現性検証(72時間以内)**
   - クラウドで作った workflow JSON を、ローカルで **同一プロンプト・同一 seed** で実行
   - 出力が同等(ピクセル完全一致は GPU 差で困難だが、構図・色味・蒼龍の特徴が監修基準で同等判定)であることを確認
   - 差分が出た場合: 原因調査(VRAM・CUDA・cuDNN・PyTorch・カスタムノード・乱数初期化)を `migration-issues.md`(必要時作成)に記録
5. **クラウド側のクリーンアップ**
   - 持ち出し完了後、クラウドの永続ボリューム削除・削除証跡保管(指示書1 §9.3)

### 6.3 移行が失敗する典型パターン

| 症状 | 原因の典型 | 防止策 |
| --- | --- | --- |
| 出力が大きく違う | seed 解釈差・PyTorch バージョン差・ノード commit 差 | バージョン・commit を厳密にピン留め |
| OOM(Out of Memory) | クラウドが 48GB 等の上位GPUだった | Phase 0 から 24GB 制約を徹底 |
| ノードがロードできない | カスタムノードの依存関係差 | `requirements.lock` を image に焼き込み、ローカルでも同一を使う |
| モデルが見つからない | パス・ファイル名差 | §9 ディレクトリ構造を厳守 |

---

## 7. 共通: 環境再現性管理(クラウド・ローカル両対応)

### 7.1 必須記録項目(`environment-manifest.md` に記録)

- 検証ID(YYYYMMDD-XX 形式)
- 実行環境(Cloud / Local の別、プロバイダ名、インスタンスID等)
- GPU(型番、VRAM、ドライババージョン)
- OS(ディストリ・バージョン)
- Docker image(タグまたは digest)
- Python バージョン
- CUDA バージョン
- PyTorch バージョン
- ComfyUI commit ハッシュ
- custom_nodes 一覧(リポジトリURL + commit + ライセンス)
- モデル一覧(ファイル名 + SHA256 + ライセンス + 公式URL)
- LoRA 一覧(ファイル名 + SHA256 + 学習元情報 + ライセンス)
- workflow JSON ファイル名 + リポジトリ内パス + commit
- プロンプト・negative・seed・生成条件
- 生成日時・出力先パス・出力ファイル一覧

### 7.2 検証ごとの運用

- 1 検証 = 1 `environment-manifest.md` を作成
- 保存先: `docs/valdia-ip-lab/manifests/<検証ID>/environment-manifest.md`
- テンプレートは本指示書で同時作成する `environment-manifest-template.md`

---

## 8. workflow JSON の保存ルール

| 項目 | ルール |
| --- | --- |
| 保存先 | VALDIA リポジトリ `docs/valdia-ip-lab/workflows/`(クラウド永続ボリュームのみに依存しない) |
| 命名規則 | `<用途>-<対象キャラ>-<バージョン>.json`(例: `image-seiryu-v0.3.json`、`video-seiryu-v0.1.json`) |
| バージョニング | git で管理。ノード追加・パラメータ変更ごとに minor を上げる |
| メタ情報 | 同名 `<...>.meta.md` を併置し、用途・前提モデル・想定VRAM・既知の注意点を記載 |
| 大規模変更 | ワークフローの構造を大きく変える際は新ファイル(`v0.x` → `v1.0`)とし、旧版を消さない |

---

## 9. ディレクトリ構造(永続ボリューム / ローカル共通)

```
<ROOT>/
├── ComfyUI/                     # commit ピン留めの公式リポジトリ
├── custom_nodes/                # 各 custom_node(commit ピン留め)
├── models/
│   ├── checkpoints/             # ベースモデル(.safetensors)
│   ├── loras/                   # LoRA ファイル
│   ├── vae/
│   ├── controlnet/
│   ├── clip/
│   └── upscale_models/
├── workflows/                   # workflow JSON(VALDIA リポジトリと同期)
├── outputs/
│   ├── images/<検証ID>/
│   └── videos/<検証ID>/
├── training/
│   ├── datasets/<対象キャラ>/   # 学習素材(自社IP/天機-OS のみ)
│   └── outputs/<検証ID>/        # 学習成果(LoRAファイル)
└── logs/
    └── generations/<検証ID>/    # プロンプト・seed・生成条件ログ
```

> クラウド・ローカルで **同一の相対パス** を使う。これにより workflow JSON 内の絶対パス依存を最小化する。

---

## 10. モデル名・バージョン・ハッシュ管理

### 10.1 ファイル命名規則

| 種別 | 命名規則 | 例 |
| --- | --- | --- |
| ベースモデル | `<モデル名>-<バージョン>-<量子化>.<拡張子>` | `qwen-image-v1.0-fp16.safetensors` |
| LoRA | `<対象キャラ>-<画風タグ>-<バージョン>-r<rank>.safetensors` | `seiryu-watercolor-v0.1-r16.safetensors` |
| VAE | `<モデル名>-vae-<バージョン>.safetensors` | `qwen-image-vae-v1.0.safetensors` |

### 10.2 ハッシュ管理

- 全モデルファイルの SHA256 を取得し、`environment-manifest.md` の「モデル」「LoRA」セクションに記録
- 取得方法: `sha256sum <file>` の結果を貼り付け(クラウド・ローカルで同一値であることを移行時に照合)
- LoRA 学習成果物は学習完了直後にハッシュ取得し、学習条件と紐付けて記録

### 10.3 モデル更新時の差し替え手順

1. **新モデルのライセンス再確認**(§4 チェックリスト全項目)
2. **既存検証への影響評価**: 旧モデルで作った workflow JSON が新モデルで動くか試行
3. **両バージョン併存期間**: `models/checkpoints/` 内に新旧両方を残す(2週間目安)
4. **`environment-manifest.md` 更新**: 新検証から新モデルを使う場合、新しい検証ID で manifest を作成(旧 manifest は不変)
5. **削除判断**: 新モデルでの再現が安定し、旧 manifest による再生成需要がなくなった時点で旧モデル削除

---

## 11. LoRA ファイル命名規則(再掲・詳細)

```
<対象キャラ>-<画風/用途タグ>-<バージョン>-r<rank>[-step<ステップ数>].safetensors
```

| 要素 | 例 | 備考 |
| --- | --- | --- |
| 対象キャラ | `seiryu` / `byakko` / `genbu` | キャラ単位で固定 |
| 画風/用途タグ | `watercolor` / `editorial` / `magazine-cover` | 指示書3・4 の世界観タグに準拠 |
| バージョン | `v0.1` / `v0.2` / `v1.0` | 学習試行ごとに minor を上げる |
| rank | `r8` / `r16` / `r32` | LoRA rank |
| step(任意) | `step3000` | 同一試行内で step 違いを比較するときのみ |

例:
- `seiryu-watercolor-v0.1-r16.safetensors`
- `seiryu-editorial-v0.3-r32-step5000.safetensors`

---

## 12. プロンプト・seed・生成条件ログ

### 12.1 ログ形式

各生成について、以下を JSON または Markdown で記録し、`logs/generations/<検証ID>/` に保存。

```json
{
  "generation_id": "20260512-001",
  "verification_id": "20260512",
  "datetime": "2026-05-12T10:23:45+09:00",
  "workflow_json": "workflows/image-seiryu-v0.3.json",
  "model": {
    "checkpoint": "qwen-image-v1.0-fp16.safetensors",
    "checkpoint_sha256": "...",
    "loras": [
      {"file": "seiryu-watercolor-v0.1-r16.safetensors", "weight": 0.8, "sha256": "..."}
    ],
    "vae": "qwen-image-vae-v1.0.safetensors"
  },
  "prompt_positive": "...",
  "prompt_negative": "...",
  "seed": 1234567890,
  "steps": 30,
  "cfg": 5.5,
  "sampler": "euler",
  "scheduler": "normal",
  "width": 1024,
  "height": 1820,
  "output_files": ["outputs/images/20260512/seiryu_001.png"],
  "review": {
    "checked": false,
    "result": null,
    "notes": ""
  }
}
```

### 12.2 ログ取得の仕組み

- ComfyUI のメタデータ機能(PNG メタ)を有効にしたうえで、生成ごとに上記 JSON を別途書き出す
- 自動化が難しい場合は、検証ごとに最低限の手作業ログを Markdown で残す

---

## 13. environment-manifest.md の運用

- テンプレート: `docs/valdia-ip-lab/environment-manifest-template.md`(本指示書で同時作成)
- 1 検証 = 1 manifest
- 保存先: `docs/valdia-ip-lab/manifests/<検証ID>/environment-manifest.md`
- 検証 ID 形式: `YYYYMMDD-XX`(例: `20260512-01`)
- 必須記載は §7.1 の通り

---

## 14. 商用本番運用時の注意事項

| # | 注意事項 |
| --- | --- |
| 1 | 全採用モデル・custom_node のライセンスを再確認(本書 §4 のチェックリスト)。Phase 0 検証時の確認に加え、本番運用直前に **再確認** する |
| 2 | 帰属表示(Attribution)が必要なライセンスについて、納品物・公開物への表記方針を法務・広報と合意 |
| 3 | LoRA を商用納品する場合、**ベースモデルのライセンス連鎖** を顧客契約書に記載 |
| 4 | 顧客IP素材を扱う案件は原則ローカル RTX 4090(指示書1 §3.2)。Secure Cloud を使う場合の許諾・削除証跡条件を契約書に明記 |
| 5 | モデル・custom_node の更新は本番運用中は **バッチで** 行わない。検証 → 本番反映の段階を踏む |
| 6 | 万が一ライセンス違反が判明した場合の **緊急差し戻し手順**(該当モデル・成果物の利用停止、顧客通知、代替モデルへの再生成)を年初に演習する |
| 7 | 生成物の倫理レビュー(商品化適性チェック・指示書4 §)を運用に組み込む |

---

## 15. 改訂履歴

| 日付 | バージョン | 変更内容 | 作成者 |
| --- | --- | --- | --- |
| 2026-05-11 | 0.1 | 初版作成(Draft) | Claude (指示書2) |

---

## 16. 関連文書

- `cloud-gpu-evaluation-criteria.md`(指示書1)
- `environment-manifest-template.md`(本指示書で同時作成)
- `characters/seiryu.md`(指示書3、予定)
- `prompts/seiryu-prompts.md`(指示書4、予定)
- `prompts/seiryu-review-checklist.md`(指示書4、予定)
