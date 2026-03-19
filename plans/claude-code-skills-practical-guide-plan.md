# 企画書: Claude Code Skills実践ガイド：SKILL.mdで定型作業をチームの武器にする

## メタ情報
- **作成日**: 2026-03-19
- **記事タイプ**: tech
- **読者レベル**: 中級者
- **slug**: claude-code-skills-practical-guide
- **emoji**: "🧩"
- **参照学習ノート**: 30_Resouces/zenn-research/claude-code-skills-research.md

## タイトル
- **採用案**: Claude Code Skills実践ガイド：SKILL.mdで定型作業をチームの武器にする
- **候補**:
  1. Claude Code Skills実践ガイド：SKILL.mdで定型作業をチームの武器にする
  2. Claude Code Skills入門：Progressive DisclosureでAIワークフローを肥大化させない
  3. Claude Code Skillsを作ってみた：再利用できる開発フローの設計とハマりどころ

## Topics（最大5個）
claudecode, claude, ai, productivity, automation

## ターゲット読者
- **レベル**: 中級者
- **前提知識**: Claude Codeの基本操作はできるが、長文プロンプトや定型作業の整理に悩んでいる
- **読後に得られるもの**:
  - Skillsと他機能の使い分けが分かる
  - `SKILL.md` の最小構成と設計ポイントが分かる
  - チームで再利用できるワークフロー資産の作り方が分かる

## 構成アウトライン

### はじめに
- 同じプロンプトを毎回打っていた体験から入る
- この記事で得られるものを3項目で提示する
- 対象読者はClaude Code初級を脱した人に設定する

### なぜSkillsが必要か：CLAUDE.mdに何でも書くと破綻する
- 常時コンテキストと必要時コンテキストの違いを説明する
- `CLAUDE.md` / Slash Commands / Hooks / Skills の住み分けを表で整理する
- 図: Mermaid flowchart で「常時ロード」と「必要時ロード」を可視化する

### 実際にやってみた: 最小のSkillを1つ作る
- `.claude/skills/<name>/SKILL.md` を作る流れを示す
- `name` と `description` の重要性を説明する
- コード例: 最小の research skill を掲載する
- 図: ディレクトリ構成のASCII図

### 実際にやってみた: チームで使えるSkillに育てる
- steps / 完了条件 / templates / scripts を足していく流れを示す
- Progressive Disclosure を意識した粒度の切り方を説明する
- 図: Mermaid graph で Skill と supporting files の関係を示す

### ハマりポイント・注意事項
- description が曖昧で呼ばれない
- 1つのSkillに責務を詰め込みすぎる
- 手順だけで検証ステップがなく、再利用されなくなる
- 強調: :::message alert で「まず小さく作る」「1 Skill 1責務」を警告する

### まとめ
- CLAUDE.md / Commands / Hooks / Skills の使い分けを表で再整理する
- 次のステップとして、自分の繰り返し作業を1つだけSkill化する行動を勧める

## 体験・失敗談の材料
- **実際にやったこと**: 既存のClaude Code関連記事群を見直し、まだ空いているテーマを調べたうえでSkill中心の記事設計をした
- **ハマった点**: 最初はSlash Commandsの記事と差別化しづらく、単なる「便利機能紹介」になりそうだった
- **解決方法**: 「Progressive Disclosure」「再利用可能なワークフロー資産」「CLAUDE.mdとの住み分け」に軸を寄せた
- **気づいたこと**: 技術そのものよりも、どの情報を常時読ませ、どの手順を必要時だけ読むかの設計が本質だった

## 図の計画（最低3つ）
| セクション | 図の種類 | 内容 |
|-----------|---------|------|
| なぜSkillsが必要か | Mermaid flowchart | 常時ロードされる情報と必要時ロードされる情報の違い |
| 実際にやってみた: 最小のSkillを1つ作る | ASCII | `.claude/skills/` 配下のディレクトリ構成 |
| 実際にやってみた: チームで使えるSkillに育てる | Mermaid graph | `SKILL.md`, templates, scripts の関係 |
| まとめ | テーブル | CLAUDE.md / Commands / Hooks / Skills の住み分け |

## 差別化ポイント
- 既存記事のSlash CommandsやHooksと競合せず、Claude Codeの「ワークフロー資産化」に焦点を当てる
- 2026年のトレンドであるSkills活用とProgressive Disclosureを、実務でどう使うかまで落とし込む
