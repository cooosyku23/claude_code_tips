# Claude Code Tips

[Claude Code](https://claude.ai/code)をより便利に使うためのTips集です。

## コンテンツ

| ガイド | 内容 |
|--------|------|
| [並列化ガイド](./01_Parallel_Guide/PARALLEL_GUIDE.md) | Git worktreeで複数セッションを同時実行し、待ち時間を有効活用して開発スピードを上げる |
| [Agent Teams 利用ガイド](./02_Agent_Teams/AGENT_TEAMS_GUIDE.md) | 複数のClaude Codeインスタンスをチームとして協調動作させ、多角的レビューや敵対的調査を行う |

## スラッシュコマンド

Agent Teamsガイドには、すぐに使えるカスタムスラッシュコマンドが付属しています。

| コマンド | 内容 |
|---------|------|
| `/agent-team` | タスクを入力すると、チーム構成・フェーズ設計・制約を含むエージェントチーム用プロンプトを自動生成する |

### 導入方法

[`02_Agent_Teams/.claude/`](./02_Agent_Teams/.claude/) をプロジェクトの `.claude/` にコピーしてください。
