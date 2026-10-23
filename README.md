# harness-engineering

コーディングエージェント（主に Claude Code）が、同じ失敗をくり返す・待つ・やり直すところを減らすための、スキルと道具の入口。道具（rpatch・devsrv・agentlog・wtprune）は、それぞれのリポジトリにある。

- **自分の環境に入れたい**：[SETUP.md](SETUP.md)。エージェントに「SETUP.md に従って入れて」と頼めば進められるように書いてある。
- **スキル**：[skills/grow-guardrails](skills/grow-guardrails/SKILL.md)（プロジェクトの関所を、痛みが出た分だけ育てる）、[skills/tidy-rules](skills/tidy-rules/SKILL.md)（メモリ・AGENTS.md・CLAUDE.md のルールの棚卸し）。
- **道具**：[rpatch](https://github.com/lenalab-me/rpatch)（ファイルの編集）、[devsrv](https://github.com/lenalab-me/devsrv)（開発用サーバーの起動・待ち・停止）、[agentlog](https://github.com/lenalab-me/agentlog)（セッションのログとメモリを引く）、wtprune（T3 Code が作った worktree とブランチのうち、マージ済みで、もう誰も使っていないものだけを消す。T3 Code を使う人向けで、リポジトリは非公開）。

持ち主が自分の作業のために作り、使っているもの。スキルの中の例（コミットのハッシュ、network-designer などの名前）は持ち主の手元のもので、見られなくても使える。
