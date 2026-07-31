---
title: "【shell】検査は「該当なし」を返し続けた。違反は入っていたのに"
emoji: "🤫"
type: "tech"
topics: ["個人開発", "shell", "git", "自動化", "cloudflarepages"]
published: true
---

ビルドは成功。デプロイも成功。HTTP も 200。それでも、本番に出たのは古い成果物でした。

前回、そのデプロイ事故を記事にしました。処理が失敗したのではありません。成功したのに、結果が間違っていたのです。原因は「成功条件の不足」でした。

書いたあとで気づきました。あれは特殊な1例ではありません。以下では、実際に踏んだ問題の原因を、手元で再現できる最小例にして示します。掲載するコマンドと出力は、次の環境で動かした結果です。

```text
macOS 26.5.2 (arm64) / git 2.50.1 / zsh 5.9 / bash 3.2.57 / Python 3.9.6
```

## なぜ「成功する失敗」が危ないのか

エラーで落ちる失敗は、実はありがたい。赤信号が出るので、その場で気づいて直せます。

こわいのは、コマンドが返した結果を、成果物の正しさと取り違えることです。多くは `exit 0` や HTTP 200 として現れます。非0の意味を読み違える場合もあります。どちらも警告は出ません。だから間違ったまま次へ進み、誤った成果物が後工程へ流れます。前回のデプロイ事故も、これでした。以下、原因の位置に沿って4つに分けます。

## 型1：検査ロジック自体が、想定と違う動きをする

チェックを書いても、そのチェックが動いていないことがあります。しかも `exit 0` を返します。

**シェルの単語分割**。表記ゆれの検査が、いつも「該当なし」を返していました。原因は、bash の単語分割を前提にしたコードを zsh で動かしていたことです。zsh は既定で、引用していない変数展開を単語に分けません。

```console
$ tmp=$(mktemp -d) && cd "$tmp"
$ cat > check.sh <<'EOF'
pairs="デプロイ:ディプロイ ログイン:ログオン"
hit=0
for pair in $pairs; do
  IFS=: ; set -- $pair ; unset IFS
  ng="$2"
  [ -n "$ng" ] || continue
  if grep -Fq -- "$ng" target.txt; then echo "違反: $ng"; hit=1; fi
done
[ $hit -eq 0 ] && echo "該当なし"
exit $hit
EOF
$ printf 'このツールはディプロイに失敗した\n' > target.txt
$ bash ./check.sh ; echo "exit=$?"
違反: ディプロイ
exit=1
$ zsh ./check.sh ; echo "exit=$?"
該当なし
exit=0
```

`target.txt` には、わざと違反を入れてあります。bash は検出しました。zsh は「該当なし」を出して `exit 0` を返しました。`$pairs` も `$pair` も分割されないため `$2` が空になり、`grep` が一度も走らないからです。

直すなら、文字列に複数の値を詰めず、配列で持ちます。

```console
$ cat > check2.zsh <<'EOF'
pairs=("デプロイ:ディプロイ" "ログイン:ログオン")
hit=0
for pair in $pairs; do
  ng="${pair#*:}"
  if grep -Fq -- "$ng" target.txt; then echo "違反: $ng"; hit=1; fi
done
(( hit == 0 )) && echo "該当なし"
exit $hit
EOF
$ zsh ./check2.zsh ; echo "exit=$?"
違反: ディプロイ
exit=1
```

検索は `grep -Fq --` にしてあります。`-F` は固定文字列として扱う指定です。これが無いと、`.` や `*` を含む語を登録したときに正規表現として解釈されます。

`setopt SH_WORD_SPLIT` を有効にすると、引用していないパラメータ展開に、他の多くのシェルへ近い分割規則が適用されます。ただしシェル全体が bash 互換になるわけではなく、他の展開にも影響します。

**バッククオートのコマンド置換**。`python3 -c "..."` のダブルクオートの中にバッククオートを書いたら、保存された値が空になりました。これは zsh 固有ではありません。ダブルクオートの中でも、バッククオートは POSIX 系シェルが先に評価します。

```console
$ tmp=$(mktemp -d) && cd "$tmp"
$ python3 -c "
open('out.txt','w').write('path=[`false`]')
" ; echo "python_exit=$?"
python_exit=0
$ cat out.txt
path=[]
```

`false` は非0で終わりますが、何も表示しません。挿入されるのは空文字です。標準エラーを出すコマンドであっても、外側の Python が正常終了すれば、最終の終了コードは 0 になります。残るのは、壊れた `out.txt` だけです。

対策は、コードをシングルクオートで囲むことです。あるいは値を引数で渡し、コードとデータを分けます。

検査スクリプトには、正常系だけでなく、意図的な違反を検出できるテストも必要でした。

## 型2：入力が、調べたい対象を十分に表していない

この型だけは、コマンドが成功を返した例ではありません。共通しているのは、終了コードが表す条件を、こちらの知りたい条件と取り違えたことです。

`git check-ignore` で踏みました。`.gitignore` に `dist/` があるのに、`dist` をまだ作っていない状態で確認すると、こうなります。

```console
$ tmp=$(mktemp -d) && cd "$tmp" && git init -q .
$ printf 'dist/\n' > .gitignore
$ git check-ignore dist ; echo "exit=$?"
exit=1
$ git check-ignore dist/ ; echo "exit=$?"
dist/
exit=0
$ mkdir dist
$ git check-ignore dist ; echo "exit=$?"
dist
exit=0
```

`dist/` は末尾スラッシュ付きなので、ディレクトリにだけ一致します。`dist` が実在しない状態でスラッシュ無しの文字列を渡すと、ディレクトリだという情報がどこにもありません。だから exit 1 が返ります。実際に `mkdir dist` すると、スラッシュ無しでも exit 0 に変わりました。

ここで exit 1 を「ignore されていない」と受け取ると、静かに間違えます。その前提でデプロイガードを設計してしまうからです。コマンドへ渡した文字列が、調べたい対象の種類や状態を表しているか確認します。

## 型3：コマンドの許容範囲が、検査条件より広い

「既にあっても文句を言わない」「多少ズレても当てる」コマンドは、ふだん便利です。ただ、その許容範囲が検査条件より広いと、確認したかったことが通ってしまいます。

```console
$ tmp=$(mktemp -d) && cd "$tmp"
$ mkdir sub ; echo "exit=$?"
exit=0
$ mkdir sub ; echo "exit=$?"
mkdir: sub: File exists
exit=1
$ mkdir -p sub ; echo "exit=$?"
exit=0
$ printf 'OLD\n' > a ; printf 'NEW\n' > b
$ cp b a ; echo "exit=$? / a=$(cat a)"
exit=0 / a=NEW
```

`mkdir -p` は、対象がすでにディレクトリとして存在していても、それをエラーにしません。こちらが確かめたかったのは「実行前には存在しなかったこと」でした。`cp` も、書き込み可能な既存ファイルの内容を、標準では確認なしに上書きします。

この2つは、実際に組み合わせて事故になりました。連番のバックアップフォルダを新しく作るつもりで `mkdir -p` を使い、そこへ `cp` しました。フォルダは既にあり、中身は別の作業の途中経過でした。git の管理外だったので、上書きされた内容はどこにも残っていません。

`git apply` も許容範囲が広めです。ハンク（パッチ内の変更のまとまり）の行番号がズレても、コンテキスト行が一致すれば別の位置へ当てます。パッチに記録された位置と、実際に適用された位置との差が offset です。

```console
# 共通準備（commit は不要。インデックスに載せれば復元元になる）
$ tmp=$(mktemp -d) && cd "$tmp" && git init -q .
$ seq 1 20 | sed 's/^/line/' > f.txt
$ git add f.txt
$ sed -i '' 's/^line10$/line10-CHANGED/' f.txt
$ git diff > p.patch
$ git restore f.txt
```

```console
# 実験A：先頭に3行足してから当てる
$ printf 'x\nx\nx\n' | cat - f.txt > t && mv t f.txt
$ git apply p.patch ; echo "exit=$?"
exit=0
$ git restore f.txt && printf 'x\nx\nx\n' | cat - f.txt > t && mv t f.txt
$ git apply --verbose p.patch ; echo "exit=$?"
Checking patch f.txt...
Hunk #1 succeeded at 10 (offset 3 lines).
Applied patch f.txt cleanly.
exit=0
```

```console
# 実験B：元に戻し、コンテキスト行を1つ書き換えてから
$ git restore f.txt
$ sed -i '' 's/^line12$/line12-EDITED/' f.txt
$ git apply --check p.patch ; echo "exit=$?"
error: patch failed: f.txt:7
error: f.txt: patch does not apply
exit=1
```

手元の git 2.50.1 では、オプション無しの `git apply` は成功時に何も出しませんでした。`--verbose` を付けて初めて、3行ズレて当てたと分かります。

`patch` コマンドなどで、前後のコンテキスト行の一部を無視して当てる挙動は fuzz と呼ばれます。今回の `git apply` の既定動作はそれではありません。全てのコンテキストが一致する別の位置を探して当てた offset です。この再現例の既定設定では、必要なコンテキストを書き換えると適用を拒否しました。なお `--ignore-whitespace` などで一致条件は変えられます。

位置のズレは吸収しますが、パッチが照合する変更行や必要なコンテキストが一致しなければ適用を拒否します。成功しても「想定した位置に当たったか」は、終了コードだけでは分かりません。これらの成否を検知器として当てにしないことです。

## 型4：ツールの成功条件が、成果物に必要な条件より弱い

ここまでの例は、冒頭に記載した筆者環境では、掲載したブロックだけで再現できます。次の Cloudflare Pages の例は、実プロジェクトの URL を伏せた実測ログです。

Cloudflare Pages には、SPA フォールバックという既定動作があります。今回のプロジェクトは、静的アセットを既定設定で配信していました。トップレベルに `404.html` が無いので、存在しないパスにもルートの `index.html` が返ります。

```console
$ U=https://<project>.pages.dev
$ for p in / /version-does-not-exist.json /version.json; do
>   curl -sS -o /dev/null \
>     -w "$p %{http_code} %{content_type}\n" "$U$p"
> done
/ 200 text/html; charset=utf-8
/version-does-not-exist.json 200 text/html; charset=utf-8
/version.json 200 application/json
$ curl -sS "$U/" | shasum -a 256
7469fd0b2e5efc7faad35ede2b6c2575e506dbbc95b587b3b046d6ecc743dddb  -
$ curl -sS "$U/version-does-not-exist.json" | shasum -a 256
7469fd0b2e5efc7faad35ede2b6c2575e506dbbc95b587b3b046d6ecc743dddb  -
```

存在しない JSON を取りに行っても 200 が返り、本文はルートと同じハッシュでした。SPA フォールバックで選ばれた HTML です。

200 が示すのは、HTTP としてリクエストが成功したことです。ただし、その成功条件は「期待した JSON を取得した」というアプリ側の条件より弱いことがあります。`response.ok` だけを見て本文を後工程へ渡すと、HTML を成果物として扱ってしまいます。

判定は3段に分けました。まず `Content-Type` から `; charset=...` などのパラメータを外します。次にメディアタイプが `application/json` と一致するか、サブタイプが `+json` で終わるかを見ます。`application/problem+json` のような形があるので、前方一致では判定できません。

```js
const mediaType = res.headers.get("content-type")
  ?.split(";", 1)[0].trim().toLowerCase();
const isJson = mediaType === "application/json"
  || mediaType?.endsWith("+json");
```

最後に本文をパースし、必須キーまで確認します。ここまでやって、ようやく「JSON が取れた」と言えます。

## 対策：成功を証明として扱わない

共通していたのは、ツールの成功条件が、成果物に必要な条件より弱かったことです。

- **成功条件を言語化する**。「何が返れば正しいのか」を、`exit 0` や 200 とは別に書き出します。
- **型とスキーマを検証する**。`Content-Type`、JSON としてのパース、必須キーの有無まで確認します。
- **わざと異常な入力を入れる**。検査が落ちることを一度確かめます。品質ゲート自体が壊れていると、違反を見逃して合格を返すことがあります。
- **最終成果物を直接見る**。前回のデプロイでは、本番の `version.json` にコミットSHAを刻み、出したものを SHA で照合しました。

どれも、コマンドの表示を信じる代わりに、成果物そのものを確かめる、という同じ発想です。

## まとめ

自動化は、成功をたくさん返します。`exit 0`、HTTP 200、緑のチェック。どれも、それぞれの仕組みが定める成功条件を満たしたことは示します。ですが、成果物がこちらの条件を満たしたかは別です。

成功表示は入口として扱い、成果物で裏を取る。それだけで、静かにすり抜ける失敗を、早い段階で見つけやすくなります。
