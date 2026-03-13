---
title: "Claude Code Hooksで開発を自動化する：PreToolUse/PostToolUse 実践入門"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "ai", "automation", "productivity", "shellscript"]
published: true
---

# Claude Code Hooksで開発を自動化する：PreToolUse/PostToolUse 実践入門

Claude Codeを使い始めると、こんな場面に出くわしたことはないでしょうか。

- Claudeが「`rm -rf ./dist`」を実行しようとして、うっかり重要ファイルを消してしまった
- コードを修正したあと、自分でフォーマッターをかけるのを毎回忘れる
- 長時間の作業タスクを任せていると、確認が必要なときに気づかない

実際に私もこれらの問題を経験し、**Claude Code Hooks** を設定してから開発ワークフローが劇的に改善しました。この記事では、すぐに使えるレシピ集とともにHooksの仕組みを解説します。

:::message
**前提条件**: このスクリプトでは `jq`（JSONパーサー）を使用します。インストールされていない場合は事前に `brew install jq`（macOS）または `apt install jq`（Linux）でインストールしてください。
:::

Hooksは「AIの非決定論的な動作を、決定論的なコマンドで補完する」仕組み。  
CLAUDE.mdがClaudeへの「お願い」だとすれば、HooksはClaudeが何をしても**確実に実行される「命令」**です。

## CLAUDE.md vs Hooks：何が違うのか

まず混同しがちなCLAUDE.mdとの違いを整理しましょう。

| | CLAUDE.md | Hooks |
|--|-----------|-------|
| 役割 | Claudeへの指示・コンテキスト | ライフサイクルイベントへの処理追加 |
| 性質 | **アドバイス**（守られるとは限らない） | **命令**（必ず実行される） |
| 実行者 | Claude（LLM） | シェル（確定的） |
| 向いていること | コーディング規約・プロジェクト構造の説明 | 自動フォーマット・危険コマンドのブロック |

**CLAUDE.mdとHooksは対立するものではなく、組み合わせるもの**です。  
「コーディング規約はCLAUDE.mdで説明 → フォーマットはHooksで強制」が理想的な使い方。

## Claude Code Hooksの仕組み

### ライフサイクルイベント一覧

Hooksは以下のイベントにアタッチできます。

| イベント | タイミング |
|---------|----------|
| `SessionStart` | Claude Codeセッション開始時 |
| `UserPromptSubmit` | プロンプト送信直後 |
| `PreToolUse` | ツール実行**前** |
| `PostToolUse` | ツール実行**後** |
| `PermissionRequest` | 権限リクエスト発生時 |
| `Notification` | 通知発生時 |
| `Stop` | セッション終了時 |

最もよく使うのは **`PreToolUse`（実行前チェック）** と **`PostToolUse`（実行後の後処理）** です。

### 設定ファイルの場所と優先度

設定ファイルは3種類あり、優先度は次の通りです。

```
settings.local.json > .claude/settings.json > ~/.claude/settings.json
```

| ファイル | 用途 | Gitコミット |
|--------|------|------------|
| `.claude/settings.json` | プロジェクト共有設定（チーム全員に適用） | ✅ する |
| `.claude/settings.local.json` | ローカル個人設定（自分専用の調整） | ❌ しない |
| `~/.claude/settings.json` | グローバル個人設定（全プロジェクト共通） | - |

チームの開発規約を強制したい場合は `.claude/settings.json` をコミットし、  
個人的なカスタマイズは `.claude/settings.local.json` に分けるのがベストプラクティスです。

### 設定の基本構造

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-bash-guard.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/post-format.sh"
          }
        ]
      }
    ]
  }
}
```

**`matcher`** は正規表現で対象ツールを絞り込めます。  
例：`"Bash"` → Bashツールのみ、`"Edit|Write"` → ファイル編集・書き込み時

### exit codeによる動作制御

フックスクリプトの**終了コード**でClaude Codeの動作を制御します。

| exit code | 動作 |
|-----------|------|
| `0` | 通過（通常実行を許可） |
| `1` | 警告（stderrを表示するが実行は続ける） |
| `2` | **ブロック**（ツール実行を拒否。stderrがClaudeに伝わる） |

また、フックにはツール入力の詳細が**JSON形式でstdinに渡されます**。

```bash
# PreToolUse時にstdinに渡されるJSON例
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf ./dist"
  }
}
```

`jq`でこのJSONをパースすることで、コマンド内容に応じた条件分岐が可能です。

---

## 実践レシピ集

コピペして使える5つのレシピを紹介します。

### レシピ1: 危険コマンドのブロック（PreToolUse）

最も重要なユースケース。`rm -rf`や`git push --force`などの危険なコマンドをClaudeが実行しようとしたとき、**実行前にブロック**します。

まず `.claude/settings.json` に設定を追加：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-bash-guard.sh"
          }
        ]
      }
    ]
  }
}
```

次にスクリプトを作成：

```bash
#!/usr/bin/env bash
# .claude/hooks/pre-bash-guard.sh
set -euo pipefail

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')

# Bash以外は対象外
if [ "$TOOL_NAME" != "Bash" ]; then exit 0; fi

COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if [ -z "$COMMAND" ]; then exit 0; fi

# 危険パターンの定義
declare -a DANGEROUS_PATTERNS=(
  "rm[[:space:]]+-rf[[:space:]]+/"     # ルート削除
  "rm[[:space:]]+-rf[[:space:]]+~"     # ホームディレクトリ削除
  "git[[:space:]]+push.*--force.*main" # mainへのforce push
  "DROP[[:space:]]+TABLE"              # DB削除
  "truncate[[:space:]]+/"              # ディスク上書き
)

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$COMMAND" | grep -qiE "$pattern"; then
    echo "🚫 危険なコマンドをブロックしました: $COMMAND" >&2
    echo "パターン: $pattern" >&2
    exit 2
  fi
done

exit 0
```

```bash
chmod +x .claude/hooks/pre-bash-guard.sh
```

:::message
**ポイント**: `exit 2` でブロックすると、stderrのメッセージがClaude Codeに伝わります。  
Claudeは「なぜブロックされたか」を理解し、代替手段を提案してくれます。
:::

---

### レシピ2: 編集後に自動フォーマット（PostToolUse）

ファイル編集・書き込み後に自動でフォーマッターを実行します。  
「あとで一括フォーマット」を忘れなくなります。

```bash
#!/usr/bin/env bash
# .claude/hooks/post-format.sh
set -uo pipefail

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [ -z "$FILE_PATH" ] || [ ! -f "$FILE_PATH" ]; then exit 0; fi

case "$FILE_PATH" in
  *.py)
    # Pythonは black + isort
    black "$FILE_PATH" 2>/dev/null && isort "$FILE_PATH" 2>/dev/null
    ;;
  *.ts|*.tsx|*.js|*.jsx)
    # TypeScript/JavaScriptはprettier
    npx prettier --write "$FILE_PATH" 2>/dev/null
    ;;
  *.go)
    # Goはgofmt
    gofmt -w "$FILE_PATH" 2>/dev/null
    ;;
esac

exit 0
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/post-format.sh"
          }
        ]
      }
    ]
  }
}
```

:::message alert
**注意**: フォーマッターがインストールされていない場合は `2>/dev/null` でエラーを握り潰しています。  
本番で使う場合は、インストール確認を追加することをおすすめします。
:::

---

### レシピ3: テストファイル編集後に自動テスト実行（PostToolUse）

テストファイルを修正したらすぐにテストが走ります。  
「直したつもりがテスト通ってなかった」問題を防げます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'FILE=$(jq -r \".tool_input.file_path // empty\"); if [[ \"$FILE\" == *test* || \"$FILE\" == *spec* ]]; then pytest -x --tb=short \"$FILE\" 2>&1 | tail -20; fi'"
          }
        ]
      }
    ]
  }
}
```

:::message
**ヒント**: `pytest -x` は最初の失敗で止まるオプション。  
大量のテスト出力でClaude Codeのコンテキストを汚染しないよう `tail -20` で最後20行だけ見せています。
:::

---

### レシピ4: 確認が必要なときにデスクトップ通知（Notification）

長時間タスクを任せているとき、Claudeが確認を求めて止まっていても気づけないことがあります。  
NotificationフックでOS通知を飛ばしましょう。

**macOS版：**

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code が確認を求めています\" with title \"Claude Code\" sound name \"Glass\"'"
          }
        ]
      }
    ]
  }
}
```

**Linux（notify-send）版：**

```json
{
  "hooks": {
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' 'Claude Code が確認を求めています'"
          }
        ]
      }
    ]
  }
}
```

:::message
`~/.claude/settings.json` に書くと、すべてのプロジェクトで有効になります。
:::

---

### レシピ5: セッション終了時の作業ログ記録（Stop）

セッション終了時に日時を自動でログに残します。  
「今日どのプロジェクトでどれだけ作業したか」の記録になります。

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[$(date '+%Y-%m-%d %H:%M:%S')] Session ended in $(pwd)\" >> ~/.claude/session-log.txt"
          }
        ]
      }
    ]
  }
}
```

---

## ハマりポイントと注意事項

Hooksを使い始めたときに詰まりやすいポイントをまとめます。

### 1. JSONの中のダブルクォートエスケープ

`command`にシェルコマンドを直接書く場合、ダブルクォートのエスケープが複雑になります。  
**複雑なロジックはスクリプトファイルに分離する**のがベストです。

:::message alert
以下のコードブロックは説明のための擬似コードです。JSONはコメントをサポートしないため、実際のファイルにコメントを書かないでください。
:::

```
# ❌ NG: 複雑なコマンドをJSONに直書き（エスケープが地獄になる）
{
  "command": "if [[ \"$FILE\" == *.py ]]; then black \"$FILE\"; fi"
}

# ✅ OK: スクリプトに分離（すっきり・デバッグしやすい）
{
  "command": ".claude/hooks/post-format.sh"
}
```

### 2. exit code 2でも必ずブロックされるわけではない

`PreToolUse`でexit code 2を返してもブロックできないケースがあります。  
- Claude Codeのバージョンによって挙動が異なる場合がある
- `--dangerously-skip-permissions`フラグが使われているとフックが無効になる

### 3. PostToolUseで重い処理をするとClaude Codeが遅くなる

PostToolUseはツール実行のたびに同期的に実行されます。  
テスト実行など重い処理をすると、Claude Codeの応答が遅くなります。  
対処法：タイムアウトを設ける、または非同期実行（`&`）を使う。

```bash
# タイムアウト付き実行
timeout 30 pytest -x "$FILE" || true
```

### 4. /hooksコマンドで設定するのが安全

Claude Codeのターミナル内で `/hooks` と入力すると、  
UIを使ってフックを設定できます。JSONの手書きエラーを防げます。

```
Claude Code > /hooks
```

---

## チームでの活用：開発規約をHooksで強制する

チーム開発での強力な使い方を紹介します。

`.claude/settings.json` をGitリポジトリにコミットすると、  
**チーム全員に同じHooksが自動適用**されます。

```bash
# 開発規約を強制するHooksの例
.claude/
├── settings.json          # Gitコミット（チーム共有）
│   └── hooks:
│       ├── PreToolUse: 危険コマンドブロック
│       └── PostToolUse: 自動フォーマット（チームのlinter）
├── settings.local.json    # .gitignore（個人設定）
│   └── hooks:
│       └── Notification: 個人のデスクトップ通知設定
└── hooks/
    ├── pre-bash-guard.sh  # Gitコミット
    └── post-format.sh     # Gitコミット
```

**チームに推奨するHooksセット：**

1. 危険コマンドのブロック（全員必須）
2. プロジェクトのlinterを自動実行（eslint, flake8等）
3. コミット前に自動フォーマット

これにより「フォーマットを忘れて差分が汚れる」「うっかり本番DBを消す」といった事故を防げます。

---

## まとめ

Claude Code Hooksは、AIコーディングの**「やってほしいことを確実に実行させる」**仕組みです。

| 目的 | 使うもの |
|------|---------|
| Claudeに規約を理解させる | CLAUDE.md |
| 規約を**強制**する | Hooks |
| ツール実行前のチェック | PreToolUse |
| ツール実行後の後処理 | PostToolUse |
| セッション管理 | SessionStart / Stop |

まず試すなら、この2つから始めるのがおすすめです：

1. **危険コマンドのブロック**（レシピ1）→ 安全性の確保
2. **自動フォーマット**（レシピ2）→ コード品質の維持

Hooksを使いこなすと、Claude Codeは単なる「AIアシスタント」から  
**チームの開発規約を守る「自律的なチームメンバー」**へと進化します。

ぜひ自分のプロジェクトに合わせたHooksを設定してみてください。

---

## 参考リンク

- [Claude Code 公式ドキュメント: Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks完全ガイド（SIOS Tech Lab）](https://tech-lab.sios.jp/archives/50794)
- [Claude Code Hooks 安全設計パターン](https://www.playpark.co.jp/blog/claude-code-hooks-safety-design)
