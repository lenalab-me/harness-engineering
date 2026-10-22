# harness-engineering

コーディングエージェントのスキル（`skills/`）と、自作の道具（rpatch・devsrv・agentlog）とスキルをほかの人の環境に入れる手順（[SETUP.md](SETUP.md)）を置く公開リポジトリ。ここにある道具とスキルを自分の環境に入れるときは、この文書ではなく SETUP.md に従う。

## 持ち主が手入れするとき

- 進め方・数え方・記録（何を入れ、どう測り、どうなったか）は、非公開の `~/project/prod/harness-engineering-private` にあり、そちらの CLAUDE.md に従う。記録はそちらに書く。
- ここには、測った数字・日付・効き目・持ち主の環境（パス、ほかのプロジェクトの名前、非公開リポジトリの中身、顧客や会社の名前）を書かない。スキルの例に出すコミットのハッシュはよい。コミットする前に差分を見る。
- `~/.claude/skills/grow-guardrails`・`~/.claude/skills/tidy-rules` は、このリポジトリの `skills/` へのリンク。スキルを直したら、そのまま効く。
- SETUP.md を変えたら、まっさらな環境（コンテナなど）で上から流して確かめる。
