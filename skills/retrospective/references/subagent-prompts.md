# Subagent プロンプトテンプレート

Phase 1 Step 3 で起動する3本の subagent のプロンプト定義。
`{placeholder}` はメイン側が Step 0-2 の結果から実行時に注入する。

各 subagent の設定:
- subagent_type: `"general-purpose"`
- model: `"sonnet"`

---

## Subagent A: Slack 収集+抽出

description: `"retrospective: Slack収集"`

```
あなたは retrospective スキルの Slack データ収集を担当する。
ユーザーの今日の Slack 活動から、振り返りとゴール提案に必要な情報を抽出する。

## 入力

### 設定
- Slack ユーザーID: {slack_user_id}
- 対象日: {today}

### 前営業日のゴール（done_condition との突合用）
{previous_goals_yaml}

### 今週の週次マイルストーン
{weekly_milestones_yaml}

## 作業手順

1. `slack_search_public_and_private` で `from:<@{slack_user_id}> on:{today}` を検索
2. ページネーションで全件取得
3. 以下の3層に情報を整理する

## 出力形式（YAML、厳守）

slack_summary:
  activity_log_entries:
    - "#{チャンネル名}: {トピック概要}({投稿数}投稿)"
    # チャンネルごとに1行。主要トピックを含める

  goal_evidence:
    - goal_id: "{前営業日ゴールID}"
      evidence: "{達成を示す具体的な証拠。1-2文}"
      judgment: achieved | partial | no_evidence
    # 前営業日の各ゴールについて1エントリ。判定ソースがなければ no_evidence

  next_actions:
    - source: "#{チャンネル名}"
      action: "{明示的なアクション。依頼・約束・TODO}"
      urgency: "{今日中 | 今週中 | 期限なし}"
    # 明示的なアクションのみ。暗黙の TODO は含めない

## ルール
- 出力は YAML のみ。前置き・補足説明は不要
- goal_evidence は前営業日のゴールがない場合は空配列
- activity_log_entries はチャンネル単位で集約。個別メッセージは載せない
- next_actions は他者からの依頼、自分が宣言した TODO、期限付きアクションのみ

## エラー時の振る舞い
- MCP ツールの呼び出しが失敗した場合、取得できたデータのみで出力を構成する
- 全データ取得に失敗した場合は以下を返す:
  slack_summary:
    error: "{エラーの概要}"
    activity_log_entries: []
    goal_evidence: []
    next_actions: []
```

---

## Subagent B: セッションサマリ収集+抽出

description: `"retrospective: セッション収集"`

```
あなたは retrospective スキルのセッションサマリ収集を担当する。
今日の Claude Code セッションサマリから、振り返りとゴール提案に必要な情報を抽出する。

## 入力

### 対象ディレクトリ
{sessions_dir}

### 前営業日のゴール（done_condition との突合用）
{previous_goals_yaml}

## 作業手順

1. Glob で `{sessions_dir}/*.md` を取得
2. 各ファイルの「成果・到達点」「未解決・Next Actions」セクションを読む
3. 以下の3層に情報を整理する

## 出力形式（YAML、厳守）

session_summary:
  activity_log_entries:
    - "{プロジェクト名}: {主要成果の1行要約}"
    # セッションごとに1行

  goal_evidence:
    - goal_id: "{前営業日ゴールID}"
      evidence: "{達成を示す具体的な証拠。1-2文}"
      judgment: achieved | partial | no_evidence
    # 前営業日の各ゴールについて1エントリ

  next_actions:
    - source: "session/{session_id}"
      action: "{未解決・Next Actions から抽出}"
      priority: "{高 | 中 | 低}"
    # [高] [中] [低] のプレフィックスがあればそれに従う

## ルール
- 出力は YAML のみ。前置き・補足説明は不要
- 各セッションの「意思決定」セクションは goal_evidence の判断材料として参照するが、出力には含めない
- 「知見・地雷」セクションは出力対象外

## エラー時の振る舞い
- セッションファイルが存在しない場合は全フィールド空配列で返す
- ファイル読み込みに失敗した場合、読めたファイルのみで出力を構成する
```

---

## Subagent C: 議事録 収集+抽出

description: `"retrospective: 議事録収集"`

```
あなたは retrospective スキルの議事録収集を担当する。
今日の MTG の Gemini 議事録から、振り返りとゴール提案に必要な情報を抽出する。

## 入力

### 対象 MTG イベント
{mtg_events_json}
# 各イベント: { event_id, summary, start_time, calendar_id }

### 前営業日のゴール（done_condition との突合用）
{previous_goals_yaml}

### ユーザー名（Next Steps のアサイン抽出用）
{user_display_name}

## 作業手順

1. 各 MTG イベントの詳細を `gcal_get_event` で取得
2. attachments から「Gemini によるメモ」（mimeType: application/vnd.google-apps.document）の documentId を抽出
3. Bash で `gws docs documents get --params '{"documentId": "{docId}"}'` を実行
4. jq で `[.. | .textRun? // empty | .content] | join("")` でテキスト抽出
5. 議事録テキストから以下の3層に情報を整理する

議事録が存在しないイベント（Gemini メモが添付されていない）はスキップする。

## 出力形式（YAML、厳守）

minutes_summary:
  activity_log_entries:
    - "{開始時刻} {MTG名}: {概要の1行要約}"
    # MTGごとに1行

  goal_evidence:
    - goal_id: "{前営業日ゴールID}"
      evidence: "{達成を示す具体的な証拠。議事録の概要・詳細セクションから。1-2文}"
      judgment: achieved | partial | no_evidence
    # 前営業日の各ゴールについて、関連する MTG の議事録があれば判定

  next_steps:
    - mtg: "{MTG名}"
      assignee: "{中上裕基 | チーム | その他}"
      action: "{次のステップの内容}"
      deadline: "{期限。明記されていなければ null}"
    # 「次のステップ」セクションから [{user_display_name}] or [チーム] のアクションを抽出

## ルール
- 出力は YAML のみ。前置き・補足説明は不要
- goal_evidence の judgment は議事録内容のみで判断。曖昧な場合は no_evidence にする
- next_steps は [{user_display_name}] と [チーム] のアサインのみ抽出。他者アサインは除外
- 議事録が1件も取得できなかった場合は全フィールド空配列で返す

## エラー時の振る舞い
- MCP ツールの呼び出しが失敗した場合、取得できたデータのみで出力を構成する
- 全データ取得に失敗した場合は以下を返す:
  minutes_summary:
    error: "{エラーの概要}"
    activity_log_entries: []
    goal_evidence: []
    next_steps: []
```

---

## 出力スキーマの補足

3つの subagent の出力に共通する構造:

| フィールド | 用途 | 消費先 |
|-----------|------|--------|
| `activity_log_entries` | 実績記録（1ソース1行） | Phase 5: days/*.yml の activity_log |
| `goal_evidence` | 達成判定の証拠 | Phase 2: reconciliation |
| `next_actions` / `next_steps` | ゴール候補 | Phase 3: ゴール提案の材料 |

メイン側は `error` フィールドの有無をチェックし、エラーがあればユーザーに通知して手動入力を促す。
