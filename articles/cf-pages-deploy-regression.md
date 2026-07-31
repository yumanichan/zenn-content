---
title: "【Cloudflare Pages】デプロイ成功。なのに本番の見た目だけ2週間前だった"
emoji: "⏪"
type: "tech"
topics: ["git", "worktree", "cloudflarepages", "wrangler", "個人開発"]
published: true
---

古い worktree でビルドした `dist` を、Cloudflare Pages の本番へ公開しました。結果、本番の表示から35コミット分の更新が消えました。ビルドもデプロイも、最後まで成功したままです。

原因は単純でした。`--branch=main` を付けても、アップロードされる `dist` の中身は変わりません。筆者は three.js 製ゲームを、複数の git worktree で並行開発しています。この記事では、そこで踏んだ事故の構造・実装したガード・実測・残る限界を、順に記録します。

## 本番の見た目が、過去に戻っていた

ゲームは Cloudflare Pages で公開しています。ある時、公開URLを開くと、直近で入れたはずの調整が画面から消えていました。ロールバックした覚えはありません。デプロイ失敗の通知もありません。それでも表示は、明らかに以前の状態でした。

キャッシュを疑って取り直しても変わりません。再読み込みしても、本番URLから配信される画面は古いブランチ側のままでした。当時はまだ後述の `version.json` を仕込んでおらず、確認は見た目の照合でした。

## 原因を、3つの層に分ける

「古いブランチを出した」で片付けると、同じ設計の穴が残ります。原因を3層に分けます。

- 増幅要因：worktree が11本あり、判断の対象が増えていた
- 直接原因：古いブランチの `dist` を本番へ出した
- 根本原因：deploy にブランチ・clean・origin 一致の検査がなかった

worktree はこの事故の主犯ではありません。ただ、担当領域ごとにブランチを分けて並行するうちに、数が増えていました。ある時点では、同じ Git リポジトリに紐づく worktree が11本です。

- `feat/fun-lab` — ゲーム性の実験
- `feat/tower-mode` — 新モード
- `feat/card-costs` — カード調整
- `feat/wolf-menu` — UI まわり

11本もあると「今どれが本番に出すべき最新か」は頭から抜けます。判断の対象が増えただけ、とも言えます。

実際に本番へ出たのは、ゲーム内容側の更新を含まない UI ブランチの `dist` でした。片方でゲーム内容側を、もう片方で UI 側を進めていました。ゲーム内容側が一段落したあと、UI 側からデプロイしたのです。当時の記録では、統合ブランチにあって UI ブランチに含まれないコミットが35件ありました。その35件を欠いたビルドが本番に出て、巻き戻りました。

そして deploy コマンドには、git の事前条件がありませんでした。「今出そうとしている `dist` は、本当に出すべきブランチから作ったものか」を、誰も確認していませんでした。

（誤デプロイの正確な時刻や復旧手順は当時の記録に残っていないため、ここでは触れません。）

## なぜ気づけないか：デプロイは古いdistでも成功する

この事故のいやらしいところは、どの工程もエラーを出さないことです。

ビルドもデプロイも、渡された入力を処理できるかしか見ません。それが意図したコミットかは確認しないので、古い `dist` でも最後まで正常に終わります。途中に赤信号は一つも出ません。

つまり、**「デプロイが成功した」は「正しいものが出た」を意味しません**。`wrangler pages deploy` の成功が示すのは、指定した `dist` をアップロードできたことだけです。それが正しいコミットから作られたかは、確かめません。

## 遠回りした対策1：日時で「最新ブランチ」を探す

最初に考えたのは、「デプロイ前に最新ブランチを探す」ことでした。こう並べれば一番上が最新だろう、と。

```bash
git branch -av --sort=-committerdate
```

これは誤りでした。`--sort=-committerdate` が並べるのは「最後にコミットした日時」順です。「どのブランチが他を包含しているか」は示しません。古いブランチに今日1コミット足せば、その枝が一番上に来ます。日時の新しさと、本番に出すべき内容を持っているかは別物です。

包含関係は、日時ではなく祖先関係で見ます。比較の前に、リモートの状態を取り直します。

```bash
# Bash
git fetch origin --prune

# 左=main だけにある / 右=feat だけにあるコミット数
git rev-list --left-right --count origin/main...origin/feat/X

# feat が main を含んでいるか（終了コード0なら含む）
git merge-base --is-ancestor origin/main origin/feat/X
echo $?
```

`git fetch` を挟むのは、`origin/main` がローカルに保存された古い参照だからです。取り直さなければ、リモートの現在と食い違います。出力の読み方はこうです。

```text
$ git rev-list --left-right --count origin/main...origin/feat/X
2    18
# 左の2  = origin/main にだけあるコミット
# 右の18 = origin/feat/X にだけあるコミット
```

どちらも相手の祖先でなければ、履歴は分岐した状態です。この時に件数の多い方を「最新」と決めると、もう片方を落とします。なお、この件数は履歴上のコミット数であって、包含の証明ではありません。cherry-pick で同じ変更が別コミットとして両方に載ることもあります。だから包含は、件数でなく `merge-base --is-ancestor` の終了コードで判定します。

## 遠回りした対策2：`--branch=main` を付ければ安全だと思った

もう一つ、デプロイ先が「プレビュー」になっていた時期がありました。deploy コマンドで `--branch=main` が抜けていて、本番URLが更新されなかったのです。そこで `--branch=main` を足したところ、プレビュー流出は直りました。

しかし、これは**退行を防ぐ対策ではありません**。`--branch=main` は、このデプロイを main ブランチ由来として扱わせる指定です。このプロジェクトでは main が本番ブランチなので、本番とプレビューの振り分けには効きます。ですが、アップロードする `dist` を main から作り直したり、main を checkout したりはしません。だから古いブランチで作った `dist` を出せば、古い内容が本番に出ます。退行はそのまま再現できます。

つまり、付け替えたのはデプロイ先の名札だけで、デプロイする中身は選んでいません。ここを取り違えていました。

## 実際に効いたガード：deployの前に拒否する

デプロイ先の指定を正すだけでは足りません。効いたのは、deploy コマンド自体にガードを組み込むことでした。出してはいけない条件なら、その場で止まります。手元は Windows と Mac の2台で開発しています。bash に依存しない Node 製（`.mjs`）にして、`npm run deploy` をこのスクリプト経由に差し替えました。

以下は判定部分の抜粋で、そのままでは動きません。実コードには、メッセージを出して終了する `fail()` と、子プロセスを実行する `run()` があります。`run()` は Windows と macOS の両方で動くよう、子プロセスの起動を分岐しています。

```javascript
// scripts/deploy-guard.mjs（判定部分の抜粋）
// execFileSync=node:child_process / rmSync,writeFileSync=node:fs
const DEPLOY_BRANCH = "feat/monster-tuning";  // 唯一のデプロイ元（統合ブランチ）
const git = (a, allowFail = false) => {
  try { return execFileSync("git", a, { encoding: "utf8" }).trim(); }
  catch (e) { if (allowFail) return ""; throw e; }
};

// 1) 統合ブランチ以外（detached HEAD 含む）は拒否
const branch = git(["symbolic-ref", "--short", "-q", "HEAD"], true) || "detached HEAD";
if (branch !== DEPLOY_BRANCH) fail(`deploy は ${DEPLOY_BRANCH} からのみ（今: ${branch}）`);

// 2) 追跡ファイルの未コミット変更があれば拒否（未追跡は許容＝後述）
if (git(["status", "--porcelain=v1", "--untracked-files=no"]) !== "") fail("未コミットの変更あり（追跡済みファイル）");

// 3) origin と一致しなければ拒否
run("git", ["fetch", "origin", DEPLOY_BRANCH, "--quiet"]);
const localSha = git(["rev-parse", "HEAD"]);
if (localSha !== git(["rev-parse", `refs/remotes/origin/${DEPLOY_BRANCH}`])) fail("origin と不一致");

// 4-5) dist を消して作り直し、build 前後の差分で「build が変えた分」だけを検知
const before = git(["status", "--porcelain=v1", "--untracked-files=normal"]);
rmSync("dist", { recursive: true, force: true });
run("npm", ["run", "build"]);
if (git(["status", "--porcelain=v1", "--untracked-files=normal"]) !== before) fail("build により作業ツリーが変更されました");

// 6) build 中に origin/HEAD が動いていないか再照合（並行 push を deploy 前に検知）
run("git", ["fetch", "origin", DEPLOY_BRANCH, "--quiet"]);
if (git(["rev-parse", "HEAD"]) !== localSha
    || git(["rev-parse", `refs/remotes/origin/${DEPLOY_BRANCH}`]) !== localSha) {
  fail("build 中に HEAD または origin が更新されました");
}

// 7) 出したコミットSHAを version.json に刻む
writeFileSync("dist/version.json", JSON.stringify({ commit: localSha, builtAt: new Date().toISOString() }) + "\n");

// 8) dry-run（検証用）。ここまで通し、本番へは出さずに終わる
if (process.env.DEPLOY_DRY_RUN === "1") { console.log("DEPLOY_DRY_RUN=1: wrangler deploy を実行せず終了"); process.exit(0); }

// 9) 本番へ。wrangler は devDependencies に exact 固定＝ローカル版のみ（--offline --no で取得禁止）
run("npm", ["exec", "--offline", "--no", "--", "wrangler", "pages", "deploy", "dist",
  "--project-name=glitch-survivors-ts", "--branch=main",
  `--commit-hash=${localSha}`, "--commit-dirty=false"]);
```

デプロイ元は統合ブランチ1本に固定しました。別ブランチ・追跡ファイルの変更・origin 不一致・build 中の更新のどれかがあれば止めます。origin は build の前と後の2回照合します。build の数秒の間に別セッションが同じブランチへ push しても、deploy の直前で気づけるようにしました。そのうえで `dist` は毎回消して作り直し、出したコミットSHAを `version.json` に刻みます。

`--commit-dirty=false` は、clean を確認済みだと Cloudflare 側の記録に反映するためです。`dist` とコミットの対応を作っているのは、このフラグではありません。clean 確認・origin 一致・再 build の一連です。wrangler も `devDependencies` に固定し、`npm exec --offline --no` でインストール済みのローカル版だけを実行します。

dirty の判定は、追跡ファイルだけを見ています。このリポジトリは並行セッションで開発しています。レビュー用のメモなど、意図して git 管理外に置いた未追跡ファイルが常にあるからです。未追跡まで数えると、ガードが毎回止まって使えません。ビルドが取り込むのは追跡済みの `src/` だけなので、未追跡は本番に混ざりません。build 後のチェックも同じ理由で、build 前後の差分だけを見ています。

デプロイ元を `main` でなく統合ブランチにしたのは、普段そこへ機能を集約しているからです。本番に出せる入口を、その1本に絞りました。

## ちゃんと止まるかを、実際に確かめる

事故に直結する条件ごとに、止まるかを実測しました。検証は使い捨ての一時 clone 内で、共有ブランチを汚さずに行いました。すべての実行に `DEPLOY_DRY_RUN=1` を付けています。別ブランチ・dirty・origin 不一致で止まるのは、build の前です。build 中の origin 更新だけは、build の後で止まります。判定を誤って抜けても、dry-run が動く限り wrangler へは届きません。

```text
# 統合ブランチ以外で実行
deploy は feat/monster-tuning からのみ（今: test-other-branch）
exit=1

# 追跡ファイルに未コミット変更がある状態で実行
未コミットの変更あり（追跡済みファイル）
exit=1

# ローカルだけ1コミット進み、origin と不一致の状態で実行
origin と不一致
local : e1f66a0…
origin: 4a8f2f6…
exit=1

# build 中に別セッションが同じブランチへ push（並行 push を build 後に検知）
build 中に HEAD または origin が更新されました
origin: b077e42… / build時: 4a8f2f6…
exit=1

# 正常条件（clean・origin一致）＋ DEPLOY_DRY_RUN=1
build commit: 4a8f2f6…
version file: dist/version.json
DEPLOY_DRY_RUN=1: wrangler deploy を実行せず終了
exit=0
```

4つの拒否条件はすべて `exit=1` で止まりました。正常条件だけが build を通過し、dry-run で `exit=0` になりました。

## 出した内容を、SHAで照合する

以前は、コミットメッセージにビルドIDを手で書いて記録していました。ですが、そのビルドID（`assets/index-<hash>.js` のハッシュ）は、アセットの内容で決まる名前です。コミットSHAと1対1で対応する保証はありません。再ビルドや依存の変化でずれます。

そこで、ビルド時に `version.json` へコミットSHAを直接埋め込みました。上のスクリプトの `writeFileSync` の行です。埋め込んだ SHA が HEAD と一致することは、その場で確認できました。本番が今どのコミットかも、キャッシュを避けて取れば分かります。ただし一つ落とし穴があります。

このプロジェクトはトップレベルの `404.html` を置いていません。その場合、Cloudflare Pages は存在しないパスにもルートの `index.html` を返します。実際に本番で試すと、こうでした。

```text
$ curl -sI "https://<project>.pages.dev/does-not-exist"
HTTP/2 200
content-type: text/html; charset=utf-8

$ curl -sI "https://<project>.pages.dev/version.json"
HTTP/2 200
content-type: application/json
```

存在しないパスでも `200` が返ります。つまり `version.json` が未配信でも、body だけ見れば HTML を JSON と読み違えかねません。判定は、`content-type` が `application/json` で始まるかで行います。

```bash
# 出したコミットを手元に置いた状態で実行する前提
url="https://<project>.pages.dev/version.json?t=$(date +%s)"
expected="$(git rev-parse HEAD)"
ct="$(curl -fsS -H 'Cache-Control: no-cache' -o version.json -w '%{content_type}' "$url")"

case "$ct" in
  application/json*) ;;                    # 先頭一致。charset 付きも許容
  *) echo "想定外の content-type: $ct" >&2; exit 1 ;;
esac

actual="$(jq -er '.commit' version.json)" || { echo "commit を取得できません" >&2; exit 1; }
[ "$actual" = "$expected" ] && echo "一致: $actual" \
  || { echo "本番SHA不一致 expected=$expected actual=$actual" >&2; exit 1; }
```

これで、本番から取得した SHA とデプロイ対象の SHA が一致するところまで、機械で判定できます。ビルドIDの目視は、この照合の補助に格下げしました。

## このガードで防げないこと

この仕組みは、通常の `npm run deploy` 経路での操作ミスを止めるものです。完全なデプロイ統制ではありません。このガードには、次の制限があります。

- **build 中の origin 更新は検知して止めますが、最終照合の後、Cloudflare へのアップロードが終わるまでの窓は残ります**。クライアント側だけでは完全に閉じられません。
- `wrangler` を直接呼べば、ガードを通さずデプロイできます（`npm run deploy` を経由しない実行）。
- ビルド環境や依存が変われば、同じコミットでも出力は変わり得ます。`version.json` の SHA は、スクリプトの自己申告です。
- build 後に別プロセスが `dist` を書き換える経路までは、このガードは塞ぎません。

ここまで強制するなら、デプロイ権限を CI に寄せます。保護されたブランチのコミットだけを build・deploy する構成です。個人開発ではまず、手元の操作ミスを止めるところから始めました。

## まとめ：名札ではなく、事前条件で止める

worktree で複数ブランチを並行させる開発は強力です。ただ、「今どれが本番に出すべき最新か」を記憶に置くと、いつか古い枝を出します。しかもデプロイは、それを成功として通します。

効いた対策は、`--branch=main` を足すことではありませんでした。本番に出せるブランチを1本に固定しました。そのうえで clean と `origin` 一致、`dist` の作り直し、SHA 埋め込みを、deploy の前提条件にしました。指定を貼り替えるのではなく、出す前に条件で止める。それが、成功したデプロイに巻き戻された事故から学んだことです。
