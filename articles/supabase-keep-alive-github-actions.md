---
title: "7日で止まるSupabase無料プランをGitHub Actionsで永続化してみた"
emoji: "🔄"
type: "tech"
topics: ["supabase", "githubactions", "個人開発", "devops", "cicd"]
published: true
---

## はじめに

個人開発で Supabase を使っていると、久しぶりにアプリを開いたときに「なぜか繋がらない…」という経験をしたことはありませんか？

その原因の多くは **Supabase 無料プランの自動停止（Pause）機能** です。

この記事では、Supabase の Pause ポリシーを理解した上で、**GitHub Actions の cron スケジュールで毎日 API ping を送り続ける最小構成**を実際に試してみました。

### 対象読者

- Supabase を個人開発・学習用途で使っている方
- GitHub Actions を使ったことはあるが cron スケジュールは初めての方
- できるだけシンプルな構成で自動化したい方

### この記事で得られること

- Supabase 無料プランの停止ポリシーの全貌
- `curl` 1コマンドだけで実現する最小 keep-alive 構成
- 見落としがちな「GitHub Actions 自体も止まる」問題とその対策

---

## Supabase 無料プランが止まる仕組み

### 7日間アクティビティがないと自動停止

Supabase の無料プランには、**7日間アクティビティがない場合にプロジェクトを自動で一時停止（Pause）する**仕組みがあります。

| 条件 | 内容 |
|------|------|
| 停止条件 | 7日間アクティビティなし |
| 通知 | 停止前にメール送信あり |
| 復元可能期間 | Pause から **90日以内**（ダッシュボードからワンクリック） |
| 90日経過後 | 復元ボタン消滅。バックアップのダウンロードのみ |
| 有料プラン | Pause なし |

:::message
アクティビティとしてカウントされるのは、REST/GraphQL/Realtime などの **API リクエスト**、**ダッシュボードへのアクセス**、**DB への直接クエリ**です。
アプリの静的フロントエンドへのアクセスは含まれません。
:::

### なぜ個人開発で問題になるか

- **週1回以下しか触らないプロジェクト**は普通に止まる
- 止まったまま90日放置すると**データを失うリスク**がある
- 有料プランへのアップグレードは月$25〜と個人には高い

だからこそ、**自動で定期 ping を送る仕組み**を作っておくのが有効です。

---

## 解決策：GitHub Actions で毎日 curl を叩く

今回試した構成がこちら: [`biki-cloud/supabase-dont-stop`](https://github.com/biki-cloud/supabase-dont-stop)

### 構成のシンプルさが魅力

```
supabase-dont-stop/
├── README.md
└── .github/
    └── workflows/
        └── keep_alive.yml
```

**たったこれだけ。** Node.js も Python も不要です。

### ワークフローの全コード

```yaml
name: Supabase Keep Alive

on:
  schedule:
    # 毎日 0:00 (UTC) に実行（日本時間 9:00）
    - cron: '0 0 * * *'
  workflow_dispatch: # 手動実行ボタン用

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Send API Request
        run: |
          curl -i -X GET "${{ secrets.SUPABASE_URL }}/rest/v1/" \
          -H "apikey: ${{ secrets.SUPABASE_ANON_KEY }}" \
          -H "Authorization: Bearer ${{ secrets.SUPABASE_ANON_KEY }}"
```

### なぜこれで動くのか

Supabase の REST API エンドポイント（`/rest/v1/`）に GET リクエストを送るだけで、Supabase 側はアクティビティとして記録します。データの読み書きは不要で、**疎通確認するだけでリセットされる**のです。

:::message
`-i` オプションはレスポンスヘッダーも出力します。GitHub Actions のログで HTTP ステータスコードが確認できて便利です。
:::

---

## セットアップ手順

### ステップ1: リポジトリを作成してワークフローを追加

既存のリポジトリでも、新しく作ったリポジトリでも構いません。`.github/workflows/keep_alive.yml` を上記のコードで作成します。

フォークして使う場合は [`biki-cloud/supabase-dont-stop`](https://github.com/biki-cloud/supabase-dont-stop) をフォークするだけです。

### ステップ2: Supabase の認証情報を取得

Supabase ダッシュボードを開き、**Settings → API** へ移動します。

以下の2つをコピーしておきます：

- **Project URL** → `SUPABASE_URL` に使用
- **anon public key** → `SUPABASE_ANON_KEY` に使用

:::message alert
anon key は公開しても問題ありませんが（Row Level Security が前提）、念のため GitHub Secrets に入れておくのがベストプラクティスです。
:::

### ステップ3: GitHub Secrets に登録

リポジトリの **Settings → Secrets and variables → Actions → New repository secret** から2つ登録します。

```
SUPABASE_URL      = https://xxxxxxxxxxxx.supabase.co
SUPABASE_ANON_KEY = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### ステップ4: 動作確認（手動実行）

cron は UTC の都合上すぐには実行されません。**Actions タブ → 「Supabase Keep Alive」→ Run workflow** から手動実行して動作確認しましょう。

成功すると以下のようなログが出力されます：

```
HTTP/2 200
...
{"paths":{...}}
```

HTTP 200 が返れば完了です🎉

---

## 見落としがちな落とし穴

### GitHub Actions 自体も60日で止まる

実はここが重要なポイントで、**他の記事ではあまり触れられていません。**

GitHub は **リポジトリに60日間コミットがないと、cron スケジュールのワークフローを自動で無効化します**。

```
つまり：
Supabase を守るために → GitHub Actions で ping
GitHub Actions を守るために → リポジトリを定期的に更新
```

という構図になります。

:::message
対策として以下のいずれかを検討してください：
- 別のアクティブなリポジトリから `workflow_dispatch` で呼び出す
- 60日ごとにリポジトリに何かコミットする習慣をつける
- GitHub の「Actions を有効にする」通知メールに気づいたら手動で再有効化する
:::

### cron の時刻は UTC 基準

`cron: '0 0 * * *'` は UTC の 0:00 = **日本時間の朝9:00** に実行されます。

日本時間で時刻を指定したい場合は9時間引いて設定します。

```yaml
# 日本時間 0:00（UTC 15:00 = 前日）
- cron: '0 15 * * *'
```

---

## 代替アプローチとの比較

| 手法 | 特徴 | 難易度 |
|------|------|--------|
| **curl（本構成）** | 最小構成、依存なし | ⭐ |
| **Supabase JS クライアント** | select() でクエリ実行、より確実 | ⭐⭐ |
| **Python スクリプト** | 複数プロジェクト対応 | ⭐⭐⭐ |
| **KeepSupabaseAlive** | ブラウザでワークフロー自動生成（外部サービス） | ⭐ |

今回の `curl` 構成は**最も手数が少なく**、GitHub Actions と Supabase 以外に何も必要ありません。個人開発のサイドプロジェクトには十分な選択肢です。

---

## まとめ

- Supabase 無料プランは **7日間無操作で自動停止**、**90日で復元不可**になる
- GitHub Actions の cron スケジュール + `curl` で **依存ゼロの最小 keep-alive** が実現できる
- **GitHub Actions 自体も60日でcronが止まる**ことに注意（見落としがち！）
- セットアップは Secrets を2つ登録するだけで完了

個人開発で Supabase を使っている方は、ぜひ試してみてください。

---

**参考リンク**

- [biki-cloud/supabase-dont-stop](https://github.com/biki-cloud/supabase-dont-stop) — 本記事で紹介したリポジトリ
- [Supabase Docs: Pausing Projects](https://supabase.com/docs/guides/troubleshooting/pausing-pro-projects)
- [Paused Free Plan projects are restorable for 90 days](https://github.com/orgs/supabase/discussions/27497) — 公式ディスカッション
