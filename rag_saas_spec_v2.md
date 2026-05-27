# 中小製造業向け RAG SaaS — 包括設計仕様書

**バージョン**: 1.0  
**対象フェーズ**: PoC（1〜3社・パイロット）  
**作成日**: 2026-05-23  

---

## 目次

1. [サービス概要](#1-サービス概要)
2. [技術スタック](#2-技術スタック)
3. [アーキテクチャ](#3-アーキテクチャ)
4. [データモデル](#4-データモデル)
5. [API 設計](#5-api-設計)
6. [チャンク設計](#6-チャンク設計)
7. [図面分析パイプライン（Gemini Vision）](#7-図面分析パイプラインgemini-vision)
8. [セキュリティ設計](#8-セキュリティ設計)
9. [引用表示・PDFビューア設計](#9-引用表示pdfビューア設計)
10. [画面構成](#10-画面構成)
11. [暗黙知のデータ化](#11-暗黙知のデータ化)
12. [フィードバック・間違い報告設計](#12-フィードバック間違い報告設計)
13. [登録サービス一覧](#13-登録サービス一覧)
14. [PoC サンプルデータ](#14-poc-サンプルデータ)
15. [開発ロードマップ](#15-開発ロードマップ)
16. [PoC 後に判断する事項](#16-poc-後に判断する事項)

---

## 1. サービス概要

中小製造業（従業員 20〜300 名、IT 専任担当なし）を対象に、社内に散在する PDF・Excel・スキャン図面・暗黙知を自然言語で検索・質問できるチャットボット型 SaaS。

### 1-1. 解決する課題

- 図面・仕様書・検査記録が紙・PDF 散在しており、必要情報へのアクセスに時間がかかる
- ベテランの暗黙知が属人化しており、退職時にナレッジが消える
- 専任 IT 担当がいないため既存のエンタープライズ RAG ツールの導入が困難

### 1-2. 競合との差別化

| サービス | ターゲット | 中小製造業への弱点 | 本サービスの優位性 |
|---|---|---|---|
| Azure AI Search | IT 部門のある大企業 | 開発スキル必須・業種特化なし | ノーコード・製造業ドメイン特化 |
| ORION (SparkPlus) | 大手〜中堅製造業 | エンタープライズ価格帯 | 中小向け価格・伴走サポート |
| Dify / Flowise | エンジニア向け | ノンテクニカルには設定困難 | 即日利用・業種知識込み |
| NotionAI 等 | オフィスワーカー全般 | 製造業文書・図面対応弱 | 図面・帳票・スキャン対応標準 |

### 1-3. セキュリティ方針

ブラウザベースのクラウド構成を基本とする。「クラウドは危険」という懸念の多くは感覚的なもので、既にクラウド会計・クラウド ERP を利用している中小製造業には説明で解消できる。防衛・航空宇宙など本当に機密性が高い案件はオプションのローカル LLM 構成で対応する（PoC 後に判断）。

---

## 2. 技術スタック

| レイヤー | 採用技術 | 備考 |
|---|---|---|
| フロントエンド | Next.js (React / TypeScript) | Vercel にデプロイ |
| バックエンド API | Python + FastAPI | Render / Railway にデプロイ |
| RAG フレームワーク | LlamaIndex | PDF・Excel・画像の読み込み対応 |
| LLM | Gemini 2.0 Flash | マルチモーダル・日本語・無料枠あり |
| 図面分析 | Gemini Vision + PyMuPDF | スキャン図面の寸法・公差・記号を読み取り |
| Embedding | text-embedding-3-small (OpenAI) | 1536 次元・日本語精度◎ |
| ベクトル DB | Qdrant | ローカルは Docker、本番は Qdrant Cloud |
| リレーショナル DB | Supabase (PostgreSQL) | 認証・RLS・テナント管理・Vision キャッシュ |
| ファイルストレージ | Cloudflare R2 | Egress 無料・10GB/月無料枠 |
| OCR（オプション） | Gemini Vision | スキャン図面対応・必要時に追加 |

### 2-1. PoC 月額コスト試算

1〜3 社・月間クエリ 1,000 回・文書 500 ページ想定

| 項目 | 費用 |
|---|---|
| Gemini 2.0 Flash (LLM) | ~$3〜8/月 |
| text-embedding-3-small | ~$1/月 |
| Qdrant（セルフホスト） | $0 |
| Supabase（無料枠） | $0 |
| Cloudflare R2（10GB 以内） | $0 |
| Render / Railway（Starter） | $0〜7/月 |
| Next.js（Vercel 無料枠） | $0 |
| **合計** | **約 $5〜16（800〜2,400 円）** |

### 2-2. スケールパス

| レイヤー | PoC | 本番移行後 | 難易度 |
|---|---|---|---|
| LLM | Gemini 2.0 Flash | Claude / GPT-4o / ローカル LLM | 低（抽象化しやすい）|
| Embedding | text-embedding-3-small | 多言語特化モデルへ変更も | 中（再インデックス必要）|
| ベクトル DB | Qdrant（Docker） | Qdrant Cloud | 低（互換あり）|
| インフラ | Render / Railway | AWS ECS / GCP Cloud Run | 低（Docker 化済みなら容易）|
| テナント分離 | Supabase RLS | そのまま継続 or 自前 DB へ | **高（設計変更は大改修）** |
| チャンク戦略 | デフォルト設定 | 製造業文書に最適化が必要 | **高（精度に直結）** |

---

## 3. アーキテクチャ

### 3-1. データ準備フロー（インデックス登録）

```
Raw Data（PDF / Excel / 画像）
  → [A] Cloudflare R2 に原本保存
  → [B] LlamaIndex でパース・テキスト抽出
  → [C] 図面の場合: Gemini Vision で画像全体を分析（寸法・公差・表面粗さ・幾何公差）
  → [D] チャンク分割（chunk_size=512, overlap=50）
  → [E] text-embedding-3-small でベクトル化
  → Qdrant に保存（tenant_id をペイロードに付与）
  → Supabase の files テーブルに r2_key / qdrant_id / vision_cache を記録
```

### 3-2. 質問・回答フロー（RAG）

```
ユーザーの質問（Next.js UI）
  → [1] text-embedding-3-small でクエリをベクトル化
  → [2] Qdrant で類似検索（tenant_id フィルタ必須・FastAPI 側で強制付与）
  → [3] 類似度スコア 0.75 未満のチャンクを除外（足切り）
  → [4] Gemini 2.0 Flash でプロンプト生成・回答
  → [5] 回答 + 引用情報（ファイル名・ページ・引用箇所・Presigned URL）を返却
```

### 3-3. R2 バケット構造

```
mfg-rag-prod/
  {tenant_id}/
    originals/              # アップロード原本
      {year}/{month}/
        {file_id}_{filename}
    processed/              # LlamaIndex 解析済みテキスト（任意）
```

> **テナント分離の原則**: R2 キーのプレフィックス = Supabase の `tenant_id`。  
> バグでアクセス制御が外れても、別テナントのファイルに物理的に触れない。

---

## 4. データモデル

### 4-1. Supabase テーブル

```sql
-- テナント（会社）
tenants (
  id          uuid PRIMARY KEY,
  name        text,
  created_at  timestamptz
)

-- ユーザー
users (
  id          uuid PRIMARY KEY,
  tenant_id   uuid REFERENCES tenants(id),
  email       text,
  role        text              -- admin / member
)

-- 登録ファイル
files (
  id                   uuid PRIMARY KEY,
  tenant_id            uuid REFERENCES tenants(id),
  filename             text,
  r2_key               text,    -- R2 のオブジェクトキー
  doc_type             text,    -- drawing / spec / manual / knowledge / other
  page_count           int,
  status               text,    -- uploading / indexing / ready / error
  vision_cache         jsonb,   -- Gemini Vision 分析結果キャッシュ
  vision_cached_at     timestamptz,
  vision_cache_version text,    -- モデル変更時の無効化用（例: "gemini-2.0-flash-v1"）
  created_at           timestamptz
)

-- 暗黙知ナレッジカード
knowledge_cards (
  id            uuid PRIMARY KEY,
  tenant_id     uuid REFERENCES tenants(id),
  category      text,           -- 工程カテゴリ（例: 溶接, 切削）
  condition     text,           -- どんな条件のとき
  action        text,           -- 何をするか
  reason        text,           -- なぜ
  caution       text,           -- 注意点・よくあるミス
  exceptions    text,           -- 例外条件
  source_type   text,           -- interview / incident / document
  contributor   text,
  verified_by   text,
  confidence    text,           -- high / medium / unverified
  r2_key        text,           -- 関連ファイルがあれば
  created_at    timestamptz
)

-- 間違い・フィードバック報告
feedbacks (
  id            uuid PRIMARY KEY,
  tenant_id     uuid REFERENCES tenants(id),
  query         text,
  answer        text,
  citation_id   text,           -- Qdrant の point ID
  type          text,           -- wrong_answer / outdated / missing_info
  comment       text,
  reported_by   uuid REFERENCES users(id),
  status        text,           -- pending / reviewing / resolved / rejected
  reviewed_by   uuid REFERENCES users(id),
  resolution    text,
  created_at    timestamptz,
  resolved_at   timestamptz
)
```

### 4-2. Qdrant ペイロード構造

```json
{
  "tenant_id":     "tenant_abc123",
  "text":          "図番: H01  図名: ピストン  材質: AC8A-T5  外径: 85mm  公差: +0/-0.02 ...",
  "r2_key":        "tenant_abc123/originals/2024/05/H01_Piston.pdf",
  "file_id":       "file-uuid-5678",
  "filename":      "H01_Piston.pdf",
  "page":          1,
  "chunk_index":   0,
  "doc_type":      "drawing",
  "figure_number": "H01",
  "figure_name":   "ピストン",
  "materials":     ["AC8A-T5", "SCM415"],
  "part_names":    ["ピストン", "ピストンピン"],
  "confidence":    "high",
  "raw_analysis":  "{...Gemini Vision の生 JSON...}"
}
```

---

## 5. API 設計

### 5-1. エンドポイント一覧

| Method | Path | 説明 |
|---|---|---|
| POST | `/api/files/upload` | ファイルアップロード → R2 保存 → インデックス登録 |
| GET | `/api/files` | 登録ファイル一覧取得 |
| DELETE | `/api/files/{file_id}` | ファイル削除（R2・Qdrant・Supabase 3 点セット） |
| POST | `/api/chat` | 質問 → RAG → 回答返却 |
| GET | `/api/files/{file_id}/presigned` | Presigned URL 取得（`#page=N` 付き） |
| POST | `/api/knowledge` | ナレッジカード登録 |
| GET | `/api/knowledge` | ナレッジカード一覧 |
| POST | `/api/feedback` | 間違い・フィードバック報告 |
| GET | `/api/admin/feedback` | フィードバック一覧（管理者用） |
| PATCH | `/api/admin/feedback/{id}` | フィードバックステータス更新 |

### 5-2. チャットレスポンス形式

```json
{
  "answer": "品番 A-1234-B の材質は SUS304、板厚は 2.0mm です。",
  "citations": [
    {
      "filename":      "仕様書_A1234.pdf",
      "page":          12,
      "quote":         "材質: SUS304  板厚: 2.0mm",
      "presigned_url": "https://xxxx.r2.cloudflarestorage.com/...#page=12",
      "score":         0.94
    }
  ],
  "confidence": "high",
  "warning":    null
}
```

### 5-3. プロンプトテンプレート

```
あなたは製造業の社内情報検索アシスタントです。
以下の社内文書のみを根拠として質問に答えてください。
文書に記載がない情報は「文書に記載がありません」と答えてください。

## 参照文書
{context}

## 質問
{query}

以下の JSON 形式で回答してください:
{
  "answer": "回答文",
  "citations": [
    {
      "filename": "ファイル名",
      "page": ページ番号,
      "quote": "根拠となった原文（30 文字以内）"
    }
  ]
}
```

### 5-4. 足切りロジック

```python
SIMILARITY_THRESHOLD = 0.75

def search_with_fallback(tenant_id, query_vector, query_text):
    chunks = qdrant_search(tenant_id, query_vector, top_k=4)
    reliable_chunks = [c for c in chunks if c["score"] >= SIMILARITY_THRESHOLD]

    if not reliable_chunks:
        return {
            "answer": "お探しの情報が社内文書に見つかりませんでした。",
            "sources": [],
            "chunks_used": 0,
        }
    return run_rag(reliable_chunks, query_text)
```

> `SIMILARITY_THRESHOLD` は PoC で実測しながら調整する。製造業の専門用語は一般コーパスで学習した Embedding と相性が悪いことがあるため、0.70 から始めて上げていく。

---

## 6. チャンク設計

### 6-1. 基本設定（PoC 初期）

```python
chunk_size    = 512   # トークン数
chunk_overlap = 50    # 重複トークン数
```

### 6-2. 文書タイプ別の対応方針

| 文書タイプ | 特性 | 対応方針 |
|---|---|---|
| 仕様書・マニュアル | 文章中心 | デフォルト設定で概ね対応可 |
| 表・帳票 | セルをまたいだ文脈が必要 | ページ単位でチャンク化を検討 |
| スキャン PDF（画像） | テキストレイヤーなし | Gemini Vision で分析（→ 7 章） |
| 品番・型番を含む文書 | 記号列が分断されやすい | `overlap` を 100〜150 に増やす |
| 図面 | テキスト＋図が混在 | ページ単位 + Gemini Vision |

### 6-3. チャンク精度の改善フロー

1. デフォルト設定で動かす
2. 精度が悪い質問を記録する
3. 問題のある文書タイプを特定する
4. 該当タイプだけ設定を調整する

> チャンクは「検索のための単位」であり「表示のための単位」ではない。引用表示には LLM に該当箇所を 30 文字以内で抜き出させる方式を採用する。

---

## 7. 図面分析パイプライン（Gemini Vision）

### 7-1. PDF 種別の自動判定

PDF には大きく 2 種類あり、処理方法が異なる。Gemini Vision はテキストが抽出できない画像 PDF にのみ使用し、テキスト PDF には LlamaIndex の直接抽出を用いる。

| 種類 | 例 | テキスト抽出 | Vision 分析 |
|---|---|---|---|
| テキスト PDF | 仕様書・マニュアル・帳票 | ◎ LlamaIndex で直接抽出 | **不要** |
| 画像 PDF | スキャン図面・写真 | ✗ テキストなし | **必要** |
| 混在 PDF | 図面 + 説明文 | △ テキスト部分のみ | **補完として使用** |

**種別判定ロジック（PoC 実装）**

```python
def detect_pdf_type(page) -> str:
    """1 ページのテキスト密度から PDF 種別を判定"""
    word_count = len(page.get_text().strip().split())
    if word_count >= 50:
        return "text"    # テキストが十分ある → LlamaIndex で処理
    elif word_count >= 10:
        return "mixed"   # 少しテキストあり → Vision で補完
    else:
        return "image"   # ほぼ画像 → Vision 必須
```

> ⚠️ **改善の余地あり**
> - 閾値（50語・10語）は経験則。実際の登録文書で誤判定が出たら調整が必要。
> - 表・図が多い仕様書は word_count が少なくても "text" 扱いしたい場合がある。
> - 混在 PDF の処理はテキストと Vision 結果のマージ戦略が未確立。本番移行時に精度を実測して方針を決める。
> - 将来的にはページ種別（本文・図・表・図面）を自動分類する専用モデルの導入も検討余地あり。

**コストと速度の比較（100 ページ仕様書の場合）**

| 処理方法 | 時間 | コスト |
|---|---|---|
| 全ページ Vision（非推奨） | 200〜500 秒 | ~$0.03 |
| テキスト抽出のみ | 2〜5 秒 | ~$0.001 |
| **自動判定（推奨）** | **2〜5 秒** | **~$0.001** |

**Supabase files テーブルへの追加カラム**

```sql
ALTER TABLE files ADD COLUMN pdf_type text;
-- 値: "text" / "image" / "mixed" / "unknown"
```

---

### 7-2. 処理フロー（種別ごと）

**テキスト PDF（仕様書・マニュアル）**

```
仕様書.pdf
  → [STEP1] PyMuPDF でテキスト直接抽出
  → [STEP2] LlamaIndex でチャンク分割（chunk_size=512, overlap=50）
  → [STEP3] text-embedding-3-small でベクトル化
  → [STEP4] Qdrant に登録
  ※ Gemini Vision 呼び出しなし
```

**画像 PDF（スキャン図面）**

```
H01_Piston.pdf
  → [STEP1] PyMuPDF で画像化（200 dpi）
  → [STEP2] キャッシュ確認（vision_cache があればスキップ）
  → [STEP3] Gemini Vision で全体分析 → 構造化 JSON
        {
          figure_number, figure_name, parts,
          dimensions, surface_finish,
          geometric_tolerances, heat_treatment,
          confidence, unreadable_parts
        }
  → [STEP4] 検索用フラットテキストに変換
  → [STEP5] text-embedding-3-small でベクトル化
  → [STEP6] Qdrant に登録
        （tenant_id・materials・figure_number・confidence をペイロードに付与）
  → [STEP7] vision_cache に保存（次回以降はスキップ）
```

**統合パイプライン**

```python
async def process_pdf_page(page, page_num, file_meta):
    pdf_type = detect_pdf_type(page)

    if pdf_type == "text":
        text = page.get_text()
        return await chunk_and_embed(text, file_meta, page_num)

    elif pdf_type == "image":
        cached = get_vision_cache(file_meta["file_id"], page_num)
        if cached:
            return await embed_from_cache(cached, file_meta, page_num)
        analysis = await call_gemini_vision(page)
        save_vision_cache(file_meta["file_id"], page_num, analysis)
        return await embed_from_analysis(analysis, file_meta, page_num)

    elif pdf_type == "mixed":
        text     = page.get_text()
        analysis = await call_gemini_vision(page)   # 図・表の補完
        combined = merge_text_and_vision(text, analysis)
        return await chunk_and_embed(combined, file_meta, page_num)
```

---

### 7-3. 通常テキスト抽出との違い（画像 PDF の場合）

| 情報 | テキスト抽出のみ | Gemini Vision 分析 |
|---|---|---|
| 品名・材質 | ○ | ○ |
| 図番・図名 | △（レイアウト依存）| ○ |
| 寸法・数値 | ✗（画像内）| ○ |
| 公差 | ✗ | ○ |
| 表面粗さ記号 | ✗ | ○ |
| 幾何公差 | ✗ | ○ |
| 熱処理指示 | △ | ○ |
| 手書き注記 | ✗ | △（品質依存）|
| スキャン PDF 対応 | ✗ | ○ |

---

### 7-4. dpi の選び方

| 状況 | 推奨 dpi |
|---|---|
| テキスト PDF（ベクタ形式）| 150 |
| スキャン図面（鮮明）| 200（デフォルト）|
| スキャン図面（古い・薄い）| 300 |
| 手書き図面 | 300 以上 |

---

### 7-5. Vision キャッシュ戦略

```
初回登録時（画像 PDF のみ）:
  画像 → Gemini Vision 分析（トークン消費）→ Supabase vision_cache に保存

2 回目以降:
  vision_cache から取得（Gemini API 呼び出しなし・コストゼロ）
```

```python
CACHE_VERSION = "gemini-2.0-flash-v1"  # モデル変更時にここを変える

# キャッシュ無効化のケース
# ① 文書が更新 → file_id が変わるため自動的に再分析
# ② モデル変更 → CACHE_VERSION を更新して全キャッシュ無効化
# ③ 手動再分析 → force_reanalyze=True で強制実行
```

> Vision キャッシュにより、図面 100 枚登録後の再インデックスや検索時の再分析コストをゼロにできる。

---

### 7-6. 非同期登録フロー

Vision 分析は 1 画像あたり 2〜5 秒かかるため、ユーザーを待たせないようバックグラウンド処理にする。

```
ユーザーがアップロード
  → R2 への保存（即時）
  → Supabase に status="indexing" で登録
  → レスポンスを即座に返す（ユーザーは待たされない）
  → バックグラウンドで PDF 種別判定 → テキスト抽出 or Vision 分析 → Qdrant 登録
  → 完了したら status="ready" に更新
  → フロントがポーリングして「インデックス中 → 登録済」に表示更新
```

```python
@app.post("/api/files/upload")
async def upload(file: UploadFile, background_tasks: BackgroundTasks,
                 tenant_id: str = Depends(get_tenant_from_jwt)):

    file_id = str(uuid.uuid4())
    r2_key  = await save_to_r2(file, file_id, tenant_id)

    # status="indexing" で即登録・即レスポンス
    await supabase.table("files").insert({
        "id": file_id, "tenant_id": tenant_id,
        "filename": file.filename, "r2_key": r2_key,
        "status": "indexing", "pdf_type": "unknown",
    }).execute()

    # 分析・インデックス登録はバックグラウンドで実行
    background_tasks.add_task(
        analyze_and_index,     # 種別判定 → 抽出/Vision → Qdrant → status="ready"
        file_id, r2_key, file.filename, tenant_id
    )

    return {"file_id": file_id, "status": "indexing"}
```

> ⚠️ **改善の余地あり**
> - PoC では FastAPI の `BackgroundTasks` で十分だが、本番移行後はタスクキュー（Celery・Cloud Tasks）への移行を検討する。サーバー再起動時に処理中のタスクが消失するリスクがある。
> - 大量ファイルの一括アップロード時の同時実行数制御（セマフォ）が未実装。Vision API のレート制限に引っかかる可能性がある。

---

## 8. セキュリティ設計

### 8-1. テナント分離

- **R2**: `{tenant_id}/` プレフィックスで物理的に分離
- **Qdrant**: 全クエリに `tenant_id` フィルタを FastAPI 側で強制付与（フロントからの値をそのまま使わない）
- **Supabase**: Row Level Security（RLS）で全テーブルに `tenant_id` 制約

### 8-2. ファイルアクセス（Presigned URL）

R2 のファイルは直接公開しない。サーバーが認可した場合のみ期限付き一時 URL を発行する。

```python
def get_presigned_url(r2_key: str, page: int = None, expires_sec: int = 3600) -> str:
    base_url = r2_client.generate_presigned_url(
        "get_object",
        Params={"Bucket": BUCKET, "Key": r2_key},
        ExpiresIn=expires_sec,
    )
    # #page=N を付けるとブラウザが該当ページを開く
    return f"{base_url}#page={page}" if page else base_url
```

**Presigned URL のメリット**
- 有効期限付き（1 時間）なので URL が漏洩しても被害が限定的
- PDF データはサーバーを経由しない（FastAPI の帯域・メモリを節約）
- フロントが直接 R2 から取得するため表示が速い

### 8-3. 認証

- Supabase Auth（メール/パスワード）
- JWT から `tenant_id` を取得し、全 API で検証
- ロール: `admin`（文書管理・フィードバック処理）/ `member`（チャットのみ）

---

## 9. 引用表示・PDF ビューア設計

### 9-1. 表示モード

| モード | 動作 | 実装 |
|---|---|---|
| インライン表示 | 引用カードクリック → チャット下部に PDF ビューア表示 | `<iframe src="{presigned_url}#page={page}">` |
| URL で開く | 引用カードの「開く」ボタン → 新しいタブで該当ページ | `window.open(presigned_url, '_blank')` |

モード設定は `localStorage` に保存し、次回起動時に引き継ぐ。

### 9-2. 引用カードの構造

```json
{
  "filename":      "仕様書_A1234.pdf",
  "page":          12,
  "quote":         "材質: SUS304  板厚: 2.0mm",
  "presigned_url": "https://...#page=12",
  "score":         0.94
}
```

- `quote` は LLM が回答に使った箇所を 30 文字以内で抜き出す（チャンク全文は返さない）
- `score`（類似度）は開発・デバッグ用。本番 UI では非表示が望ましい

### 9-3. 実装段階

**PoC 段階**: `<iframe>` + `#page=N`（Chrome 限定・ハイライトなし）で十分  
**本番移行後**: Safari 対応・ハイライト要望が出たら `react-pdf` へ移行

```tsx
// PoC 段階の実装（数行で完成）
function PdfViewer({ presignedUrl, page }: { presignedUrl: string; page: number }) {
  return (
    <iframe
      src={`${presignedUrl}#page=${page}`}
      width="100%"
      height="420px"
      style={{ border: "none", borderRadius: 8 }}
    />
  );
}
```

### 9-4. 注意点

- チャンク全文をフロントに返すと単価・仕入れ先など意図しない情報が混入する可能性がある → LLM に該当箇所だけを抜き出させる方式を採用
- PDF ページ番号は LlamaIndex の抽出結果と実際の PDF のページがズレることがある → PoC で実測して確認する
- 回答と引用が矛盾する場合がある（LLM のハルシネーション）→ UI で「必ず引用元を確認してください」という注意書きを入れる

---

## 10. 画面構成

### 10-1. 画面一覧

| 画面 | 主な機能 | 対象ユーザー |
|---|---|---|
| チャット | 質問入力・回答表示・引用カード・PDF 表示モード切替・フィードバックボタン | 全員 |
| ファイル管理 | ドラッグ&ドロップアップロード・登録ファイル一覧・ステータス表示・削除 | 全員 |
| ナレッジカード | カード一覧（カテゴリ・信頼度）・手動登録フォーム・不具合報告フォーム | 全員 |
| フィードバック管理 | 未対応報告一覧・件数サマリー・ステータス更新（解決/却下） | 管理者のみ |

### 10-2. チャット画面の詳細

```
┌─────────────────────────────────────────────┐
│ 社内文書を検索する          [インライン|URL] │  ← モード切替トグル
├─────────────────────────────────────────────┤
│                                             │
│  [ユーザー] 品番 A-1234-B の材質と板厚は？  │
│                                             │
│  [AI] 材質は SUS304、板厚は 2.0mm です。    │
│       表面処理: #400研磨                    │
│                                             │
│  出典 2件                                   │
│  ┌──────────────────────────────────────┐  │
│  │ 1  仕様書_A1234.pdf  p.12  94%       │  │  ← クリックでPDF表示
│  │    「材質: SUS304  板厚: 2.0mm」      │  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │ 2  図面管理台帳.xlsx  行47  81%      │  │
│  │    「A-1234-B  関連図面: DWG-5678」   │  │
│  └──────────────────────────────────────┘  │
│                                             │
│  👍 役に立った  ⚑ 問題を報告               │
│                                             │
│  ┌── PDF ビューア（インラインモード時）──┐  │
│  │  仕様書_A1234.pdf  p.12            × │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │  ...                          │  │  │
│  │  │  [引用箇所ハイライト]          │  │  │
│  │  │  材質: SUS304  板厚: 2.0mm    │  │  │
│  │  │  ...                          │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│  [質問を入力...                    ] [送信] │
└─────────────────────────────────────────────┘
```

### 10-3. 問題報告モーダル

- 問題の種類（3 択）: 回答が間違っている / 情報が古い / 知りたいことと違う
- 詳細コメント（任意）
- 送信 → Slack `#rag-feedback` に自動通知

---

### 10-4. 印刷機能

> ⚠️ **議論・調査の余地あり**  
> 以下は PoC デモとして実装した初期仕様。実際の現場ニーズをパイロット企業にヒアリングしたうえで仕様を確定する。

#### 概要

各画面のトップバーに「印刷」ボタンを設置。クリックするとオプション選択モーダルが開き、印刷範囲・内容を選んでからブラウザの印刷ダイアログへ渡す。

#### 印刷オプション一覧（現行実装）

| オプション | 対象画面 | 内容 |
|---|---|---|
| このページ全体 | 全画面 | 現在表示中の画面をそのまま印刷 |
| 回答のみ | チャットのみ | AI の回答テキストと出典情報のみ。図面・ツールバー・入力欄は除外 |
| 図面付き回答 | チャットのみ | 引用箇所マーキング付きの図面画像を含めて印刷 |
| カスタム | 全画面 | 質問・回答・出典・図面・ナレッジカード・メタ情報を個別チェックで選択 |

#### 印刷時の自動処理

- サイドバー・入力欄・ツールボタン・フィードバックボタンは `@media print` で自動非表示
- 印刷ヘッダー（テナント名・印刷日時）を自動挿入
- 印刷完了後に一時非表示にした要素を自動復元

#### 議論・調査が必要な項目

**① 出力形式**

現状はブラウザの印刷ダイアログ（PDF 保存も含む）を利用しているが、要件によっては専用の PDF 生成が必要になる可能性がある。

| 方式 | メリット | デメリット | 検討優先度 |
|---|---|---|---|
| ブラウザ印刷（現行）| 追加実装なし・OS の印刷設定をそのまま使える | レイアウトがブラウザ依存・ヘッダー/フッターがブラウザ任意 | PoC はこれで十分 |
| サーバーサイド PDF 生成（puppeteer/weasyprint）| レイアウト完全制御・テンプレート化できる | サーバー負荷・実装コスト大 | 本番移行後に検討 |
| react-pdf による PDF 構築 | フロントのみで完結・細かいレイアウト制御 | 学習コスト・図面画像の埋め込みが複雑 | 本番移行後に検討 |

**② 印刷対象の粒度**

- 現状は「画面単位」または「回答単位」だが、**特定の回答だけを選んで印刷**したいユースケースがあるか要確認。
- 複数回の質問をまとめて1枚のレポートにする「セッションまとめ印刷」のニーズがあるか要確認。

**③ 図面の印刷品質**

- 現状は `<img>` タグのスクリーン解像度をそのまま印刷している（72〜96dpi 相当）。
- 業務使用（図面の照合・承認・回覧）に耐えるには 150〜300dpi の画像が必要な可能性がある。
- R2 に保存している原本 PDF を高解像度で取得して印刷に使うフローが別途必要になるか要検討。

**④ 機密情報の扱い**

- 印刷物には社内の設計情報・材質・寸法が含まれる。印刷ログを残すか、印刷権限（管理者のみ可）を設けるかの運用ルールが必要。
- 特定の文書に「印刷禁止」フラグを付ける仕組みが必要になる可能性がある。

**⑤ 現場での実際の用途**

以下についてパイロット企業にヒアリングして仕様を確定する。

- 印刷した紙を何に使うか（作業指示書への添付？承認回覧？外注先への送付？）
- 電子データ（PDF メール添付）で十分か、紙への印刷が必須か
- 印刷頻度はどの程度か（毎日？週次？）
- A4 横・縦どちらが使いやすいか（図面は横が多い）

---

## 11. 暗黙知のデータ化

### 11-1. 暗黙知の種類と難易度

| 種類 | 例 | 難易度 | PoC で対応 |
|---|---|---|---|
| 手順の暗黙知 | 「このネジは斜めから入れると舐めない」 | 低 | ○ |
| 判断の暗黙知 | 「この音がしたら刃を替えるタイミング」 | 中 | ○ |
| 関係の暗黙知 | 「A 社担当者はこういう言い方をする」 | 高 | 後回し |
| 例外の暗黙知 | 「通常はこうだが○○のときだけ違う」 | 高 | 後回し |

### 11-2. データ収集の 3 ルート

#### ルート 1: インタビュー → AI 構造化

ベテランの発言をその場で Gemini に投げてナレッジカード化する。

```
ベテラン:「SUS304 の溶接は電流 80A 以下にしてる。
          母材が濡れてると絶対ダメ。」
         ↓
Gemini プロンプト:「以下をナレッジカード形式に整理してください。
                   形式: 条件 / 推奨アクション / 理由 / よくあるミス」
         ↓
担当者が確認 → 登録（unverified → high へのステータス遷移）
```

#### ルート 2: 日常業務への仕掛け

不具合報告フォーム（3 項目のみ）:
1. 何が起きましたか？
2. どうやって直りましたか？
3. 次回の注意点は？

週 1 回、溜まった報告を Gemini で構造化 → 担当者確認 → ナレッジカードとして登録。

#### ルート 3: 既存文書のマイニング

すでに社内にある以下の文書を RAG に投げることでナレッジを引き出す。

- 不良品報告書（8D 報告書）
- 品質会議の議事録
- 顧客クレーム対応記録
- ベテランが個人で作った Excel メモ

### 11-3. ナレッジカードのデータ構造

```python
{
  "category":    "溶接",
  "condition":   "SUS304 溶接時",
  "action":      "電流は 80A 以下に設定",
  "reason":      "入熱過多による鋭敏化防止",
  "caution":     "母材の水分・油分を必ず除去（ブローホール防止）",
  "exceptions":  "板厚 3mm 以上は 70A 以下",
  "source_type": "interview",
  "contributor": "田中さん（溶接歴 20 年）",
  "verified_by": "品質部 鈴木",
  "confidence":  "high",          # unverified → medium → high
}
```

### 11-4. PoC の最小構成

1. 不具合報告フォームを作る（3 項目のみ）
2. 1 週間分が溜まったら RAG に投げる
3. 「過去に同じ問題があったか？」で検索できるか確認
4. ベテランに「自分の知識と合っているか」確認してもらう

---

## 12. フィードバック・間違い報告設計

### 12-1. 間違いの種類と対処

| 種類 | 原因 | 対処 |
|---|---|---|
| `wrong_answer`（回答が間違い）| LLM の誤読・プロンプト問題 | プロンプト修正 / 正解 Q&A 登録 |
| `outdated`（情報が古い）| 仕様変更後に文書未更新 | 文書差し替え → 再インデックス |
| `missing_info`（情報がない）| 文書未登録 | 対象文書を追加登録 |

### 12-2. 運用フロー

```
ユーザーが「問題を報告」をクリック
  → 種類選択（3 択）+ コメント入力（任意）
  → Supabase feedbacks テーブルに記録
  → Slack #rag-feedback に自動通知

管理者がダッシュボードで確認（48 時間以内）
  → 「文書を更新して解決」→ 文書差し替え → 手動再インデックス
  → 「却下」→ 却下コメントを記載
```

### 12-3. 運用ルール

| 項目 | 内容 |
|---|---|
| 報告の受信 | Slack の `#rag-feedback` チャンネルに自動通知 |
| 確認期限 | 報告から 48 時間以内に担当者が確認 |
| 修正対応 | 文書差し替え → 手動で再インデックス（PoC 段階） |
| 却下基準 | ユーザーの解釈違いと判断した場合（却下コメントを記載） |

### 12-4. 後から入れるべき機能（PoC 後）

- 報告が 10 件以上溜まったら正解 Q&A を明示的に登録する仕組みを追加
- フィードバックによる自動再インデックス（本番移行後・データ量が必要）

---

## 13. 登録サービス一覧

PoC 開始前に以下 6 サービスのアカウントを作成する。

| サービス | 用途 | 登録方法 | 無料枠 |
|---|---|---|---|
| Google AI Studio | Gemini API キー発行 | `aistudio.google.com` → Google アカウントでログイン → 「Get API key」 | 1 分 15 req・1 日 1,500 req |
| OpenAI | Embedding（text-embedding-3-small） | `platform.openai.com` → アカウント作成 → API keys → クレジット $5 チャージ | なし（従量課金） |
| Supabase | DB・認証・テナント管理 | `supabase.com` → New project → DB パスワード設定 → Settings → API でキー確認 | 2 プロジェクト・500MB |
| Cloudflare | R2 ファイルストレージ | `cloudflare.com` → R2 → バケット作成 → API トークン発行 | 10GB/月・請求なし（カード登録は必要）|
| Render | FastAPI サーバー | `render.com` → GitHub 連携 → New Web Service | あり（15 分スリープあり）|
| Vercel | Next.js フロントエンド | `vercel.com` → GitHub 連携 → Import Project | 個人・小チーム無制限 |

> Qdrant はローカル開発では `docker run -d -p 6333:6333 qdrant/qdrant` のみで起動可。アカウント不要。

### 環境変数まとめ

```bash
# Gemini
GOOGLE_API_KEY=...

# OpenAI（Embedding）
OPENAI_API_KEY=...

# Supabase
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...    # サーバー側のみ使用

# Cloudflare R2
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_NAME=mfg-rag-prod

# Qdrant（本番移行後）
QDRANT_URL=https://xxxx.qdrant.io
QDRANT_API_KEY=...
```

> `.env` ファイルは必ず `.gitignore` に追加すること。Render・Vercel にはダッシュボードから環境変数を直接入力する。

---

## 14. PoC サンプルデータ

製造業の実務図面はほぼ公開されていない。以下の組み合わせで検証データを揃える。

| 目的 | 文書 | 入手先 | 登録要否 |
|---|---|---|---|
| 仕様書（表・数値）| TDK コンデンサ納入仕様書 | product.tdk.com | 不要 |
| 仕様書（品番・RoHS）| 興和化成 配線ダクト仕様書 | kowa-kasei.co.jp/download | 不要 |
| 技術資料（材料特性表）| 特殊金属エクセル テクニカルガイド（全 60P）| tokkin.co.jp/downloads | メール登録 |
| 加工ガイド（工程・不良事例）| 高橋金属 せん断加工ガイドブック | takahasi-k.jp/download | 不要 |
| 機械部品外形図 | MISUMI FA 用メカニカル標準部品カタログ PDF | jp.misumi-ec.com/maker/misumi/catalog/mech/dl/ | 不要 |
| 実機械図面（寸法・公差）| LLM で合成生成 | Gemini で生成 | — |

### 合成データ生成プロンプト例

```
以下の形式で製造業の仕様書サンプルを生成してください。

【形式】
- 品番・材質・寸法・表面処理の表を含む
- JIS 規格への参照を含む
- 関連図面番号への参照を含む
- 注記・禁止事項を含む

【内容】
ステンレス切削部品（SUS304）の納入仕様書。
品番は A-1234 シリーズで 3 種類。
板厚・穴径・公差が各品番で異なる。
```

---

## 15. 開発ロードマップ

| 週 | 内容 | 完了条件 |
|---|---|---|
| Week 1〜2 | 環境構築: FastAPI + LlamaIndex + Qdrant をローカルで動作確認 | PDF 投入 → Qdrant 登録が動く |
| Week 3〜4 | コア機能: Gemini Vision 図面分析・チャンク化・RAG 回答の一本道を完成 | 「品番の材質は？」が答えられる |
| Week 5〜6 | UI: Next.js チャット画面 + Supabase 認証 + 引用表示・モード切替 | ブラウザで使える状態 |
| Week 7〜8 | パイロット: 1 社目に渡して精度・UX を検証 / フィードバック収集 | 実ユーザーから報告が届く |

---

## 16. PoC 後に判断する事項

### 16-1. インフラ・スケール

| 項目 | 条件 | 対応 |
|---|---|---|
| ローカル LLM 対応 | 防衛・航空宇宙など機密性の高い顧客が現れたとき | Ollama + ローカルモデル構成を検討 |
| react-pdf ハイライト実装 | Safari 非対応の苦情 or ハイライト要望が出たとき | iframe から react-pdf へ移行 |
| Embedding モデル変更 | 専門用語の検索精度問題が顕在化したとき | 多言語特化モデルへ変更（再インデックス必要）|
| 自動再インデックス | 本番移行後（データ量が必要）| フィードバック → 自動更新パイプライン |
| Qdrant Cloud 移行 | テナント数が 10 社を超えたとき | ローカル Docker から Qdrant Cloud へ |
| 料金設計確定 | パイロット結果で価値を確認してから | 容量課金ベースで月 3〜10 万円を想定 |
| フィードバック自動処理 | 報告が月 30 件以上になったとき | 正解 Q&A の自動登録・優先度スコアリング |

### 16-2. LLM プロンプト最適化

#### 背景と問題意識

業界固有の知識（用語定義・判断基準・製造ルール）を LLM に持たせたい場合、API で利用するモデルへのファインチューニング（追加学習）は基本的に不可能。  
代替手段としてシステムプロンプトへの埋め込みが考えられるが、以下 2 つの問題がある。

- **トークンコストの増加**: システムプロンプトはすべてのクエリで毎回消費される。業界辞書を静的に書くと数千トークンが固定費となり、テナント数・クエリ数が増えるにつれコストが膨らむ。
- **精度の低下（Lost in the Middle）**: プロンプトが長くなるほど LLM は中間部分の情報を見落としやすくなる。無関係な用語定義が混入すると回答精度が下がる。

#### 設計方針：「静的に持たせる」ではなく「動的に取ってくる」

RAG の考え方をチャンク検索だけでなく、**用語定義・ナレッジカード・判断基準にも適用する**。クエリに関連する情報だけをその都度取得してプロンプトに注入することで、トークン量を最小化しつつ精度を維持する。

```
PoC 段階（静的）:
  システムプロンプト（固定 500 トークン）
  + 全用語辞書（固定 2,000〜5,000 トークン）  ← 毎回全量消費
  = 1クエリあたり ~5,500 トークン

本番移行後（動的）:
  システムプロンプト（固定 150 トークン）
  + 関連用語 2〜3件（動的 ~200 トークン）
  + 関連ナレッジカード 3件（動的 ~300 トークン）
  + RAGチャンク（動的 ~800 トークン）
  = 1クエリあたり ~1,450 トークン（約 74% 削減）
```

#### 実装アプローチ

**① Qdrant の `doc_type` を拡張して用語定義を管理する**

```python
# doc_type の種類（PoC では drawing/spec/manual/knowledge/other）
# 本番移行後に追加する種別:
DOC_TYPES = {
    "drawing":    "図面",
    "spec":       "仕様書",
    "manual":     "作業標準・マニュアル",
    "knowledge":  "ナレッジカード（暗黙知）",
    "term":       "用語定義（業界辞書）",   # ← 追加
    "rule":       "判断基準・社内ルール",    # ← 追加
    "other":      "その他",
}
```

**② 動的プロンプト組み立てパイプライン**

```python
async def build_prompt(tenant_id: str, query: str, query_vector: list) -> str:

    # 固定: 最小限のシステム指示（常に短く保つ）
    system = """製造業社内検索アシスタント。
社内文書のみを根拠に答える。根拠がなければ「記載なし」と答える。
出典（ファイル名・ページ）を必ず記載する。"""

    # 動的①: 関連する用語定義（上位 2〜3 件のみ）
    terms = qdrant_search(tenant_id, query_vector, doc_type="term", top_k=3)

    # 動的②: 関連するナレッジカード（上位 3 件のみ）
    cards = qdrant_search(tenant_id, query_vector, doc_type="knowledge", top_k=3)

    # 動的③: 関連する判断基準（上位 2 件のみ）
    rules = qdrant_search(tenant_id, query_vector, doc_type="rule", top_k=2)

    # 動的④: RAG チャンク（既存の実装）
    chunks = qdrant_search(tenant_id, query_vector, doc_type=None, top_k=4)

    # 関連するものだけをプロンプトに含める
    prompt = system
    if terms:  prompt += f"\n\n## 用語定義\n{format_terms(terms)}"
    if rules:  prompt += f"\n\n## 判断基準\n{format_rules(rules)}"
    if cards:  prompt += f"\n\n## 社内ナレッジ\n{format_cards(cards)}"
    prompt    += f"\n\n## 参照文書\n{format_chunks(chunks)}"
    prompt    += f"\n\n## 質問\n{query}"
    return prompt
```

**③ 用語定義の登録例**

```python
TERMS = [
    {
        "text":       "AC8A-T5: アルミニウム合金鋳物。T5は人工時効処理。耐熱・耐摩耗性に優れエンジンピストンに多用。",
        "doc_type":   "term",
        "category":   "材質",
        "keywords":   ["AC8A", "T5", "アルミ", "鋳物", "ピストン"],
    },
    {
        "text":       "鋭敏化: ステンレス鋼を特定温度域に加熱した際にクロム炭化物が析出し耐食性が低下する現象。SUS304 溶接時に注意。",
        "doc_type":   "term",
        "category":   "溶接欠陥",
        "keywords":   ["SUS304", "溶接", "ステンレス", "耐食性"],
    },
    {
        "text":       "引け巣: 鋳造時の凝固収縮により内部に生じる空洞欠陥。溶湯温度・湯口設計・押し湯で対策する。",
        "doc_type":   "term",
        "category":   "鋳造欠陥",
        "keywords":   ["鋳造", "欠陥", "空洞", "溶湯"],
    },
]
```

#### PoC での暫定対応

本番移行前の PoC 段階では、以下の最小構成で進める。

- システムプロンプトには役割・ルールのみ記載（300 トークン以内を目安）
- 業界辞書は `doc_type="term"` として Qdrant に登録しておく（検索対象に含める）
- 用語定義の動的フィルタリングは本番移行時に実装

PoC の目的は「動的取得が本当に必要なレベルの問題が発生するか」を確認することであり、問題が顕在化してから実装するのが工数的に合理的。

---

---

## 17. 実装状況レポート（2026-05-26）

### 17-1. ディレクトリ構成（実際）

| パス | 状態 | 備考 |
|---|---|---|
| backend/main.py | ✅ | |
| backend/routers/files.py | ✅ | |
| backend/routers/chat.py | ✅ | |
| backend/routers/knowledge.py | ✅ | |
| backend/routers/feedback.py | ✅ | |
| backend/services/r2.py | ✅ | SSLパッチ適用済み |
| backend/services/qdrant.py | ✅ | |
| backend/services/supabase_client.py | ✅ | supabase.py も別途あり |
| backend/services/indexer.py | ✅ | ⚠️ SSL パッチ未適用（後述）|
| backend/services/vision.py | ✅ | |
| backend/services/embedder.py | ✅ | |
| backend/services/rag.py | ✅ | |
| backend/models/schemas.py | ✅ | |
| backend/dependencies.py | ✅ | |
| backend/config.py | ✅ | |
| backend/scripts/check_connections.py | ✅ | デバッグ用 |
| frontend/app/page.tsx | ✅ | チャット画面 |
| frontend/app/files/page.tsx | ✅ | |
| frontend/app/knowledge/page.tsx | ✅ | |
| frontend/app/feedback/page.tsx | ✅ | |
| frontend/components/PdfViewer.tsx | ✅ | |
| frontend/components/AppShell.tsx | ⚠️ | 存在するが未接続 |
| frontend/components/Sidebar.tsx | ⚠️ | 存在するが未接続 |
| frontend/lib/api.ts | ✅ | |
| frontend/lib/supabase.ts | ✅ | |
| supabase/migrations/001_initial.sql | ✅ | |
| supabase/migrations/002_auth_hook.sql | ✅ | |
| upload_direct.py | ⚠️ | ルートに存在（SSL回避用スクリプト） |

---

### 17-2. 主要定数・設定値（現行実装）

| ファイル | 定数・設定 | 現在の値 | 備考 |
|---|---|---|---|
| rag.py | SIMILARITY_THRESHOLD | 0.45 | 当初 0.75 → 0.50 → 0.45 に引き下げ |
| rag.py | LLM モデル | gemini-2.5-flash | gemini-2.0-flash から変更 |
| rag.py | チャンク上限 | 4件 | |
| rag.py | ナレッジ上限 | 3件 | |
| vision.py | _CACHE_VERSION | "gemini-2.0-flash-v1" | ⚠️ モデルは 2.5-flash だが文字列は旧のまま |
| vision.py | モデル | gemini-2.5-flash | |
| indexer.py | chunk_size | 512 | |
| indexer.py | chunk_overlap | 50 | |
| indexer.py | detect_pdf_type 閾値 | word >= 50 → text / >= 10 → mixed | |
| r2.py | SSL設定 | verify=certifi.where() + OP_IGNORE_UNEXPECTED_EOF パッチ | |
| dependencies.py | JWT検証 | PyJWKClient + JWKS（ES256/HS256両対応）| |

---

### 17-3. requirements.txt（現行）

```
fastapi==0.111.0
uvicorn[standard]==0.30.1
python-multipart==0.0.9
pydantic==2.7.1
pydantic-settings==2.2.1
python-dotenv==1.0.1
google-generativeai        # バージョン未固定
llama-index==0.10.43
llama-index-llms-gemini
llama-index-embeddings-openai==0.1.11
certifi
boto3==1.34.114
qdrant-client==1.9.1
supabase==2.5.0
PyJWT==2.8.0
cryptography==43.0.0
pymupdf==1.24.3
Pillow==10.3.0
# ⚠️ openai パッケージが未記載（embedder.py が使用）
```

---

### 17-4. frontend 依存関係（現行）

| パッケージ | バージョン |
|---|---|
| next | 14.2.35 |
| react / react-dom | ^18 |
| @supabase/ssr | ^0.10.3 |
| @supabase/supabase-js | ^2.106.1 |
| react-dropzone | ^15.0.0 |
| tailwindcss | ^3.4.1 |
| typescript | ^5 |

---

### 17-5. 未解決・要対応の問題

| 項目 | 優先度 | 詳細 | 対処方針 |
|---|---|---|---|
| フロント経由アップロードの SSL エラー | 🔴 高 | Docker→R2 の SSL エラー（SSLZeroReturnError）が未解決。upload_direct.py で回避中 | indexer.py の analyze_and_index に r2.py の SSL パッチを適用する |
| indexer.py の SSL パッチ未適用 | 🔴 高 | analyze_and_index が独自に boto3.client を生成しているため R2 からの PDF 取得が失敗 | r2.py の _get_client() を使うよう修正 |
| _CACHE_VERSION の不一致 | 🟡 中 | vision.py の _CACHE_VERSION = "gemini-2.0-flash-v1" だがモデルは gemini-2.5-flash | "gemini-2.5-flash-v1" に更新 |
| openai パッケージ未記載 | 🟡 中 | embedder.py が openai を使うが requirements.txt に未記載 | openai を requirements.txt に追加 |
| AppShell.tsx / Sidebar.tsx 未接続 | 🟢 低 | ファイルは存在するが各 page.tsx から使われていない | 各 page.tsx に組み込むか削除 |
| Slack 通知未実装 | 🟢 低 | feedback.py の _notify_slack は pass のまま | 本番移行時に実装 |
| RAG 検索が常に「記載なし」を返す | 🔴 高 | SIMILARITY_THRESHOLD を 0.45 に下げても改善しない場合は Qdrant のデータ確認が必要 | Qdrant のポイント数・テナントIDフィルタを確認 |

---

### 17-6. docker-compose.yml（現行）

```yaml
services:
  qdrant:
    image: qdrant/qdrant:v1.9.2
    ports: ["6333:6333"]
    volumes: [qdrant_data]

  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      QDRANT_URL: http://qdrant:6333   # docker内部ではqdrantホスト名を使用
    # --reload フラグなし
```

---

*このドキュメントはチーム開発の起点として使用してください。*  
*仕様変更は PR 時に本ドキュメントも更新すること。*
