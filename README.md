# your-scrum-routine

あなた専属のスクラムルーティン。一人スクラムのセレモニーを Claude Code が伴走する。

## スキル一覧

| スキル | 型 | 説明 |
|---|---|---|
| retrospective | ツール型 | 思考のペースメーカー。目標に向かって毎日一番効くアクションを提案する |

## インストール

### GitHub リポジトリから（推奨）

```bash
# 1. marketplace として登録
claude plugin marketplace add nakagam3/your-scrum-routine

# 2. Plugin をインストール
claude plugin install your-scrum-routine

# 3. セッションを再起動して反映
```

### ローカルパスから

```bash
claude plugin marketplace add /path/to/your-scrum-routine
claude plugin install your-scrum-routine
```

### アップデート

```bash
# marketplace のソースを最新に同期
claude plugin marketplace update your-scrum-routine

# Plugin を最新バージョンに更新（セッション再起動で反映）
claude plugin update your-scrum-routine
```

## peerDependencies

| デッキ | 必須 | 用途 |
|---|---|---|
| [development-deck](https://github.com/lmi-mcs/development-deck) | No | 振り返りの深掘り・設計相談（dp, design-partner 経由） |

## 依存する外部ツール

retrospective スキルは以下の MCP / CLI を使用する。いずれも必須ではなく、利用できない場合は該当データソースをスキップして動作する。

### MCP（Claude Code 統合）

| MCP | 用途 | セットアップ |
|---|---|---|
| Google Calendar | 今日/明日の予定取得、MTG 特定 | [Google Workspace connectors (Claude Help Center)](https://support.claude.com/en/articles/10166901-use-google-workspace-connectors) |
| Slack | 当日の Slack 活動収集 | [Slack MCP Server - Connect to Claude (Slack Developer Docs)](https://docs.slack.dev/ai/slack-mcp-server/connect-to-claude/) |

### CLI

| CLI | 用途 | セットアップ |
|---|---|---|
| `gh` (GitHub CLI) | contributionsCollection の取得（PR/commit 実績） | [github.com/cli/cli](https://github.com/cli/cli) |
| `gws` (Google Workspace CLI) | Gemini 議事録の取得 | [github.com/googleworkspace/cli](https://github.com/googleworkspace/cli) |

### 前提

| ツール | 備考 |
|---|---|
| Claude Code | [セットアップガイド (Anthropic Docs)](https://docs.anthropic.com/en/docs/claude-code/setup) |
