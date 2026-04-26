# Day 2 受講者ガイド：RAG 実装と住民窓口チャットボット構築

> 本ガイドは `day2_rag_chatbot.ipynb` を使って演習を進めるための手引きです。  
> Google Colab の基本操作は `day1_guide.md` を参照してください。

---

## 目次

1. [Day 2 の全体像](#1-day-2-の全体像)
2. [ノートブック（.ipynb）の進め方](#2-ノートブックipynbの進め方)
3. [RAG の仕組みをイメージで理解する](#3-rag-の仕組みをイメージで理解する)
4. [演習の流れ](#4-演習の流れ)
5. [つまずきやすいポイントと対処法](#5-つまずきやすいポイントと対処法)
6. [学習の手助けになるサイト](#6-学習の手助けになるサイト)

---

## 1. Day 2 の全体像

### 今日作るもの

**住民FAQ RAGチャットボット**  
自治体のFAQ文書を事前に読み込ませ、その内容に基づいて住民の質問に正確に答えるAIシステムです。

```
Day 1 で作ったもの        Day 2 で追加するもの
─────────────────────    ──────────────────────────────────────
LLM に直接質問する         FAQ 文書を読ませて、文書に基づいて答える
（知識はモデル任せ）  →    （RAG = 検索 + LLM）
                          + Gradio で Web UI を構築
```

### 本日の技術スタック

| 役割 | 使用技術 | 源内 OSS での対応 |
|------|---------|-----------------|
| LLM | Gemini API | Amazon Bedrock |
| Embedding | Gemini text-embedding-004 | Amazon Bedrock Embeddings |
| ベクトル DB | ChromaDB（インメモリ） | Amazon OpenSearch |
| PDF 読み取り | PyMuPDF（fitz） | 同等ライブラリ |
| チャット UI | Gradio | React + TypeScript |

---

## 2. ノートブック（.ipynb）の進め方

### ノートブックを開く前の準備

1. Day 1 で Gemini API キーを Colab シークレットに登録済みであることを確認
2. 下記バッジまたは GitHub から Day 2 ノートブックを開く

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/TrSaleMane/municipality-ai-course/blob/main/docs/ipynb/day2_rag_chatbot.ipynb)

3. 「ファイル」→「ドライブにコピーを保存」で自分のドライブにコピー

### セルの進め方の注意点

Day 2 は Day 1 より処理の依存関係が強いため、**順番を飛ばすと必ずエラーになります**。

```
Section 1（セットアップ）
    ↓ ← 必ず実行してからでないと次に進めない
Section 2（FAQ データ準備）
    ↓
Section 3（チャンク分割）
    ↓ ← vector_store 変数を Section 4 以降が使う
Section 4（ベクトル DB 登録）
    ↓
Section 5（RAG チェーン実装）
    ↓
Section 6（Gradio UI 構築）
    ↓
Section 7（PDF 読み込み・発展）
```

### ノートブックの構成（Day 2）

| セクション | 内容 | 実行時間の目安 |
|-----------|------|--------------|
| Section 1 | 環境セットアップ | 1〜3分（初回） |
| Section 2 | 演習用 FAQ データの準備 | 即時 |
| Section 3 | テキストのチャンク分割 | 即時 |
| Section 4 | ChromaDB へのベクトル登録 | 1〜2分 |
| Section 5 | RAG チェーン実装・テスト | 30秒〜1分 |
| Section 6 | Gradio UI 起動 | 30秒〜1分 |
| Section 7 | PDF 読み込み（発展） | 任意 |
| チャレンジ | 精度評価・FAQ 追加・ソース表示 | 任意 |

---

## 3. RAG の仕組みをイメージで理解する

### なぜ RAG が必要か

LLM（大規模言語モデル）は学習データに含まれていない情報は知りません。  
また、古い情報や間違った情報を自信満々に答えてしまう「ハルシネーション」が起きます。

```
❌ 普通の LLM の問題点

  質問：「本市の住民票の手数料はいくらですか？」
    ↓
  LLM：「300円です」（実際は200円かもしれない）
  ※ 学習データ時点の情報 or 推測で答えてしまう
```

```
✅ RAG で解決

  質問：「本市の住民票の手数料はいくらですか？」
    ↓
  FAQ 文書を検索して「コンビニは200円」の記述を発見
    ↓
  LLM：「FAQ によると、コンビニでは200円です」
  ※ 文書に根拠を持った回答ができる
```

### RAG の 2 つのフェーズ

#### フェーズ 1：インデックス構築（事前準備）

```
FAQ 文書（テキスト）
        ↓
   チャンク分割
   （300文字ずつ等の意味のある単位に切り出す）
        ↓
  Embedding モデル
   （テキスト → 数値ベクトルに変換）
        ↓
   ChromaDB に保存
   （ベクトルとテキストをセットで格納）
```

#### フェーズ 2：質問応答（ユーザー利用時）

```
住民の質問
        ↓
  質問もベクトル化
        ↓
  ChromaDB で類似検索
  （質問ベクトルに近いチャンクを取得）
        ↓
  プロンプトに組み込む
  「次の参考情報をもとに答えてください：
   [チャンク1]
   [チャンク2]
   質問：〇〇」
        ↓
  Gemini API が回答生成
        ↓
  住民に回答
```

### Embedding（埋め込み）とは

テキストを数値のリスト（ベクトル）に変換することで、  
**意味的に近いテキスト同士をコンピュータが比較できるようにする**技術です。

```
「住民票の取り方」  →  [0.12, -0.45, 0.89, ...]（768次元のベクトル）
「住民票の手続き」  →  [0.13, -0.44, 0.87, ...]（よく似たベクトル）
「道路の修繕申請」  →  [0.67,  0.23, -0.12, ...]（全然違うベクトル）
```

---

## 4. 演習の流れ

### Section 1 〜 2：準備

```
Section 1 を実行 → 「✅ ライブラリ読み込み完了」を確認
Section 2 を実行 → 「✅ FAQデータ準備完了」を確認
```

### Section 3：チャンク分割を体験

`split_into_chunks()` 関数を実行して、FAQ テキストが何個のチャンクに分割されたか確認します。  
`chunk_size` パラメータを変えて、チャンクの数や内容の変化を観察してみましょう。

```python
# 試してみよう：chunk_size を変えると何が変わる？
chunks_small = split_into_chunks(SAMPLE_FAQ, chunk_size=100)  # 細かく分割
chunks_large = split_into_chunks(SAMPLE_FAQ, chunk_size=600)  # 大きく分割
print(f'小さいチャンク: {len(chunks_small)}個')
print(f'大きいチャンク: {len(chunks_large)}個')
```

### Section 4：ベクトル DB への登録

Embedding モデルが各チャンクをベクトル化して ChromaDB に保存します。  
チャンク数によっては **1〜2分かかります**。待ちましょう。

実行後、**検索テスト**で類似するチャンクが取得できているかを確認します。

### Section 5：RAG チェーンの実装

`MunicipalityRAGBot` クラスの `answer()` メソッドを読んで、  
RAG の 3 ステップ（検索 → プロンプト構築 → LLM 回答）を理解しましょう。

```python
# プロンプトの中身を確認してみよう
question = '住民票をコンビニで取りたいです'
relevant_chunks = vector_store.search(question, n_results=3)
context = '\n\n'.join(relevant_chunks)
print('=== 検索で取得されたコンテキスト ===')
print(context)
```

### Section 6：Gradio UI の起動

セルを実行すると `Running on public URL: https://xxxxxx.gradio.live` と表示されます。  
そのURLをブラウザで開くと、実際にチャットUIが表示されます。

> **Colab のノートブックを閉じると URL も無効になります。**  
> デモ中は Colab タブを開いたままにしてください。

---

## 5. つまずきやすいポイントと対処法

### エラー：`NameError: name 'vector_store' is not defined`

**原因：** Section 4 のセルが実行されていない  
**対処：** Section 1 から順番に実行し直す

### Section 4 でベクトル化が途中で止まる

**原因：** Gemini API のレートリミット（1分あたりのリクエスト上限）に達した可能性  
**対処：** 1〜2分待ってから再実行。または `chunk_size` を大きくしてチャンク数を減らす

```python
# チャンク数を減らしてレートリミットを回避
chunks = split_into_chunks(SAMPLE_FAQ, chunk_size=500)
```

### Gradio の URL が開かない

**原因：** セルの実行が完了していない、またはファイアウォールの制限  
**対処：**
- セル内に `Running on local URL:` と表示されているか確認
- ブラウザのポップアップブロックを無効にする
- `demo.launch(share=False)` に変更し、Colab 内のプレビューで確認

### 回答が「詳しくは窓口にお問い合わせください」ばかりになる

**原因：** 質問がFAQデータに含まれていない内容  
**対処：** `SAMPLE_FAQ` に質問に関連するQ&Aを追加してからベクトルストアを再構築する

```python
# FAQ を追加したら必ず再構築する
vector_store = FAQVectorStore()          # リセット
vector_store.add_documents(chunks)       # 再登録
rag_bot = MunicipalityRAGBot(vector_store)  # ボット再初期化
```

### ランタイム切断後の復旧手順

ランタイムが切断されると変数がすべてリセットされます。

1. 右上「再接続」をクリック
2. 「ランタイム」→「前に実行したセルをすべて実行」
3. Section 4 のベクトル化が完了するまで待つ（1〜2分）

---

## 6. 学習の手助けになるサイト

### RAG・ベクトル検索

| サイト | URL | 特徴 |
|--------|-----|------|
| LangChain 公式ドキュメント | https://python.langchain.com/docs/ | RAG の実装例が豊富 |
| ChromaDB 公式ドキュメント | https://docs.trychroma.com/ | 本講座で使うベクトル DB |
| Gemini Embedding API | https://ai.google.dev/gemini-api/docs/embeddings | Embedding の仕様 |
| RAG の仕組みを図解（AWS ブログ） | https://aws.amazon.com/jp/blogs/news/retrieval-augmented-generation/ | 日本語・わかりやすい |

### Gradio（UI 構築）

| サイト | URL | 特徴 |
|--------|-----|------|
| Gradio 公式ドキュメント | https://www.gradio.app/docs/ | コンポーネント一覧・APIリファレンス |
| Gradio ガイド | https://www.gradio.app/guides/ | チュートリアル形式 |
| Gradio でチャットUI（Zenn） | https://zenn.dev/topics/gradio | 日本語記事まとめ |

### LangChain・LLM フレームワーク

| サイト | URL | 特徴 |
|--------|-----|------|
| LangChain Python | https://python.langchain.com/ | LLM アプリ構築フレームワーク |
| LlamaIndex | https://docs.llamaindex.ai/ | RAG 特化フレームワーク |
| Haystack | https://haystack.deepset.ai/ | エンタープライズ向け RAG |

### 生成 AI・LLM 全般（学習リソース）

| サイト | URL | 特徴 |
|--------|-----|------|
| DeepLearning.AI ショートコース | https://www.deeplearning.ai/short-courses/ | 「LangChain」「RAG」等の無料コース |
| Hugging Face NLP コース | https://huggingface.co/learn/nlp-course/ja/ | 日本語対応・Transformer 基礎から |
| Google Machine Learning Crash Course | https://developers.google.com/machine-learning/crash-course?hl=ja | 日本語・基礎理論から |
| 松尾研 LLM 講座（東大） | https://weblab.t.u-tokyo.ac.jp/llm/ | 日本語・大学講義レベルの本格解説 |

### PDF 処理・テキスト前処理

| サイト | URL | 特徴 |
|--------|-----|------|
| PyMuPDF（fitz）ドキュメント | https://pymupdf.readthedocs.io/ | 本講座で使う PDF ライブラリ |
| pdfplumber | https://github.com/jsvine/pdfplumber | 表形式データに強い PDF ライブラリ |

### 自治体 AI・政府 OSS 関連

| サイト | URL | 特徴 |
|--------|-----|------|
| ガバメントAI「源内」リポジトリ | https://github.com/digital-go-jp/genai-web | 本講座のベース OSS |
| デジタル庁デザインシステム | https://design.digital.go.jp/ | 行政UI設計ガイドライン |
| 総務省 自治体 AI 活用ガイド | https://www.soumu.go.jp | 自治体AI導入の公式方針 |
| e-Gov 行政ポータル | https://www.e-gov.go.jp | 行政手続き情報の一次情報源 |

---

## Day 2 チェックリスト

演習終了前に確認してください。

- [ ] ノートブックを自分のドライブにコピーした
- [ ] Section 1〜6 を順番に実行した
- [ ] ChromaDB へのベクトル登録が完了した（`✅ ●個のチャンクをDBに登録完了`）
- [ ] RAG チェーンのテスト3問で回答が得られた
- [ ] Gradio の URL をブラウザで開いてチャットを体験した
- [ ] チャレンジ課題 A（精度評価）の表を埋めた
- [ ] RAG ありと RAG なしで回答の質がどう違うかを言葉で説明できる

---

> Day 3 では、今日作ったシステムをベースに、チームで担当業務テーマ（土木・福祉・教育など）を選んで  
> オリジナル機能を追加し、発表します。`day3_theme_customization.ipynb` を参照してください。
