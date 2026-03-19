# 企画書: Claude CodeでPRレビューを分業する：設計・テスト・セキュリティをGitHub Actionsに載せる実践ガイド

## メタ情報
- **作成日**: 2026-03-19
- **記事タイプ**: tech
- **読者レベル**: 中級者
- **slug**: claude-code-pr-review-github-actions-20260319
- **emoji**: 🧪
- **参照学習ノート**: 30_Resouces/zenn-research/claude-code-pr-review-github-actions-research.md

## タイトル
- **採用案**: Claude CodeでPRレビューを分業する：設計・テスト・セキュリティをGitHub Actionsに載せる実践ガイド
- **候補**:
  1. Claude CodeでPRレビューを分業する：設計・テスト・セキュリティをGitHub Actionsに載せる実践ガイド
  2. Claude CodeでPRレビュー待ちを減らす：レビュー観点をGitHub Actionsで分業する方法
  3. Claude Codeで自動PRレビューを育てる：GitHub Actionsにレビュー基準を落とし込む実践メモ

## Topics（最大5個）
claudecode, github-actions, cicd, automation, ai

## ターゲット読者
- **レベル**: 中級者
- **前提知識**: Claude Codeを触ったことがあり、GitHub Actions で `on:` と `jobs:` を読める
- **読後に得られるもの**:
  - Claude Codeの managed review と self-hosted review の違い
  - GitHub Actions で PR レビューを自動化する最小構成
  - `CLAUDE.md` / `REVIEW.md` を使ってレビュー基準を育てる方法

## 構成アウトライン

### はじめに
- PRレビューが属人化すると、待ち時間と観点の抜け漏れが同時に増える
- Claude Code を「人の代わり」ではなく「観点の分担役」として使う話だと先に明言する
- GitHub Actions と REVIEW.md を使えば、レビュー基準を repo に残せると伝える

### なぜレビュー待ちが起きるのか
- 設計、テスト、セキュリティを1人のレビュー担当に背負わせると詰まりやすい
- managed review と self-hosted review の違いを整理する
- 図: Mermaid flowchart で「人だけのレビュー」と「Claude併用レビュー」の流れ比較

### 実際にやってみた: GitHub Actions の最小構成
- `anthropics/claude-code-action@v1` を使う
- `ANTHROPIC_API_KEY`、permissions、draft除外、timeout を入れる
- コード例: workflow YAML
- 図: sequenceDiagram で PR 作成からコメント返却まで

### 実際にやってみた: レビュー観点を repo に落とし込む
- `CLAUDE.md` は全体方針、`REVIEW.md` はレビュー専用ルール
- 「設計」「テスト」「セキュリティ」の3観点をどう言語化するかを見せる
- コード例: `REVIEW.md`
- 図: テーブルで CLAUDE.md / REVIEW.md / prompt の役割分担

### ハマりポイント・注意事項
- push のたびにレビューしてコストが増えた
- permissions を広げすぎると怖い
- 指示が広すぎるとノイズコメントが増える
- 強調: :::message alert

### まとめ
- 自動レビューは「人の代替」ではなく「レビュー待ちを減らす補助線」
- managed review と self-hosted review の選び方を表で整理する
- 次のステップとして、既存の Claude Code / GitHub Actions 記事に誘導する

## 体験・失敗談の材料
- **実際にやったこと**: 公式ドキュメントを確認しながら、Claude Code Action v1 の最小PRレビュー構成と `REVIEW.md` 設計を整理した
- **ハマった点**: 最初は `sub-agents` 前提で書こうとして、GitHub Actions の公式ドキュメントがそこを主軸にしていないことに気づいた
- **解決方法**: 「観点の分業」と「specialized agents / REVIEW.md / prompt 設計」に言い換え、一次情報に寄せた
- **気づいたこと**: 既存記事との差別化は、ツール紹介ではなく運用設計に寄せる方が出しやすい

## 図の計画（最低3つ）
| セクション | 図の種類 | 内容 |
|-----------|---------|------|
| なぜレビュー待ちが起きるのか | Mermaid flowchart | 人手レビューだけの場合と Claude 併用時の流れ |
| 実際にやってみた: GitHub Actions の最小構成 | Mermaid sequenceDiagram | PR作成 → Actions起動 → Claudeレビュー → コメント返却 |
| 実際にやってみた: レビュー観点を repo に落とし込む | テーブル | `CLAUDE.md` / `REVIEW.md` / `prompt` の役割分担 |

## 差別化ポイント
- 既存の Copilot CLI × GitHub Actions 記事とは違い、**Claude Code Action v1 と REVIEW.md 運用** に焦点を当てる
- 既存の Hooks / Skills / CLAUDE.md 記事とは違い、**PRレビューの待ち時間とレビュー観点の分業**を扱う
- managed Code Review と self-hosted GitHub Actions の比較まで入れて、読者が選択できるようにする

## 近い既存記事
- CI/CDにAIを組み込む！GitHub Copilot CLI × GitHub Actions 実践入門 - GitHub ActionsでAIレビューを回す構図が近い
- Claude Code Hooksで開発を自動化する：PreToolUse/PostToolUse 実践入門 - Claude Code自動化という大枠が近い
- CLAUDE.mdを制する者がClaude Codeを制す：階層設計から実例テンプレートまで - ルールを文章化する発想が近い

## 関連記事候補
- CI/CDにAIを組み込む！GitHub Copilot CLI × GitHub Actions 実践入門（https://zenn.dev/biki/articles/github-actions-copilot-cli）
- Claude Code Hooksで開発を自動化する：PreToolUse/PostToolUse 実践入門（https://zenn.dev/biki/articles/claude-code-hooks-workflow-automation）
- CLAUDE.mdを制する者がClaude Codeを制す：階層設計から実例テンプレートまで（https://zenn.dev/biki/articles/claude-code-claude-md-guide）
