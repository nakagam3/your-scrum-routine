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

## 検証の二層

このスキルは2種類の検証を Phase を分けて行う。役割重複を避けるための区分。

| 層 | タイミング | 対象 | 実装 |
|---|---|---|---|
| **YAML 自己評価** | Phase 3.5（テキスト生成前） | sprint-log の妥当性（done_condition・接続・rationale） | SKILL.md 内で直接検証 |
| **テキスト Lint** | Phase 5（出力直前） | 日報テキストの可読性（読者向け語彙・表現） | document-reviewer (Agent 経由) |

同じ問題を2回見ない。YAML の問題は YAML 段階で潰し、テキストの問題はテキスト段階で潰す。

---

## ワークフロー

```
/retrospective
│
├─ セットアップ未完了 → references/setup.md を参照して実行
│   （今Q目標.yml と マイルストーン.yml の両方が存在しなければ未完了）
│
└─ 通常 → 振り返りモード（当日ファイルが既に存在していても再実行する）
    │
    ├─ Phase 1: データ収集
    ├─ Phase 2: 振り返り
    ├─ Phase 3: 明日のゴール提案
    ├─ Phase 3.5: YAML 自己評価
    └─ Phase 5: 出力
```

セットアップモードの詳細手順は [references/setup.md](references/setup.md) を参照。
判定: `今Q目標.yml` と `マイルストーン.yml` の両方の存在で決まる（ディレクトリではなくファイル単位）。片方だけ存在する場合は欠けているステップから再開する。

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

#### 3-0. MS 鮮度チェック（入力バリデーション）

マイルストーン.yml の今週の weekly MS が未設定、または先週以前のままの場合、ユーザーに警告する。
AskUserQuestion で「マイルストーン.yml を更新してから再実行する」か「現状のまま進める」かを選ばせる。
ゴール提案の品質は MS の鮮度に依存するため、ここで検出する。

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

### Phase 3.5: YAML 自己評価

テキスト生成に進む前に、生成した sprint-log YAML の妥当性を検証する。
問題があれば Phase 3 の対話に戻って修正してから Phase 5 に進む。

- **done_condition の具体性**: 「作業継続」「進める」「対応する」等の曖昧な表現になっていないか。該当する場合は具体化する
- **放置マイルストーン**: 今週の weekly マイルストーンで、明日のゴールを含めても一度も日次ゴールが紐づかないものがないか。ある場合は Phase 3 の対話に戻りユーザーに伝える
- **rationale の有無**: 各ゴールに「なぜ明日効くか」の理由が付いているか。ない場合、提案として弱いので追記する
- **parent の接続**: 孤児ゴール（parent なし）が含まれていないか。含まれている場合は警告のみ（ブロックはしない。雑多な作業も記録する）

この層は YAML の品質保証に特化する。テキスト表現の問題（語彙・冗長性等）は Phase 5 Lint で扱う。

### Phase 5: 出力

1. **YAML保存**:
   - 前営業日の sprint-log/ ファイルを更新（Phase 2 の reconciliation + activity_log）
   - 明日の sprint-log/ ファイルを新規作成（Phase 3 の goals）

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

3. **テキスト生成（ドラフト）**:
   - [references/output-format.md](references/output-format.md) のフォーマット定義に従いドラフトを生成する
   - 今Q目標.yml の `goals[].status` を参照し、North Star 行を組み立てる（confirmed 以外は「設定中」）
   - この段階では可読性の最終調整（codename 置換・週番号相対化・時刻削除等）は気にしなくてよい。次ステップの Lint が検証・修正する

4. **Lint（document-reviewer による検証）**:
   - ドラフトを **Agent ツール経由** で document-reviewer に渡し可読性を検証する（context isolation 目的）
     - subagent_type: `"general-purpose"`
     - description: `"retrospective: 日報Lint"`
     - プロンプトには以下を同梱する:
       - ドラフトテキスト本体
       - マイルストーン.yml の `project_aliases` セクション（codename → 正式名称の authoritative mapping）
       - 基準日（今日の日付）と ISO 週番号（W{N} 相対化のため）
       - [references/output-format.md](references/output-format.md) の「変換ルール」セクション
     - 指示: 「読者プロファイル **部署全員（PD室）** で検証し、以下を修正して返す」
       - codename・内部IDの置換（`project_aliases` を正として適用。辞書にない codename は文脈から推定せず原文維持して指摘）
       - `W{N}` を「先週/今週/来週」に（基準日の ISO 週と比較）
       - 時刻の付帯情報削除（summary/done_condition から `HH:MM`, `HH:MM-HH:MM` 等）
       - self-evident な補足の削除
       - summary と done_condition の同義反復回避
       - 個人内省フレーズの削除
     - 修正不要なら原文を返す
   - **修正は最大1回**（無限ループ防止）。結果が最終出力となる
   - document-reviewer が利用できない場合（development-deck 未インストール等）、Lint をスキップしドラフトをそのまま最終出力にする（ブロックしない）

5. **Slack配信**:
   - 最終出力をチャットに出力し、同内容を Slack 自分宛 DM に送信する（`slack_send_message` で channel_id に config.yml の slack_user_id を指定）
   - **メッセージ全体をコードブロック（triple backtick）で囲む**。Slack のマークダウン解釈を回避し、整形済みテキストとして表示するため
   - コードブロック内の注意: `~` は範囲表現で Slack がストライクスルーとして解釈するため `-` に置換する
   - 投稿失敗時はエラーを表示し、チャット出力のみで続行する（ブロックしない）

6. **深掘りの提案**:
   - 日報出力の後、AskUserQuestion で「より深くふりかえる?」と聞く
   - Yes → Skill ツールで `development-deck:development-partner` にフォワーディングする。その際、以下のコンテキストを渡す:
     - 今日の予実照合結果
     - 気になるパターン（繰越が続いているゴール、想定外の作業が多い等）
   - development-deck が利用できない場合（スキルの呼び出しに失敗した場合）、「development-deck をインストールするとより深い振り返りができます」と表示してスキルを終了する
   - No → スキル終了

---

## ハードルール

1. **done_condition に「作業継続」を許可しない**: 各ゴールには具体的な完了条件を設定する。「XXXの作業を進める」ではなく「XXXの〇〇まで到達する」「XXXについて△△と合意する」のように検証可能な条件にする。検証可能性がないと Phase 2 の達成判定が主観的になり、PDCA が回らない
2. **先に提案する**: Phase 3 ではユーザーに「何やる?」と聞く前に、必ず叩き台を提示する。ユーザーの思考を回すための叩き台が価値の源泉
3. **前営業日ファイルの存在を前提としない**: 初回実行時や休暇明けなど、前営業日のファイルがない場合は Phase 2 をスキップし、Phase 3 から開始する。前回の日報から2日以上空いている場合は「前回は{date}です（{N}日前）。その間の振り返りは省略します」と表示する
4. **Markdown層は読み取り専用**: product-vision/ 配下の .md ファイルをスキルが書き換えない。更新が必要な場合はユーザーに手動更新を促す。Markdown 層はユーザーの人生観・キャリア観を含むためデータ所有権を侵さない
5. **データ収集の失敗はブロックしない**: MCP 経由のデータ取得が失敗した場合（認証切れ等）、該当ソースをスキップして続行する。エラーメッセージを表示し、手動入力を促す。日報は毎日出るほうが価値があるため、完璧を待たずに部分出力する
