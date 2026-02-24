# LLM Papers Pipeline

LLM関連論文の自動収集・日本語要約・Hugo公開・Slack通知パイプライン。

## 概要

arXiv、Semantic Scholar、HuggingFace Daily Papersから毎日LLM関連論文を自動収集し、Claude Code CLIで日本語要約を生成、Hugo静的サイトとして公開、Slackに通知する。

```
[launchd: 毎日7:00]
       |
       v
[run_pipeline.sh]
       |
       v
[pipeline.py]
  |-- 1. 状態ファイル読み込み (state.py)
  |-- 2. 論文収集 (fetchers.py)
  |       |-- arXiv API
  |       |-- Semantic Scholar API
  |       |-- HuggingFace Daily Papers API
  |       |-- 重複排除 + 人気度ソート
  |-- 3. 処理済み論文の除外
  |-- 4. 日本語要約生成 (summarizer.py) ... claude -p
  |-- 5. スクリーンショット取得 (screenshot.py) ... Playwright
  |-- 6. Hugo記事生成 (publisher.py)
  |-- 7. 状態ファイル保存
  |-- 8. git commit → hugo build → git push
  |-- 9. Slack通知 (notifier.py) ... claude -p + Slack MCP
```

## セットアップ

### 前提条件

- Python 3.12+
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)（要約生成・Slack通知に使用）
- Hugo（ブログビルド）
- Playwright（スクリーンショット取得）

### インストール

```bash
git clone https://github.com/zatoima/llm-papers-pipeline.git
cd llm-papers-pipeline

python3.12 -m venv venv
./venv/bin/pip install -r requirements.txt
./venv/bin/python -m playwright install chromium
```

### 環境変数

| 変数 | 説明 | デフォルト |
|------|------|------------|
| `HUGO_PROJECT_ROOT` | Hugoプロジェクトのルートパス | スクリプトの3階層上 |

`run_pipeline.sh` 内で設定するか、実行前にexportする。

## 使い方

### 手動実行

```bash
# 通常実行
./venv/bin/python3 pipeline.py --verbose

# ドライラン（取得対象の確認のみ）
./venv/bin/python3 pipeline.py --dry-run

# スクリーンショットとpushをスキップ
./venv/bin/python3 pipeline.py --skip-screenshots --skip-push

# 取得数を指定
./venv/bin/python3 pipeline.py --max-papers 10
```

### CLIオプション

| オプション | 説明 |
|------------|------|
| `--verbose` | DEBUGレベルのログ出力 |
| `--skip-screenshots` | スクリーンショット取得をスキップ |
| `--skip-push` | git commit/pushをスキップ |
| `--dry-run` | 処理対象の論文を表示するのみ |
| `--max-papers N` | 取得する論文数を指定 |

### 定期実行（macOS launchd）

```bash
cp com.zatoima.llm-papers-pipeline.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.zatoima.llm-papers-pipeline.plist
```

## ディレクトリ構成

```
├── config.py               # 設定値の一元管理
├── fetchers.py             # 3ソースからの論文取得
├── summarizer.py           # Claude CLIによる要約生成
├── screenshot.py           # Playwrightによるスクリーンショット
├── publisher.py            # Hugo記事の生成
├── notifier.py             # Slack通知（Claude CLI + Slack MCP）
├── state.py                # 冪等性の状態管理
├── pipeline.py             # メインオーケストレーター
├── run_pipeline.sh         # launchdから呼ばれるシェルラッパー
├── requirements.txt        # Python依存パッケージ
├── processed_papers.json   # 処理済み論文のID記録（自動生成）
└── pipeline.log            # アプリケーションログ（自動生成）
```

## 論文ソースと人気度スコア

| ソース | API | 対象期間 | 人気度指標 | 最大取得数 |
|--------|-----|----------|------------|------------|
| arXiv | arxiv Python library | 最新50件 | S2 Batch APIで補完 | 50 |
| Semantic Scholar | Graph API v1 | 過去3日 | citationCount | 20/keyword |
| HuggingFace | Daily Papers API | 当日分 | upvotes | 全件 |

論文は人気度スコア（降順）→ 公開日（新しい順）でソートされ、上位N件（デフォルト5件）が選出される。

## 冪等性

`processed_papers.json` で処理済み論文のIDを管理し、同一論文の重複処理を防ぐ。

```json
{
  "processed_ids": {
    "arxiv:2602.10693": {
      "title": "VESPO: Variational Sequence-Level Soft Policy Optimization...",
      "processed_at": "2026-02-23T21:52:36.202101"
    }
  },
  "last_run": "2026-02-24T09:16:33.952783"
}
```

## 設定（config.py）

主要な設定値：

| 設定 | デフォルト | 説明 |
|------|------------|------|
| `TOP_N_PAPERS` | 5 | 取得する論文数 |
| `FETCH_WINDOW_DAYS` | 3 | Semantic Scholarの検索対象日数 |
| `REQUEST_DELAY` | 3.0秒 | APIコール間の待機時間 |
| `MAX_RETRIES` | 3 | リトライ回数 |
| `ARXIV_CATEGORIES` | cs.CL, cs.AI | arXivの対象カテゴリ |

## ブログ記事

パイプラインの詳細な技術解説：

https://zatoima.github.io/llm-papers-pipeline-with-claude-code/
