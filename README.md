# CodeAgent

ローカル LLM（Ollama）で動く汎用コーディングエージェント。Web ブラウザ UI 付き。
MT5 (wine) の EA コンパイル・バックテストをツールとして組み込み済み。

## 構成

```
codeagent/
├── main.py                 # 起動スクリプト
├── config.yaml             # 設定（モデル・権限・MT5 パス）
├── codeagent/
│   ├── agent.py            # エージェントループ（LLM ⇄ ツール）
│   ├── llm.py              # Ollama / Anthropic クライアント
│   ├── prompts.py          # システムプロンプト
│   ├── session.py          # 会話履歴の保存（JSON）
│   ├── tools/
│   │   ├── fs.py           # read_file / write_file / edit_file / list_dir / glob / grep
│   │   ├── shell.py        # run_command
│   │   └── mt5.py          # mt5_compile / mt5_backtest
│   └── server/
│       ├── app.py          # FastAPI + WebSocket
│       └── static/index.html
```

## セットアップ

```bash
cd codeagent
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

`config.yaml` を環境に合わせて編集:

| 項目 | 内容 |
|---|---|
| `llm.model` | 使うモデル（例: `gemma4:26b`） |
| `llm.tool_mode` | `native`（Ollama tool calling）/ `json`（非対応モデル用フォールバック） |
| `workspace.root` | エージェントが触れる範囲。この外は拒否される |
| `workspace.default_project` | 新規セッションの既定プロジェクト |
| `permissions.*` | ツールごとに `allow` / `ask` / `deny` |
| `mt5.*` | WINEPREFIX と MT5 のインストール先 |

## 起動

```bash
python main.py
# → http://127.0.0.1:8765 をブラウザで開く
```

## 使い方

1. 左上でモデルを選び、プロジェクトディレクトリを入力して「新しいセッション」
2. 指示を入力（例: 「USDJPY H1 の移動平均クロス EA を作ってコンパイルして」）
3. `write_file` / `edit_file` / `run_command` は承認ダイアログが出る。「このセッションでは常に許可」で以降スキップ可能
4. ツールカードをクリックすると引数と結果が展開される
5. 「停止」で処理を中断

プロジェクト直下に `AGENT.md`（または `CLAUDE.md`）を置くと、その内容がシステムプロンプトに読み込まれる。EA の命名規則や共通ライブラリの説明などを書いておくと精度が上がる。

## tool calling が使えないモデルの場合

`llm.tool_mode: json` にすると、モデルに次の形式でツール呼び出しを書かせて解析する:

````
```tool_call
{"name": "read_file", "arguments": {"path": "a.py"}}
```
````

## MT5 ツール

- `mt5_compile`: `.mq5` を `MetaEditor64.exe /compile` でコンパイル。MQL5 フォルダ外のファイルは `MQL5/Experts/CodeAgent/` にコピーしてからコンパイルし、`.ex5` を元の場所にも戻す
- `mt5_backtest`: `terminal64.exe /config:tester.ini` でストラテジーテスターを実行し、HTML レポートから純利益・PF・DD などを抽出

いずれも `WINEPREFIX=~/.wine_oanda_mt5` を使う（config.yaml で変更可）。

## Claude API を使う場合

```yaml
llm:
  provider: anthropic
anthropic:
  model: claude-sonnet-4-5
```

環境変数 `ANTHROPIC_API_KEY` を設定して起動。

## ツールの追加

`codeagent/tools/` に新しいモジュールを作り、`register(Tool(...))` で登録して `tools/__init__.py` の `load_builtin()` に import を足す。
