# セットアップモード

`~/.claude/your-scrum-routine/product-vision/` 配下に `今Q目標.yml` と `マイルストーン.yml` の**両方**が存在しない場合に発動する。
片方だけ存在する場合は、欠けているファイルのステップから再開する。

初回セットアップ専用のため SKILL.md 本体から切り出してある。通常運用では参照されない。

---

## 手順

1. ディレクトリ `~/.claude/your-scrum-routine/product-vision/` と `~/.claude/your-scrum-routine/sprint-log/` を作成する
2. 設定ファイル `~/.claude/your-scrum-routine/config.yml` を作成する（後述）
3. 以下の順に対話で各ファイルを生成する

### Step 0: ユーザー設定

Slack のユーザーIDを取得する。方法:

1. `slack_search_public_and_private` で適当な検索（例: `from:me`）を実行し、レスポンスに含まれるユーザーIDを取得する
2. 取得できない場合は AskUserQuestion でユーザーに直接聞く

取得したIDを `~/.claude/your-scrum-routine/config.yml` に保存する:

```yaml
# 例: 実際の値はユーザーごとに異なる
slack_user_id: "UXXXXXXXXXX"
github_username: "your-github-username"
timezone: "Asia/Tokyo"
```

### Step 1: North Star（Markdown層）

AskUserQuestion で以下を聞く:

- 「輝かしい未来（3年後のキャリアイメージ）を教えてください」
- 「目指す姿（今期ストレッチ、半年後の姿）を教えてください」

回答をもとに以下のファイルを生成する:

- `product-vision/輝かしい未来.md`
- `product-vision/目指す姿.md`

他の Markdown 層（中期経営計画、PD室年間テーマ）は任意。ユーザーが言及した場合のみ作成する。

### Step 2: 今Q目標

`product-vision/今Q目標.yml` が既に存在する場合はスキップする。

AskUserQuestion で聞く:

- 「今Qの目標を教えてください（複数可）。それぞれ、目指す姿のどこに繋がるかも教えてください」

回答をもとに `product-vision/今Q目標.yml` を生成する。
north_star_link はユーザーの言葉をそのまま使う（厳密なID参照ではない）。

### Step 3: マイルストーン

`product-vision/マイルストーン.yml` が既に存在する場合はスキップする。

AskUserQuestion で聞く:

- 「今週のマイルストーン（達成条件）を教えてください。スプレッドシートからコピペでもOKです」
- 「今月のマイルストーン（月末時点の到達状態）があれば教えてください」

回答をもとに `product-vision/マイルストーン.yml` を生成する。
各マイルストーンの parent は、今Q目標のどれに繋がるかを対話で確認する。

project_aliases セクションは、ユーザーが略語・内部IDを使っている場合に提案する（空でも可。日報 Lint で必要になった段階で追加すればよい）。

### Step 4: 確認

生成した全ファイルの概要を表示し、AskUserQuestion で最終確認する。
確認が取れたらセットアップ完了。そのまま振り返りモードの Phase 3（明日のGoal提案）に移行する。

---

## セットアップ判定の注意

`今Q目標.yml` と `マイルストーン.yml` の両方の存在で判定する（product-vision/ ディレクトリの有無ではない）。
片方だけ存在する場合は欠けているステップから再開する。これにより部分的に作り直したいケースに対応できる。
