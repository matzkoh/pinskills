# pinskills

[English](README.md) | 日本語

`npx skills` で入れたエージェント用スキルをコミット SHA に固定し、上流の変更を審査してから固定先を進めるための CLI。

## 背景

`npx skills`（vercel-labs/skills）は、初回の `add` で skills.sh の第三者監査（Gen Agent Trust Hub、Socket、Snyk）の結果を表示する。
ただし表示するだけで、Critical と判定されてもインストールは止まらない。
`update` では監査そのものが走らないので、インストール後に上流の中身がプロンプトインジェクションを含むものに差し替わっても、そのまま取り込まれる。

pinskills はこの穴を、検出の精度ではなく変更の管理で塞ぐ。
スキルをコミット SHA に固定しておけば、`npx skills update` を実行しても中身は変わらない。
中身を変えられるのは、審査を通った SHA へ固定先を進める `approve` だけになる。

スキルを入れる `skills` CLI 自体も同じ扱いにする。
`npx -y skills` は実行のたびに最新版を取ってきて、依存（`tar`、`yaml` など）も semver の範囲で解決する。
CLI か依存が乗っ取られると、審査した SHA を渡しても別の中身を入れられるので、CLI も依存ごと固定して審査する。

## 仕組み

### SHA 固定

`npx skills add owner/repo#<SHA>` で入れたスキルは、lock ファイル（`~/.agents/.skill-lock.json`）に `ref` として SHA が記録される。
`update` はこの `ref` の時点のツリーとしか比較しないので、上流が進んでも更新対象にならない。

### 審査（check）

`check` は lock の各スキルについて、上流の HEAD を SHA に解決し、固定中のツリーとの差分を取る。
差分は次の三段で審査し、すべて通ったものだけを **PASS** とする。

- **ルール検査**：不可視文字、`curl | sh`、長い base64 の塊、秘密情報のパス、指示の上書きを狙う定型句、破壊的なコマンド、データ送信先として知られるサービス。バイナリファイルと symlink、既存ファイルのモード変更も、中身を審査役が読めないので人間に回す。
- **claude**：`--setting-sources ""` と `--tools ""` で起動する。ユーザーの CLAUDE.md、スキル、プラグイン、MCP、hook を読まず、ツールを一つも持たない。
- **codex**：空の HOME（`auth.json` だけを symlink）で、`--sandbox read-only` のもとで起動する。接続アプリ、プラグイン、web 検索はアカウント側に紐付いていて HOME を空にしても消えないので、フラグで明示的に外す。

審査役が騙されても、外部へデータを送ったりファイルを書き換えたりはできない。
騙されて素通りさせる危険には、二つの審査役がそろって `pass` を返したときだけ通すことで備える。
時間切れ、異常終了、出力が schema に合わない場合は、すべて失敗として扱う。

差分が大きいときは、ファイル単位（巨大なファイルは行単位）で約 250KB ずつに分けて審査する。
分けた審査のうち一つでも `fail` かエラーなら、そのスキル全体を PASS にしない。

差分の本文は `reports/` に書き出すだけで、標準出力には出さない。
pinskills を実行するのがエージェントであっても、信頼できない差分がその文脈に入らないようにするためである。

### skills CLI の固定

pinskills はスキルの追加・削除に `npx skills` を使わず、自分で固定した skills CLI を呼ぶ。
固定は `~/.local/share/pinskills/cli/` に置いた package-lock.json で行い、依存まで版と integrity が決まる。
`check` では `skills-cli` という名前で、スキルと同じ一覧に並ぶ。

- 最新版の `skills` を依存ごと解決した lock を作り、固定中の lock と比べる。依存だけが変わった場合も更新として扱う。
- 両方の lock の各パッケージを npm レジストリから取り、integrity を確かめて展開し、git のツリーにしてから差分を取る。
- 差分はスキルと同じルール検査と二つの審査役に通す。審査役には CLI 向けの観点（指定と違う中身を入れる処理、秘密情報の送信、不審な依存など）を渡す。
- 実行されない `*.map` と型定義（`*.d.ts` など）は審査から外す。同梱の `.gz` は展開してテキストとして審査する。
- CLI は `DISABLE_TELEMETRY=1` と `DO_NOT_TRACK=1` を付けて呼ぶ。skills は既定で、入れたり消したりしたスキルの取得元（非公開リポジトリ名を含む）を skills.sh に送るため。
- `approve skills-cli` は、審査した lock を別の場所に `npm ci --ignore-scripts` で入れてから差し替える。失敗しても固定中の CLI は壊れない。

### 承認（approve）

`approve` は、直近の `check` で PASS になった SHA へ固定先を進める。
PASS 以外のスキルを進めるには、差分を読んだうえで `--force` を付ける。
進めた後は lock の `ref` とフォルダのハッシュを照合し、審査した中身が入ったことを確かめる。

## 必要なもの

- Node.js 22 以上
- git
- ログイン済みの `claude`（Claude Code）と `codex`（Codex CLI）

## インストール

```sh
npm install -g github:matzkoh/pinskills
```

手元で改造しながら使う場合は、clone して `npm link` する。

```sh
git clone https://github.com/matzkoh/pinskills.git
cd pinskills
pnpm install
npm link
```

## 使い方

### 初回

まだ固定していないスキルを、いま入っている中身のコミットに固定する。
固定するだけなので、中身は変わらない。

```sh
pinskills pin-current
```

skills CLI がまだ固定されていなければ、`npx skills` がいま取ってくるのと同じ最新版を依存ごと固定する。
スキルの固定には固定した CLI を使うので、CLI が先に入る。

続けて、いま入っている中身の全体を審査する。

```sh
pinskills check --baseline
```

### 定期的な確認

```sh
pinskills check
```

端末から引数なしで `pinskills` を起動すると、更新チェックを実行するか聞かれる。
既定は Yes なので、Enter だけで `check` に進む。

上流の取得はスピナーで、審査の進み具合は進捗バーとスキルごとの行で表示される。
審査は最大4スキルを並列に進める。
各行は待機中、審査中（審査役ごとに何分割中いくつ終えたか）、結果（PASS か NEEDS-REVIEW）の順に、同じ位置で更新される。

終わると、各スキルが次のいずれかに分類され、色分けして表示される。

- **最新**（緑）：上流に変更がない。1行にまとめて表示する。
- **PASS**（緑）：上流に変更があり、審査をすべて通った。変更内容の要約が付く。
- **NEEDS-REVIEW**（赤）：ルール検査に引っかかったか、審査役のどちらかが `pass` を返さなかった。引っかかったルール、各審査役の判定と指摘、差分ファイルの場所が付く。差分を読んで判断する。

次の三つは黄色の注意として表示される。

- **比較不能**：固定中のコミットが上流に見つからない（force-push で履歴が書き換わった場合など）。
- **フォルダがない**：上流の HEAD からスキルのフォルダが消えた（削除や別リポジトリへの移動）。固定中のコミットはそのまま使える。不要なら削除できる（下記）。
- **未固定**、**審査対象外**：SHA 固定されていない、または GitHub 以外から入れたスキル。`update` で中身が変わりうる。

一つでも PASS か最新以外があれば、終了コードは 1 になる。

終わると、片付けられる項目ごとに次の順で確認される。
`y` を押して Enter で、その場で実行する。

1. 未固定のスキルや skills CLI があれば「未固定の N 件をいまの中身に固定しますか？」。`pin-current` と同じ固定を行う。
2. PASS があれば「PASS の N 件を更新しますか？」。固定先を進める。
3. フォルダがないスキルがあれば「上流から消えた N 件を削除しますか？」。

どれも既定は No なので、Enter だけでは何も変わらない。

この確認は、端末に人がいるとき（標準入出力がともに TTY で、Claude Code の配下でないとき）だけ出る。
それ以外では確認を出さず、`approve`・`remove`・`pin-current` のコマンドを案内して終わる。
出力も、スピナーをアニメーションさせない制御文字なしの行になる。

### 承認

```sh
pinskills approve <skill...>
pinskills approve <skill...> --force   # NEEDS-REVIEW を差分を読んだうえで進める
```

承認は人間が行う前提で作っている。
Claude Code から使う場合は、プロンプトで `!` を付けて自分で実行する。

### 削除

```sh
pinskills remove <skill...>
```

固定した skills CLI の `skills remove` に任せて削除し、lock から消えたことを確かめる。
上流から消えたスキルの片付けを想定している。

## エージェントに承認させない設定

Claude Code では、`~/.claude/settings.json` の `permissions.deny` に次を入れておくと、エージェントが自分でスキルを追加したり承認したりできなくなる。

```json
"Bash(*skills add *)",
"Bash(*skills a *)",
"Bash(*skills install *)",
"Bash(*skills i *)",
"Bash(*skills experimental_install*)",
"Bash(*skills use *)",
"Bash(*skills sync*)",
"Bash(*skills@* add *)",
"Bash(*skills@* a *)",
"Bash(*skills@* install *)",
"Bash(*skills@* i *)",
"Bash(*pinskills* approve*)",
"Bash(*pinskills* pin-current*)"
```

この設定が効くのは Claude Code だけである。
`~/.agents/skills` を共有している Codex や GitHub Copilot が `npx skills add` を実行するのは止められない。

## ファイルの置き場所

| 用途 | 場所 |
| --- | --- |
| 審査記録（`state.json`）とレポート | `~/.local/state/pinskills/`（`XDG_STATE_HOME` に従う） |
| 固定した skills CLI | `~/.local/share/pinskills/cli/`（`XDG_DATA_HOME` に従う） |
| 上流リポジトリのキャッシュ | `~/.cache/pinskills/`（`XDG_CACHE_HOME` に従う。消しても作り直される） |
| 守る対象 | `~/.agents/skills/` と `~/.agents/.skill-lock.json` |

審査記録を `~/.agents` の外に置いているのは、Codex や Copilot も書き込む場所から離すためである。

テスト用に、次の環境変数で場所を差し替えられる。

- `PINSKILLS_AGENTS_DIR`：lock ファイルを探すディレクトリ（既定は `~/.agents`）
- `PINSKILLS_STATE_DIR`：審査記録とレポートの置き場所
- `PINSKILLS_DATA_DIR`：固定した skills CLI の置き場所

## 限界

pinskills は危険な変更を確実に検出するものではない。
プロンプトインジェクションは普通の文章として書けるので、ルール検査も LLM による審査も見逃すことがある。
確実に言えるのは、「審査を通っていない変更は `approve` を経ずに入らない」ことまでである。

- 二つの審査役を同時に騙す文面には、最後の人間の確認でしか対抗できない。
- codex の審査役は、書き込みと外部への送信はできないが、ローカルのファイルは読める。
- `reports/` の差分ファイルは信頼できない内容をそのまま含む。エージェントに読ませるときは注意する。
- 審査役の判定は、スキルを読み込んだエージェントがどう振る舞うかを実際に試したものではない。
