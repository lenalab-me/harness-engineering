# 自分の環境に入れる

このリポジトリと、そこから使う自作の道具（rpatch・devsrv・agentlog）とスキルを、別の人の環境に入れる手順。読むのはその人のコーディングエージェント（Claude Code を想定）。上から順に進め、各段の「確かめ」が通ってから次へ進む。通らなければ、そこで止めて利用者に出力を見せる。

このリポジトリの `CLAUDE.md` は、持ち主がこのリポジトリを手入れするための決まり。入れるときには従わなくてよい（読むと何をしているかは分かる）。

## 何が入るか

| もの | 役目 | 入れ方 |
|---|---|---|
| rpatch | ファイルの編集を、完全一致・全部か無しかで行う CLI。`sed -i` や python の置換の代わり | Rust でビルド |
| rpatch の hook（`edit_guard.py`） | 確かめのない編集（`sed -i`、python の置換、追跡しているファイルへの `cat >`）を止め、rpatch を案内する | Claude Code の設定 |
| devsrv | 開発用サーバーの起動・待ち・停止。`cmd &`・`sleep` のループ・`pkill` の代わり | Rust でビルド |
| agentlog | Claude Code（と Codex）のセッションのログから、くり返しの失敗、言ったこと、メモリ、禁止ルールを引く | Rust でビルド |
| grow-guardrails | プロジェクトの関所（Schema・テスト・check）を、痛みが出た分だけ育てる手順 | スキル |
| tidy-rules | メモリ・AGENTS.md・CLAUDE.md のルールの棚卸し | スキル |

どれも単独で使える。全部入れなくてよい。

## 0. 環境を決める

| 環境 | 入るもの |
|---|---|
| macOS | 全部（持ち主の環境。ここでだけ使い込んでいる） |
| Linux、Windows の WSL2（Ubuntu） | 全部。まっさらな Ubuntu 24.04（arm64）で、この手順の 1〜3・6 と3つの道具の `cargo test` が通ることを確かめてある。devsrv の Linux 版は使い込みが浅いので、下の「確かめ」を必ず流す |
| Windows そのもの | rpatch とスキルだけ。agentlog と devsrv は Unix の API を使うのでビルドできない。hook は未確認 |

Windows なら WSL2 の Ubuntu に Claude Code を入れ、その中で進めるのがいちばん楽。以下は macOS・Linux・WSL2 の手順。

## 1. 前提

要るのは、C のリンカ、git、curl、python3（hook を使うなら）、Rust の stable（1.88 以上）。リポジトリはどれも公開なので、GitHub のアカウントはいらない。

Ubuntu（WSL2 を含む）なら、apt を先に入れてから Rust を入れる（逆だと rustup がリンカが無いと警告する）。

```sh
sudo apt update && sudo apt install -y build-essential git curl ca-certificates python3
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

macOS なら、Command Line Tools（C のリンカ、git、python3）を入れてから Rust を入れる。`xcode-select --install` はダイアログを出してすぐ戻るので、ダイアログで入れ終えてから次の行を流す（入っていれば何もしない）。

```sh
xcode-select -p >/dev/null 2>&1 || xcode-select --install
```

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

rustup は PATH を足す行をシェルの設定に書くが、今開いているシェルには効かない。Claude Code の Bash は呼び出しのたびに環境を引き継がないので、Claude Code を開き直すまでは、cargo と入れた道具を使うコマンドの頭に `. "$HOME/.cargo/env" &&` を付ける（以下のブロックには付けてある。rustup 以外で入れた Rust なら外す）。

確かめ（順に流し、どれかが失敗したらそこで止める）：

```sh
. "$HOME/.cargo/env" && cc --version | head -1 && cargo --version && python3 --version
```

## 2. リポジトリを取ってくる

置き場所は自由。以下では `~/src` に置く。`~/src` 以外に置くなら、5・6 のパス（`~/src/...`）も読み替える。一時フォルダ（`/tmp`、`mktemp -d`）には置かない（5 の hook は一時フォルダのファイルを止めないので、確かめが黙って何も出さない）。

```sh
mkdir -p ~/src && cd ~/src
for r in harness-engineering rpatch devsrv agentlog; do git clone https://github.com/lenalab-me/$r; done
```

## 3. 道具をビルドして入れる

```sh
. "$HOME/.cargo/env" && for r in rpatch devsrv agentlog; do (cd ~/src/$r && cargo install --locked --path .) || break; done
```

`~/.cargo/bin` が PATH に入っていること（rustup が入れる）。

確かめ（それぞれ、出力の1行目が道具の説明になり、終了コード 0）：

```sh
. "$HOME/.cargo/env" && rpatch --help | head -3 && devsrv --help | head -3 && agentlog --help | head -3
```

動きの確かめ（1回の呼び出しで流す）：

```sh
. "$HOME/.cargo/env" && d=$(mktemp -d) && printf 'one\ntwo\n' > "$d/a.txt"
rpatch "$d/a.txt" <<'EOF' && cat "$d/a.txt"
<<<<<<< OLD
two
=======
TWO
>>>>>>> NEW
EOF
devsrv start check --port auto -- python3 -m http.server {port} && devsrv ls && devsrv stop check
agentlog rules --count; echo "agentlog: $?"
```

`a.txt` の2行目が `TWO` になる、devsrv が URL を返して止まる、agentlog が 0 か 3（ログやメモリのフォルダがまだ無い）で終わる、の3つを見る。

各道具の詳しい使い方は `<道具> --help`（要約、40 行以内）と、`agentlog help COMMAND`・`devsrv help COMMAND`・`rpatch --help TOPIC`。

## 4. エージェントに道具を案内する

道具は、エージェントが知らなければ使われない。`~/.claude/CLAUDE.md`（無ければ作る）に、使う道具の行を足す。持ち主が使っている文をそのまま置く（英語のままでよい）。

```markdown
# CLI tools

- Edit files: To change a file, always use `rpatch` (format: `rpatch --help`; `<<<<<<< OLD part` covers part of a line). Chain follow-up commands on the opening line (`rpatch <<'EOF' && …`): a command after the closing EOF runs even when the edit failed. Don't use a python heredoc
- Background processes (dev servers, watchers): `devsrv start NAME --port auto -- CMD {port}` returns the URL once it answers; then `devsrv logs`, `ls`, `who PORT`, `stop NAME`. To wait for anything, `devsrv wait URL [--status 200]`, `wait port N`, `wait file PATH REGEX` or `wait cmd -- CMD` (until it exits 0). Use them instead of `cmd &`, `sleep`/`until` loops, `lsof -i` and `pkill` (`devsrv --help`: a summary; `devsrv help COMMAND` for one command)
- Past Claude Code sessions (what was said, why something was decided): `agentlog search REGEX --who user` lists matches with time and session id, then `agentlog show ID --at TIME` reads the talk around one. `agentlog timeline --repo DIR --since WHEN` lines up the user's words and the repo's commits by time (`agentlog --help`: a summary; `agentlog help COMMAND` for one command). Reach for them when the user points to earlier work, and before asking the user something that may already have been decided
- Memory of other projects and rules: `agentlog memory REGEX [--type feedback]` searches every project's memory files; use it when needed, not every session. Before adding a rule to memory or AGENTS.md, `agentlog rules --repo .` lists the prohibitions a session here reads, so you can rewrite a clashing one instead of adding another
```

Codex を使うなら `~/.codex/AGENTS.md` にも同じ行を写す。

確かめ：新しいセッションを始めて「python3 -m http.server を裏で立てて、URL を教えて、止めて」と頼み（道具の名前は出さない）、`devsrv start` が使われること。

## 5. hook を入れる（任意）

rpatch の hook は、`sed -i`・python の置換・追跡しているファイルへの `cat >` などを止め、rpatch を使うよう案内する。`~/.claude/settings.json` の `hooks` に足す（既にある `hooks` は消さずに、`PreToolUse` の並びに加える）。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 \"$HOME/src/rpatch/plugin/hooks/edit_guard.py\"", "timeout": 10 }
        ]
      }
    ]
  }
}
```

確かめ（Claude Code が渡すのと同じ形の入力を hook に流す。止めるときは `permissionDecision` が `deny` の JSON を出す。git で追跡していないファイルへの `sed -i` は止めないので、追跡しているファイルで試す）：

```sh
cd ~/src/rpatch && echo '{"tool_name":"Bash","tool_input":{"command":"sed -i s/a/b/ README.md"},"cwd":"'"$PWD"'"}' | python3 ~/src/rpatch/plugin/hooks/edit_guard.py
```

そのあと新しいセッションで、git で追跡しているファイルに `sed -i` をかけるよう頼み、hook に止められて rpatch を案内されること。

## 6. スキルを入れる

Claude Code は `~/.claude/skills/<名前>/SKILL.md` を読む。コピーでもよいが、リンクにすると `git pull` で新しくなる。

```sh
mkdir -p ~/.claude/skills
for s in grow-guardrails tidy-rules; do ln -sfn ~/src/harness-engineering/skills/$s ~/.claude/skills/$s; done
ln -sfn ~/src/rpatch/plugin/skills/rpatch ~/.claude/skills/rpatch
```

スキルの中には、持ち主の環境のパスと、ここに入っていないものが出てくる。読み替える。

- `~/project/prod/<名前>` は、持ち主のリポジトリの置き場所。上の手順なら `~/src/<名前>`。
- grow-guardrails の「Web アプリなら雛形 app-template から」の雛形は非公開で、入っていない（無くてもスキルは使える。その段落は飛ばす）。共通部品 sx-ui は公開している（[lenalab-me/sx-ui](https://github.com/lenalab-me/sx-ui)）が、使うかどうかは自由。
- grow-guardrails の例のコミット（`git show <ハッシュ>`）は network-designer という持ち主の非公開リポジトリのもので、見られない。サインと関所の対応の表だけで使える。
- agentlog の `--root ~/.local/share/agent-log-archive/...`（消えたログの写し）は、持ち主が macOS の launchd で回している写しで、このリポジトリには入っていない。無ければ使わない。

確かめ：新しいセッションで `/grow-guardrails` と打ち（または「このプロジェクトの関所を見直して」と頼み）、スキルが読まれること。

## 7. 入れたあと

- 何が効いたかを測るなら、agentlog の [使いどころ](https://github.com/lenalab-me/agentlog/blob/main/docs/recipes.md) に、道具や案内を入れた時刻を `--since` に渡して前後を比べるやり方がある。
- 入れたのに使われていないものは `agentlog unused --since 30d` で見る（入れた MCP・スキル・「CLI tools」に挙げた道具を、使われた回数の少ない順に並べる）。
- 道具を新しくするときは `cd ~/src/<道具> && git pull && cargo install --locked --path .`。
- うまく動かなかったところは、持ち主に出力ごと知らせる（この手順を直す）。
