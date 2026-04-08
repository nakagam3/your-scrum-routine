---
name: retrospective
description: >
  思考のペースメーカー。目標に向かって毎日一番効くアクションを提案し、
  選ぶ・捨てるの判断を支援する。「/retrospective」で明示的に呼び出す。
allowed-tools: Read, Write, Glob, Bash, Agent, AskUserQuestion, mcp__claude_ai_Google_Calendar__gcal_list_events, mcp__claude_ai_Google_Calendar__gcal_get_event, mcp__claude_ai_Google_Calendar__gcal_create_event, mcp__claude_ai_Slack__slack_send_message
---

# Retrospective

**思考のペースメーカー。**

目標に向かって毎日一番効くアクションを提案し、叩き台を見せることで思考を回す。
ユーザーが悩む時間を最小化し、「ぺっぺっと選んで明日に備える」体験を提供する。

正確な予実管理が目的ではない。提案の質が生命線。

---

## 設計思想

```
目的: 毎日、目標に向かって一番効く行動を選び続けること
手段: データ収集 → 状況把握 → ゴール提案 → ユーザーが選ぶ
価値: 叩き台があることで思考が回る、輪郭がはっきりする、言語化が進む
```

予実照合・接続検証は「良い提案を作るための内部処理」であり、ユーザーに見せる主役ではない。

---

## データ構造

ストレージ構成・YAMLスキーマ・各フィールド定義は [references/schema.md](references/schema.md) を参照。
出力フォーマットは [references/output-format.md](references/output-format.md) を参照。

---

## ワークフロー

```
/retrospective
│
├─ セットアップ未完了 → セットアップモード
│   （今Q目標.yml と マイルストーン.yml の両方が存在しなければ未完了）
│
└─ 通常 → 振り返りモード（当日ファイルが既に存在していても再実行する）
    │
    ├─ Phase 1: データ収集（今日何が起きたか）
    ├─ Phase 2: 振り返り（軽く。提案の材料にする）
    ├─ Phase 3: 明日のゴール提案（ここが主役）
    ├─ Phase 4: 周期境界チェック（日付で自動判定）
    └─ Phase 5: 出力
```

---

## セットアップモード

`~/.claude/your-scrum-routine/product-vision/` 配下に `今Q目標.yml` と `マイルストーン.yml` の**両方**が存在しない場合に発動する。
片方だけ存在する場合は、欠けているファイルのステップから再開する。

### 手順

1. ディレクトリ `~/.claude/your-scrum-routine/product-vision/` と `~/.claude/your-scrum-routine/sprint-log/` を作成する
2. 設定ファイル `~/.claude/your-scrum-routine/config.yml` を作成する（後述）
3. 以下の順に対話で各ファイルを生成する:

**Step 0: ユーザー設定**

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

**Step 1: North Star（Markdown層）**

AskUserQuestion で以下を聞く:
- 「輝かしい未来（3年後のキャリアイメージ）を教えてください」
- 「目指す姿（今期ストレッチ、半年後の姿）を教えてください」

回答をもとに以下のファイルを生成する:
- `product-vision/輝かしい未来.md`
- `product-vision/目指す姿.md`

他の Markdown 層（中期経営計画、PD室年間テーマ）は任意。ユーザーが言及した場合のみ作成する。

**Step 2: 今Q目標**

`product-vision/今Q目標.yml` が既に存在する場合はスキップする。

AskUserQuestion で聞く:
- 「今Qの目標を教えてください（複数可）。それぞれ、目指す姿のどこに繋がるかも教えてください」

回答をもとに `product-vision/今Q目標.yml` を生成する。
north_star_link はユーザーの言葉をそのまま使う（厳密なID参照ではない）。

**Step 3: マイルストーン**

`product-vision/マイルストーン.yml` が既に存在する場合はスキップする。

AskUserQuestion で聞く:
- 「今週のマイルストーン（達成条件）を教えてください。スプレッドシートからコピペでもOKです」
- 「今月のマイルストーン（月末時点の到達状態）があれば教えてください」

回答をもとに `product-vision/マイルストーン.yml` を生成する。
各マイルストーンの parent は、今Q目標のどれに繋がるかを対話で確認する。

**Step 4: 確認**

生成した全ファイルの概要を表示し、AskUserQuestion で最終確認する。
確認が取れたらセットアップ完了。そのまま振り返りモードの Phase 3（明日のGoal提案）に移行する。

---

## 振り返りモード

### Phase 1: データ収集

今日の実績データと、明日のゴール提案に必要な材料を集める。
**重いデータソース（Slack、セッションサマリ、議事録）は subagent で並列収集し、メインコンテキストには構造化サマリのみ返す。**

subagent のプロンプトテンプレートと出力スキーマは [references/subagent-prompts.md](references/subagent-prompts.md) を参照。

#### Step 0: 前準備（メインで実行）

- config.yml を読み込む（slack_user_id, github_username, timezone）
- 前営業日の sprint-log/ ファイルを Glob で特定し、goals を抽出する
- product-vision/マイルストーン.yml から今週の weekly MS を抽出する
- product-vision/今Q目標.yml から calendar_color_id マッピングを取得する
- 翌営業日の日付を算出する

#### Step 1: 軽量データ取得（メインで実行、並列可）

以下を取得する。いずれもレスポンスが軽いためメインコンテキストに直接載せる。

- **Google Calendar（今日）**: `gcal_list_events` で today 00:00〜23:59 を取得。timeZone は config.yml の timezone を使用
- **Google Calendar（明日）**: `gcal_list_events` で翌営業日の予定を取得。MTG・作業ブロック・凡事（〆切付き通知）を分類し、作業可能時間を算出
- **GitHub**: Bash で `gh api graphql` を実行し contributionsCollection を取得。config.yml の github_username を使用

#### Step 2: MTG イベント特定（メインで実行）

Step 1 の Calendar（今日）結果から、議事録取得対象の MTG を特定する:
- `mtg//` プレフィックス or 参加者2名以上
- 前営業日の goals に対応する MTG + 明日の MTG に関連する今日の MTG を優先
- 各イベントの `{ event_id, summary, start_time, calendar_id }` をリスト化

#### Step 3: 重量データ収集（subagent で並列実行）

Agent tool で 3本の subagent を **1メッセージで同時に起動** する。

各 subagent の共通設定:
- subagent_type: `"general-purpose"`
- model: `"sonnet"`
- prompt: [references/subagent-prompts.md](references/subagent-prompts.md) のテンプレートに以下を注入

| subagent | description | 注入するパラメータ |
|----------|-------------|-------------------|
| A: Slack | `"retrospective: Slack収集"` | slack_user_id, today, previous_goals_yaml, weekly_milestones_yaml |
| B: セッションサマリ | `"retrospective: セッション収集"` | sessions_dir (`~/.claude/your-scrum-routine/increments/{today}`), previous_goals_yaml |
| C: 議事録 | `"retrospective: 議事録収集"` | mtg_events_json (Step 2の結果), previous_goals_yaml, user_display_name |

**3本を1つのメッセージ内で同時に起動する。順次起動は禁止。**

#### Step 4: 結果統合（メインで実行）

3本の subagent から受信した構造化サマリを統合する:

| フィールド | 統合先 |
|-----------|--------|
| `activity_log_entries` | Phase 5 の activity_log に集約（Calendar, GitHub の結果と合わせる） |
| `goal_evidence` | Phase 2 の達成判定材料。3ソースの evidence を goal_id ごとにマージ |
| `next_actions` / `next_steps` | Phase 3 のゴール候補に使用 |

**エラーハンドリング**: subagent の出力に `error` フィールドがある場合、該当ソースのエラーをユーザーに通知し、手動入力を促す。他のソースの処理はブロックしない。

**精度フォールバック**: Phase 2 でユーザーが達成判定に異議を唱えた場合、メインから直接 MCP ツールを使って原典を再取得できる（`gcal_get_event` は allowed-tools に残してある）。

取得結果は Phase 5 で activity_log として YAML に保存する。

### Phase 2: 振り返り（軽く）

前営業日の goals の結果を確認する。**提案の材料を作ることが目的。厳密な判定は目的ではない。**

1. 前営業日の sprint-log/ ファイルを Glob で特定（最新ファイル）
2. 前営業日のファイルがない場合はスキップ（「前回は{date}です（{N}日前）。その間の振り返りは省略します」と表示）
3. 各ゴールの結果を判定（Phase 1 Step 4 で統合した goal_evidence を使用）:
   - **evidence あり（judgment: achieved/partial）**: subagent が抽出した証拠を提示
   - **PR/commit あり**: GitHub データと突合
   - **evidence なし（judgment: no_evidence）**: ユーザーに聞く
   - ユーザーが判定に異議を唱えた場合、メインから直接 MCP ツールで原典を再取得して確認する
4. 結果をまとめて提示し、一括確認:
   - 「こう判定した。合ってる? 違うのがあれば番号で教えて」
   - 個別に1つずつ聞かない
5. partial/missed のゴールは Phase 3 の繰越候補にする
6. 計画外の作業を簡潔に確認
7. 前営業日のファイルを更新（reconciliation セクション）

### Phase 3: 明日のGoal提案（ここが主役）

Phase 1-2 の全データを材料に、**明日のゴールを提案する**。ユーザーに聞いてから考えるのではなく、先に叩き台を出す。

#### 3-1. ゴール候補の収集

以下のソースからゴール候補を集める:

| ソース | 候補の出し方 |
|--------|-------------|
| 議事録の Next Steps | [中上裕基] アサインのアクションをそのまま候補に |
| 明日のカレンダー MTG | MTGの colorId を今Q目標.yml の calendar_color_id と照合 → マッチした q_goal_id の weekly MS を逆引きして parent 候補に。マッチしない場合は従来通り手動設定。MTGの目的・前回の議事録から done_condition を推定 |
| 明日のカレンダー 凡事 | 〆切付き通知 → そのまま候補に |
| 接近中の凡事マイルストーン | マイルストーン.yml で done_condition 付き・deadline が7日以内の凡事MSがあれば、毎日15-30分の作業ブロックをゴール候補に追加する。done_condition は「{凡事MS名}に向けて{具体的な次の一歩}を完了する」のように日次で検証可能にする |
| 繰越（partial/missed） | Phase 2 の結果から |
| 放置マイルストーン | 今週の weekly MS で、まだ日次ゴールが紐づかないもの |
| セッションサマリの Next Actions | session-closer の「未解決・Next Actions」から |

#### 3-2. 内部分類（Must / Win）

候補を内部的に Must / Win に分類する。**この分類は出力フォーマットには出さない。** 提案の優先順位付けに使う。

**Must（落としたら負け）**:
- 他者とのMTGで成果物・準備が必要なもの
- 議事録 Next Steps で期限が今週内のアクション
- 凡事（〆切付き）
- 前日からの繰越で2回目以上のもの

**Win（できたら勝ち）**:
- 放置マイルストーンに紐づく最小アクション
- 議事録 Next Steps で期限が明示されてないが戦略的に重要なもの
- 今Q目標に直結するが今週のマイルストーンにまだ落ちてないもの
- 第2領域（緊急じゃないが重要）に該当するもの

#### 3-3. ゴール提案の組み立て

1. Must を先に配置し、残りの作業可能時間で Win を配置
2. 各ゴールに done_condition をつける。**「作業継続」は許可しない**
3. 各ゴールに parent（週次マイルストーン）を紐づける
4. 作業可能時間に対してゴール数が多すぎないか検証（目安: MTG除いた作業時間1hにつき1ゴール）
5. **提案理由を1行つける**（なぜこれが明日のゴールとして効くか）

#### 3-4. ユーザーとの対話

提案を提示した上で、AskUserQuestion で確認:
- 「これで行く? 入れ替えたいもの・追加したいものがあれば教えて」
- ユーザーが修正 → 反映
- done_condition が曖昧な場合は具体化を促す
- parent が設定できないゴールは孤児として軽く警告（ブロックはしない）

**対話のトーン**: 提案は叩き台。ユーザーが「いや、こっち」と言いやすい出し方をする。

#### 3-5. YAML生成

確定したゴールで明日の sprint-log/YYYY-MM-DD.yml を生成する。

### Phase 4: 周期境界チェック

実行日の日付を見て、以下の条件に該当する場合に追加フェーズを挿入する。

| 条件 | 追加フェーズ | 内容 |
|---|---|---|
| 月曜 | 週次マイルストーン更新 | 先週の振り返り + 今週のマイルストーン設計 |
| 月末5営業日前〜月末 | 月次マイルストーン照合 | 月次マイルストーンの達成度照合 + 来月設計 |
| Q初日（1月/4月/7月/10月の最初の営業日） | 四半期照合 | 前Qの今Q目標振り返り + 今Q目標更新 + 目指す姿との接続確認 |

**月曜の処理**:
1. マイルストーン.yml の先週分 weekly マイルストーンの status を確認・更新
2. 今週の weekly マイルストーンを AskUserQuestion で設計する
3. 各マイルストーンの parent（月次 or 今Q目標）を確認する
4. changelog に記録する

**月末付近の処理**:
1. マイルストーン.yml の当月 monthly マイルストーンの達成度を照合
2. 来月の monthly マイルストーンを AskUserQuestion で設計する

**Q初日の処理**:
1. 前Qの今Q目標.yml の各ゴールの達成度を照合（roof / moon それぞれ判定）
2. 今Qの今Q目標.yml を AskUserQuestion で作成 or 更新
3. 目指す姿.md を読み込み、今Q目標.yml の context.vision との接続を確認
4. 必要なら目指す姿.md の更新を促す（ただしスキルは読み取り専用。ユーザーが手動更新）

### Phase 5: 出力

1. **YAML保存**:
   - 前営業日の sprint-log/ ファイルを更新（Phase 2 の reconciliation + activity_log）
   - 明日の sprint-log/ ファイルを新規作成（Phase 3 の goals）
   - マイルストーン.yml を更新（Phase 4 で変更があった場合）

2. **カレンダー登録**:
   - Phase 3 で確定したゴールのうち、既存の MTG でカバーされないものを Google Calendar に作業ブロックとして登録する
   - 既存 MTG と重複するゴール（例: MTG 参加が done_condition のゴール）は登録しない
   - 明日のカレンダーの空き時間に配置する。配置ルール:
     - Must を先に、作業可能な最も早い空きブロックに配置
     - Win は残りの空き時間に配置（No MTG Time 等の長いブロックを優先）
   - イベント設定:
     - summary: 通常ゴールは `work//dev/{プロジェクト名}/{ゴールsummary}`、凡事マイルストーン由来は `work//other//{ゴールsummary}`
     - colorId: 今Q目標.yml の calendar_color_id を参照（parent の週次MS → 月次MS → 今Q目標の順に辿る）
     - sendUpdates: `none`
   - **凡事マイルストーン接近時（deadline 7日以内）**: 他のゴール用ブロックとは別に、最低15分（理想30分）の作業ブロックを必ず確保する。MTG過密日でも隙間に押し込む
   - ツール: `mcp__claude_ai_Google_Calendar__gcal_create_event`

3. **テキスト生成 + Slack配信**:
   - [references/output-format.md](references/output-format.md) に定義されたフォーマットに従い、テキストを生成してチャットに出力する
   - 周期境界チェックの追加出力がある場合はフォーマットに従い挿入する
   - 生成したテキストを Slack 自分宛 DM に送信する（`slack_send_message` で channel_id に config.yml の slack_user_id を指定）
   - **メッセージ全体をコードブロック（triple backtick）で囲む**。Slack のマークダウン解釈を回避し、整形済みテキストとして表示するため
   - コードブロック内の注意: `~` は範囲表現で Slack がストライクスルーとして解釈するため `-` に置換する
   - 投稿失敗時はエラーを表示し、チャット出力のみで続行する（ブロックしない）

4. **深掘りの提案**:
   - 日報出力の後、AskUserQuestion で「より深くふりかえる?」と聞く
   - Yes → Skill ツールで `development-deck:development-partner` にフォワーディングする。その際、以下のコンテキストを渡す:
     - 今日の予実照合結果
     - 気になるパターン（繰越が続いているゴール、想定外の作業が多い等）
   - development-deck が利用できない場合（スキルの呼び出しに失敗した場合）、「development-deck をインストールするとより深い振り返りができます」と表示してスキルを終了する
   - No → スキル終了

---

## ハードルール

1. **done_condition に「作業継続」を許可しない**: 各ゴールには具体的な完了条件を設定する。「XXXの作業を進める」ではなく「XXXの〇〇まで到達する」「XXXについて△△と合意する」のように検証可能な条件にする
2. **先に提案する**: Phase 3 ではユーザーに「何やる?」と聞く前に、必ず叩き台を提示する。ユーザーの思考を回すための叩き台が価値の源泉
3. **前営業日ファイルの存在を前提としない**: 初回実行時や休暇明けなど、前営業日のファイルがない場合は Phase 2 をスキップし、Phase 3 から開始する。前回の日報から2日以上空いている場合は「前回は{date}です（{N}日前）。その間の振り返りは省略します」と表示する
4. **Markdown層は読み取り専用**: product-vision/ 配下の .md ファイルをスキルが書き換えない。更新が必要な場合はユーザーに手動更新を促す
5. **データ収集の失敗はブロックしない**: MCP 経由のデータ取得が失敗した場合（認証切れ等）、該当ソースをスキップして続行する。エラーメッセージを表示し、手動入力を促す
6. **セットアップは個別ファイル単位で判定する**: product-vision/ ディレクトリの有無ではなく、`今Q目標.yml` と `マイルストーン.yml` の両方が存在するかで日報モード / セットアップモードを判定する。片方だけ存在する場合は欠けているステップから再開する

---

## 自己評価（出力の前に実行）

テキストを生成する前に、以下を検証する。問題があれば修正してから出力する。

### チェック1: 提案の具体性

明日のゴールの done_condition が「作業継続」「進める」等の曖昧な表現になっていないか確認。該当する場合は具体化を促す。

### チェック2: 放置マイルストーン

今週の weekly マイルストーンで、明日のゴールを含めても一度も日次ゴールが紐づかないものがないか確認。ある場合は Phase 3 の対話中にユーザーに伝える。

### チェック3: 提案理由の有無

各ゴールに「なぜこれが明日のゴールとして効くか」の理由が紐づいているか確認。理由がないゴールは提案として弱い。
