---
title: "Claude Codeカスタムスラッシュコマンド完全ガイド：.claude/commandsで繰り返し作業を撲滅する"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "claude", "ai", "tool", "cli"]
published: true
---

## はじめに

Claude Codeを使い始めて最初に感じる「あるある」があります。

**「毎回同じプロンプトをコピペしている……」**

「このコードをConventional Commits形式でコミットして」「ステージングされた変更をレビューして」——こういったプロンプトを何度も打ち込んだ経験はありませんか？

カスタムスラッシュコマンドを使えば、この問題をまるごと解決できます。`.claude/commands/review.md` というファイルを置くだけで `/review` と打つだけでいつものレビューが実行できるようになります。しかも、このファイルをgitに含めれば**チーム全員が同じコマンドを使える共有ライブラリ**になります。

この記事で得られるもの：

- ✅ カスタムスラッシュコマンドの仕組みとファイル構造の理解
- ✅ そのままコピーして使える実践コマンド10選
- ✅ `allowed-tools` によるセキュリティ制御の方法
- ✅ チームで共有するコマンドライブラリの設計・管理方法
- ✅ ハマりがちなポイントとその解決策

**対象読者**: Claude Codeをある程度使っている方（インストール・基本操作は済んでいる想定）

:::message
Claude CodeのバージョンによってはUIや一部動作が異なる場合があります。本記事は2026年3月時点の情報を元に執筆しています。
:::

---

## なぜカスタムスラッシュコマンドが必要か

### 毎回のコピペが開発の流れを断ち切る

Claude Codeは非常に強力ですが、使い込んでいくと「同じプロンプトを何度も書いている」という場面が増えてきます。

たとえば私の場合、以下のようなプロンプトを毎日何十回も入力していました：

```
ステージングされた変更をレビューしてください。
バグ、セキュリティ問題、テスト漏れの観点でチェックして、
問題があれば重要度（高/中/低）を付けて報告してください。
```

これをコピペするたびに、頭の中にある「今考えていること」が一度途切れます。地味ですが、1日に何十回もこれをやると積み重ねでかなりのストレスになります。

### カスタムコマンドで何が変わるか

カスタムスラッシュコマンドを使うと、上記のプロンプトが **`/review`** の一言になります。

```
# Before（毎回打つ）
ステージングされた変更をレビューしてください。バグ、セキュリティ問題...（以下省略）

# After（コマンド一発）
/review
```

さらに、プロジェクト固有のルール（「このプロジェクトではTypeScriptのstrictモードを使っています」など）もコマンドに埋め込めます。チーム全員が同じ品質基準でAIを活用できるようになります。

### ビルトインコマンドとカスタムコマンドの違い

Claude Codeにはあらかじめ組み込まれたコマンド（ビルトイン）があります。カスタムコマンドはその**拡張版**です。

| 種類 | 例 | 特徴 |
|------|-----|------|
| ビルトイン | `/compact`, `/clear`, `/cost`, `/model` | Claude Code本体に含まれる。変更不可 |
| カスタム（プロジェクト） | `/review`, `/commit`, `/deploy` | `.claude/commands/` に置く。gitで共有可能 |
| カスタム（個人） | `/my-review` | `~/.claude/commands/` に置く。全プロジェクトで使用可 |

---

## カスタムコマンドの基本構造

### ディレクトリ構造

カスタムコマンドは、特定のディレクトリにMarkdownファイルを置くだけで有効になります。

```
プロジェクトルート/
├── .claude/
│   └── commands/                  # ← プロジェクト共有コマンド
│       ├── review.md              #   → /review
│       ├── commit.md              #   → /commit
│       ├── gen-test.md            #   → /gen-test
│       └── deploy.md              #   → /deploy
├── src/
└── ...

~/.claude/
└── commands/                      # ← 個人コマンド（全プロジェクトで有効）
    └── my-snippet.md              #   → /my-snippet
```

**ポイント**:
- `.claude/commands/` は**gitリポジトリに含めてOK**。チームで共有できます
- `~/.claude/commands/` は**個人設定**。自分だけのショートカット用
- 同名コマンドがある場合は**プロジェクト側が優先**されます

### コマンドファイルの書き方

コマンドファイルはMarkdown形式です。YAMLフロントマター（オプション）とプロンプト本文から構成されます。

```markdown
---
description: "コードレビューを実行する"
argument-hint: "[file]"
allowed-tools: [Bash(git diff:*), Read]
---

ステージングされた変更をレビューしてください。

$ARGUMENTS が指定された場合はそのファイルのみを対象にしてください。

以下の観点でチェックしてください：
- バグ・ロジックエラー
- セキュリティ問題
- テスト漏れ
- パフォーマンス問題

問題があれば重要度（🚨高・⚠️中・💡低）を付けて報告してください。
```

### フロントマターの各フィールド

| フィールド | 必須 | 説明 |
|-----------|:----:|------|
| `description` | 推奨 | `/help` 一覧に表示される説明文 |
| `argument-hint` | 任意 | 引数のヒント（例: `<file> [severity]`） |
| `allowed-tools` | 任意 | 使用可能なツールのホワイトリスト |

フロントマターを省略して、プロンプト本文だけでもコマンドとして機能します：

```markdown
# review.md（フロントマターなし・最小構成）

ステージングされた変更をレビューして、問題点を報告してください。
```

---

## 引数を使いこなす

### `$ARGUMENTS` で柔軟なコマンドに

`$ARGUMENTS` は、コマンド呼び出し時にユーザーが入力したすべての引数を受け取る特殊変数です。

```markdown
# explain.md
---
description: "指定したコードを説明する"
argument-hint: "<コードまたはファイルパス>"
---

以下を分かりやすく説明してください：

$ARGUMENTS

説明の際：
- 何をしているコードか
- なぜこの実装が選ばれているか
- 初心者が疑問に思いそうな点
を含めてください。
```

使い方：

```
/explain src/utils/auth.ts
/explain useCallback フックについて
/explain この正規表現: ^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$
```

### 位置引数（`$1`, `$2`）

より細かく引数を制御したい場合は `$1`, `$2` を使います：

```markdown
# fix-issue.md
---
description: "指定したIssue番号の問題を修正する"
argument-hint: "<issue番号> [優先度]"
---

Issue #$1 を修正してください。
優先度: $2（指定がなければ通常）

まず現在のコードを確認してから、最小限の変更で修正してください。
```

使い方：

```
/fix-issue 123
/fix-issue 456 高
```

---

## allowed-toolsでセキュリティを制御する

### なぜ制限が必要か

カスタムコマンドはデフォルトでClaude Codeが使えるすべてのツール（Bash含む）を使用できます。しかし、コマンドによっては**意図しない副作用**が起きるリスクがあります。

たとえば「コードを説明して」というコマンドで、実際にはファイルの読み取りしか不要なのにBashコマンドの実行が許可されていると、セキュリティ上のリスクになり得ます。

`allowed-tools` でコマンドが使えるツールを明示的に指定することで、このリスクを低減できます。

### 書き方のパターン

```yaml
# パターン1: 特定ツールのみ許可
allowed-tools: [Read]

# パターン2: ReadとBash全体を許可
allowed-tools: [Read, Bash]

# パターン3: 特定のgitコマンドのみ許可（細粒度制御）
allowed-tools: [Bash(git diff:*), Bash(git add:*), Bash(git commit:*)]

# パターン4: 特定ディレクトリのファイルのみ読み取り許可
allowed-tools: [Read(src/**/*.ts), Bash(npx tsc:*)]
```

### 使い分けの目安

| コマンドの用途 | 推奨設定 |
|--------------|---------|
| コードを読んで説明・レビュー | `[Read]` |
| git操作（diff確認・コミット） | `[Read, Bash(git *:*)]` |
| テスト実行 | `[Bash(npm test:*), Bash(pytest:*)]` |
| ファイル生成・編集 | `[Read, Write]` |
| 汎用（何でもOK） | 省略（制限なし） |

---

## 実践コマンド集10選

実際に使えるコマンドを10個紹介します。そのままコピーして `.claude/commands/` に置けばすぐに使えます。

### 1. `/review` — コードレビュー

```markdown
# .claude/commands/review.md
---
description: "ステージングされた変更をレビューする"
allowed-tools: [Bash(git diff:*), Read]
---

ステージングされた変更をレビューしてください。

1. `git diff --staged` で変更内容を確認
2. 以下の観点でチェック：
   - 🚨 **重大**: バグ・セキュリティ問題・データ損失リスク
   - ⚠️ **注意**: パフォーマンス問題・エラーハンドリング漏れ
   - 💡 **改善提案**: 可読性・テスト・リファクタリング

問題がなければ「✅ LGTM」と報告してください。
```

### 2. `/commit` — Conventional Commitsに従ったコミット

```markdown
# .claude/commands/commit.md
---
description: "Conventional Commits形式でコミットを作成する"
argument-hint: "[type] [message]"
allowed-tools: [Bash(git add:*), Bash(git commit:*), Bash(git diff:*)]
---

ステージングされた変更を確認して、Conventional Commits形式でコミットしてください。

変更内容を `git diff --staged` で確認してから、適切なコミットタイプを選択：
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメントのみの変更
- `style`: フォーマット変更（コードの動作に影響しない）
- `refactor`: リファクタリング
- `test`: テストの追加・修正
- `chore`: ビルドプロセスや補助ツールの変更

$ARGUMENTS が指定された場合はそれをヒントとして使用してください。

コミットメッセージを提案し、確認後に `git commit -m "..."` を実行してください。
```

### 3. `/gen-test` — テスト自動生成

```markdown
# .claude/commands/gen-test.md
---
description: "指定ファイルのユニットテストを生成する"
argument-hint: "<file-path>"
allowed-tools: [Read, Write]
---

`$ARGUMENTS` のユニットテストを生成してください。

手順：
1. ファイルを読み込んでコードの構造を把握
2. テストすべき関数・メソッドをリストアップ
3. 以下を含む包括的なテストを生成：
   - 正常系（期待通りの動作）
   - 異常系（エラー・例外）
   - エッジケース（空文字・null・境界値）
4. テストフレームワークはプロジェクトの既存設定に合わせる
5. `$ARGUMENTS` と同ディレクトリに `{filename}.test.{ext}` として保存

既存のテストファイルがある場合は、スタイルを合わせてください。
```

### 4. `/doc` — ドキュメントコメント生成

```markdown
# .claude/commands/doc.md
---
description: "関数・クラスにドキュメントコメントを追加する"
argument-hint: "<file-path>"
allowed-tools: [Read, Write]
---

`$ARGUMENTS` のコードにドキュメントコメントを追加してください。

- JSDoc / docstring 形式（言語に合わせる）
- 各公開関数・メソッドに追加
- パラメータ名・型・説明
- 戻り値の型・説明
- 使用例（`@example`）
- 既存のコメントは上書きしない（ない場合のみ追加）

ファイルを上書き保存してください。
```

### 5. `/refactor` — リファクタリング提案

```markdown
# .claude/commands/refactor.md
---
description: "コードのリファクタリングを提案する"
argument-hint: "<file-path>"
allowed-tools: [Read]
---

`$ARGUMENTS` を読み込んで、リファクタリングの提案をしてください。

観点：
- 重複コードの排除（DRY原則）
- 関数・クラスの責務の明確化（単一責任原則）
- 複雑な条件分岐の簡素化
- マジックナンバー・マジック文字列の定数化
- 命名の改善（変数名・関数名）

**提案のみ行い、実際には変更しないでください。**
変更してよい場合は「変更して」と追加で指示してください。
```

### 6. `/explain` — コード説明

```markdown
# .claude/commands/explain.md
---
description: "コードやファイルを日本語で説明する"
argument-hint: "<file-path or コード>"
allowed-tools: [Read]
---

以下を分かりやすく日本語で説明してください：

$ARGUMENTS

説明に含めること：
- 何をするコード・ファイルか（一言サマリー）
- 主要な処理の流れ
- 重要な設計上の判断とその理由
- 初心者が疑問に思いそな点の補足
```

### 7. `/pr-desc` — プルリクエスト説明文の生成

```markdown
# .claude/commands/pr-desc.md
---
description: "現在のブランチのPR説明文を生成する"
allowed-tools: [Bash(git log:*), Bash(git diff:*)]
---

現在のブランチとmainブランチの差分を元に、プルリクエストの説明文を生成してください。

1. `git log main..HEAD --oneline` でコミット履歴を確認
2. `git diff main...HEAD` で変更内容を確認
3. 以下の形式で説明文を作成：

## 概要
[変更内容の一言サマリー]

## 変更内容
- [変更点1]
- [変更点2]

## テスト方法
[動作確認の手順]

## 注意事項（あれば）
[レビュアーへの補足]
```

### 8. `/perf` — パフォーマンス分析

```markdown
# .claude/commands/perf.md
---
description: "コードのパフォーマンス問題を分析する"
argument-hint: "<file-path>"
allowed-tools: [Read]
---

`$ARGUMENTS` のパフォーマンス上の問題点を分析してください。

チェック観点：
- 不必要なループ・ネストされたループ（O(n²)以上）
- 無駄なデータコピー・メモリアロケーション
- データベースN+1問題
- キャッシュ可能な計算の繰り返し
- 非同期処理の最適化余地

問題が見つかれば、改善案のコードも提示してください。
```

### 9. `/security` — セキュリティチェック

```markdown
# .claude/commands/security.md
---
description: "コードのセキュリティ問題をチェックする"
argument-hint: "<file-path>"
allowed-tools: [Read]
---

`$ARGUMENTS` のセキュリティ問題をチェックしてください。

チェック観点：
- SQLインジェクション・XSS・CSRF
- 認証・認可の漏れ
- シークレット・パスワードのハードコーディング
- 入力値の検証不足
- 安全でない依存パッケージの使用
- 権限の過剰付与

問題を発見した場合は、重大度（Critical/High/Medium/Low）と修正方法を示してください。
```

### 10. `/changelog` — CHANGELOG生成

```markdown
# .claude/commands/changelog.md
---
description: "最新タグからのCHANGELOGを生成する"
argument-hint: "[version]"
allowed-tools: [Bash(git log:*), Bash(git tag:*), Read, Write]
---

最新タグから現在のHEADまでのCHANGELOGを生成してください。

1. `git tag --sort=-v:refname | head -1` で最新タグを確認
2. `git log {最新タグ}..HEAD --pretty=format:"%s"` でコミット履歴を取得
3. Conventional Commitsを元に分類：

## [バージョン] - $(date +%Y-%m-%d)

### 🚀 Features
- [feat: から始まるコミット]

### 🐛 Bug Fixes
- [fix: から始まるコミット]

### 📝 Documentation
- [docs: から始まるコミット]

### ♻️ Refactoring
- [refactor: から始まるコミット]

バージョンは $ARGUMENTS が指定された場合それを使用、なければ "Unreleased" とする。
CHANGELOG.md の先頭に追記してください。
```

---

## チームでのコマンドライブラリ管理

### gitに含めてチームで共有する

`.claude/commands/` ディレクトリをgitにコミットすることで、チーム全員が同じコマンドを使えるようになります。

```bash
# コマンドを追加してコミット
mkdir -p .claude/commands
echo "レビューコマンド..." > .claude/commands/review.md
git add .claude/commands/
git commit -m "feat: Claude Codeカスタムコマンドを追加"
git push
```

これだけで、チームメンバーが `git pull` するたびに最新のコマンドが手元に届きます。

### README.mdにコマンド一覧を記載する

コマンドの存在をチームに周知するため、README.mdにリストを追加すると親切です：

````markdown
## 🤖 Claude Code コマンド一覧

`.claude/commands/` に以下のカスタムコマンドがあります：

| コマンド | 説明 | 使い方 |
|---------|------|-------|
| `/review` | コードレビュー | `/review` または `/review src/auth.ts` |
| `/commit` | コミット作成 | `/commit` |
| `/gen-test` | テスト生成 | `/gen-test src/utils.ts` |
| `/pr-desc` | PR説明文生成 | `/pr-desc` |
````

### 個人コマンドとチームコマンドの使い分け

```mermaid
graph TD
    A[コマンドを作りたい] --> B{チームで使う?}
    B -->|Yes| C[.claude/commands/ に置く]
    B -->|No| D[~/.claude/commands/ に置く]
    C --> E[gitにコミット → チーム共有]
    D --> F[個人環境のみで有効]
    C --> G[プロジェクト固有のルールを埋め込む]
    D --> H[自分のスニペット・ショートカット]
```

---

## ハマりポイント・注意事項

実際にカスタムコマンドを使い始めて、私がハマった点を正直に書きます。

### ハマり1: コマンドが認識されない

**何が起きたか**: `.claude/commands/review.md` を作ったのに `/review` と打っても「そのコマンドは存在しません」と言われた。

**原因**: ファイルのパスが間違っていた。プロジェクトルートではなく、`src/` 配下に作ってしまっていた。

**解決方法**:

```bash
# 正しいパスを確認
ls -la .claude/commands/

# .claude/が存在するか確認（プロジェクトルートで実行）
find . -name "*.md" -path "*/.claude/commands/*" 2>/dev/null
```

:::message alert
**`.claude/` はプロジェクトのルートディレクトリ（`git init` したディレクトリ）に置くこと。**
サブディレクトリに置いてもそのプロジェクトでは認識されません。
:::

### ハマり2: allowed-toolsが厳しすぎてコマンドが動かない

**何が起きたか**: `/commit` コマンドを作ったのに「このツールへのアクセスが許可されていません」とエラーになった。

**原因**: `allowed-tools: [Bash(git commit:*)]` だけを指定していたが、`git diff` や `git add` も必要だった。

**解決方法**: 必要なコマンドを一通り洗い出してから `allowed-tools` を設定する：

```yaml
# commit.md のフロントマター
---
allowed-tools: [Bash(git diff:*), Bash(git add:*), Bash(git commit:*), Bash(git status:*)]
---
```

**コツ**: まず `allowed-tools` を省略して動作確認してから、必要なツールを特定して制限を追加するほうが楽です。

### ハマり3: `$ARGUMENTS` が空のとき変な動作をする

**何が起きたか**: `/explain` を引数なしで実行したら、「`$ARGUMENTS` を読み込もうとしたが存在しない」というエラーが出た。

**解決方法**: コマンドに引数なしの場合のフォールバックを記述しておく：

```markdown
# explain.md

$ARGUMENTS が指定された場合はそれを説明してください。
指定がない場合は「何を説明しますか？ファイルパスかコードスニペットを入力してください」と聞いてください。
```

### ハマり4: 同名コマンドの優先順位

**何が起きたか**: `~/.claude/commands/review.md`（個人）と `.claude/commands/review.md`（プロジェクト）の両方があり、どちらが実行されるか混乱した。

**ルール**: **プロジェクトコマンドが優先**されます。同名コマンドがある場合、プロジェクトの `.claude/commands/` 側が使われます。

---

## まとめ

Claude Codeのカスタムスラッシュコマンドは、**「プロンプトのスニペット管理システム」**です。

| 機能 | できること |
|------|----------|
| コマンドの作り方 | `.claude/commands/{name}.md` を置くだけ |
| チーム共有 | `.claude/commands/` をgitにコミットする |
| 個人用 | `~/.claude/commands/` に置く |
| 引数 | `$ARGUMENTS`, `$1`, `$2` で受け取る |
| セキュリティ | `allowed-tools` で使用ツールを制限する |
| 優先順位 | プロジェクトコマンド > 個人コマンド |

最初は `/review` と `/commit` の2つだけ作ってみてください。毎日使っているうちに「このプロンプトもコマンド化したい」という場面が必ず出てきます。その積み重ねで、自分とチームに最適化されたコマンドライブラリが育っていきます。

### 次のステップ

カスタムコマンドに慣れたら、以下もぜひ試してみてください：

- **CLAUDE.md** — プロジェクト全体のルール・コンテキストを定義（[関連記事](https://zenn.dev/biki/articles/claude-code-claude-md-guide)）
- **Hooks** — PreToolUse/PostToolUseでコマンド実行前後に自動処理を挿入（[関連記事](https://zenn.dev/biki/articles/claude-code-hooks-workflow-automation)）
- **MCP Server** — 外部ツール・APIとClaude Codeを接続（[関連記事](https://zenn.dev/biki/articles/mcp-python-server-claude-code-guide)）

カスタムコマンドは「Claude Codeをチームの標準ツールにする」ための第一歩です。ぜひ自分のチームに合わせたコマンドライブラリを育ててみてください！
