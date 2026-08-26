---
title: "このブログの構成 — Hugo と Congo を Hugo Modules で運用する"
date: 2026-08-25T13:30:00+09:00
draft: false
categories: ["技術"]
tags: ["Hugo", "Congo", "GitHub Actions", "静的サイト"]
---

最初の記事は、このブログそのものの話にする。

何をどう選んで、いまどういう構成で動いているのか。とくに「テーマを Hugo Modules で入れる」という一点については、日本語の情報がまだ少ないわりに、後から助けられる場面が多いので、実際の設定ファイルをそのまま出しながら書く。

前半は選定の理由、後半は実装の中身という構成にした。

## 何を求めて個人サイトを持つのか

個人サイトに求めるものは、突き詰めると 3 つしかなかった。

まず、書くことに集中できること。記事を 1 本書くたびに管理画面にログインして、エディタの挙動と格闘して、プレビューが本番と違う、みたいなことをやりたくない。書きたいのはテキストであって、それ以外の操作は少ないほどいい。

次に、長く持つこと。個人サイトは書いた瞬間より、5 年後に読み返せることのほうが価値がある。サービスが終了したら消える場所や、独自形式でデータが閉じ込められる場所には置きたくない。

あとは、運用に手がかからないこと。サーバの面倒は見たくないし、セキュリティパッチのために夜中に起きていたくもない。月額を払い続ける前提も避けたい。

この 3 つを並べると、選択肢はかなり絞られる。Markdown をファイルとして持って、静的な HTML を吐いて、それをどこかに置くだけ。要するに静的サイトジェネレータになる。

## なぜ Hugo なのか

静的サイトジェネレータはいくつもあるが、Hugo を選んだ理由は単純だった。

決め手は、単一バイナリで完結すること。Hugo は Go で書かれていて、実行に必要なのは `hugo` コマンドひとつだけだ。`node_modules` を作らないし、パッケージマネージャの都合でビルドが壊れることもない。実際このリポジトリには `package.json` すら存在しない。数年ぶりに触ったときに「まず依存を直す」という作業が発生しない構成は、個人サイトでは想像以上にありがたい。

ビルドが速いのも大きかった。手元でこのサイトを丸ごとビルドしても 1 秒かからず、プレビューサーバは変更を即座に反映する。書いている最中に待たされないというのは、書く量そのものに直結する。

そして、出力がただの HTML であること。生成物は静的ファイルの集まりなので、置き場所を選ばない。今は GitHub Pages に置いているが、気が変われば別のどこかに移せる。ロックインが弱いというのは、10 年単位で持つつもりなら大きい。

ひとつ前提がある。**Hugo には通常版と extended 版があり、このサイトは extended 版が必須**になっている。後述する Congo テーマが Sass のコンパイルを要求するためで、通常版の Hugo ではビルドが通らない。Homebrew で入る `hugo` は extended 版なので普段は意識しないが、CI では明示的に extended のバイナリを取りに行っている。

## なぜ Congo なのか

テーマは Congo を使っている。これは実際に使っている機能から逆算して説明したほうが早い。

`config/_default/params.toml` が、そのままテーマに何をさせているかの一覧になっている。

```toml
colorScheme = "ocean"
defaultAppearance = "dark"
enableSearch = true
enableCodeCopy = true

# User defined CSS
customCss = ["css/user.css"]

[header]
    logo = "images/logo.png"

[footer]
    showAppearanceSwitcher = true

[homepage]
    layout = "profile"

[article]
    showAuthor = false
    showBreadcrumbs = true
    showEdit = false
    editURL = "https://github.com/tocky/tocky.net/tree/main/content/"
    showReadingTime = false
    showTableOfContents = true
    showTaxonomies = true
    showComments = false

[list]
    showBreadcrumbs = true
    showTableOfContents = true
    showTaxonomies = true
```

上から順に、欲しかったものが並んでいる。

- `homepage.layout = "profile"` — トップページを記事一覧ではなくプロフィール表示にする。個人サイトとして、まず「誰が書いているか」を出したかった
- `enableSearch` — 全文検索。`hugo.toml` 側で `home` の出力に `JSON` を足してインデックスを吐かせている
- `enableCodeCopy` — コードブロックにコピーボタンが付く。技術記事を書く前提なら必須
- `showTableOfContents` — 目次。長い記事を書くつもりだったので
- `defaultAppearance = "dark"` と `showAppearanceSwitcher` — 既定はダーク、読者は切り替えられる
- `colorScheme = "ocean"` — 配色

そして多言語対応。Congo は言語ごとの設定ファイルとメニューを素直に分けられる。日英で持つつもりだったので、ここが破綻しないテーマであることは条件だった。

`showEdit = false` にしつつ `editURL` だけ書いてあるのは、いつでも「GitHub で編集」リンクを出せるようにしておくためで、いまは無効にしてある。

## なぜテーマを git submodule ではなく Hugo Modules で入れるのか

ここがこの記事で一番書きたかったところ。

Hugo のテーマ導入は、長らく `themes/` 配下に git submodule で置く方法が主流だった。ドキュメントもチュートリアルもだいたいそう書いてある。だがこのサイトは、最初から **Hugo Modules** でテーマを入れている。

Hugo Modules は Go のモジュールシステムをそのまま使う仕組みで、テーマを「依存ライブラリ」として扱う。設定は `config/_default/module.toml` の 2 行だけだ。

```toml
[[imports]]
path = "github.com/jpanther/congo/v2"
```

これだけ。バージョンは `go.mod` が持つ。

```
module github.com/tocky/tocky.net

go 1.23

require github.com/jpanther/congo/v2 v2.14.0 // indirect
```

そして `themes/` ディレクトリの中身はこうなっている。

```
themes/
└── .gitkeep
```

**テーマのファイルは 1 つもリポジトリに入っていない。** ディレクトリを空のまま残しているのは、Git が空ディレクトリを追跡しないための `.gitkeep` があるだけで、それ以外は何もない。実際このリポジトリには `.gitmodules` が一度も存在したことがない。

この方式には、submodule にはない利点がある。

**バージョンが明示的に固定される。** submodule でもコミットハッシュで固定はされるが、`go.mod` に `v2.14.0` と書いてあるほうが、何を使っているのか一目でわかる。`go.sum` がハッシュを持つので、取得したものが改変されていないことも検証される。

**クローンが素直。** submodule は `git clone --recursive` を忘れると空のディレクトリだけができて、ビルドが謎のエラーで落ちる。Hugo Modules ならビルド時に自動で取りに行くので、クローン手順に注意書きが要らない。

**更新が 2 コマンドで終わる。** テーマを最新にするのは以下だけ。

```sh
hugo mod get -u   # モジュールを最新に
hugo mod tidy     # go.mod / go.sum を整理
```

差分は `go.mod` と `go.sum` の数行にしか出ない。submodule のポインタ更新より、レビューしやすい変更になる。

一方で気をつける点もある。Hugo Modules は **Go のツールチェーンを要求する**。ローカルにも CI にも Go が必要で、このリポジトリでは Go 1.23 を使っている。Hugo だけあればいい、という手軽さは失われる。それでも、テーマをコードとして抱え込まないことのほうが、長い目で見て得だと判断した。

## なぜローカルオーバーライドを最小限にするのか

Hugo は、プロジェクト側の `layouts/` `assets/` `static/` に置いたファイルを、テーマの同じパスより優先して使う。テーマの一部だけを差し替えられる便利な仕組みで、実際このサイトでも使っている。

ただし、置きすぎると必ず詰まる。

上書きしたファイルは、テーマ側が更新されても古いまま残り続ける。テーマの変更に追従しないコピーが手元に溜まっていくと、バージョンを上げた瞬間にレイアウトが壊れる。そして壊れた原因が「自分が数年前にコピーしたファイル」であることには、まず気づけない。

実際このサイトでも、かつて `layouts/partials/head.html` にテーマのテンプレートを 118 行まるごとコピーして置いていた時期があった。少しだけ手を入れたくてコピーしたのだと思うが、テーマが更新されてもそこだけ取り残される。結局まるごと削除して、テーマ側の実装に戻した。

なので今は、上書きするのは本当に必要な 3 つだけにしている。

```
layouts/partials/comments.html   # Disqus を有効にする
assets/images/logo.png           # ヘッダーのロゴ
assets/images/author.jpg         # プロフィール画像
static/css/user.css              # 日本語まわりの調整
```

`comments.html` の中身は 1 行だ。

```go-html-template
{{ template "_internal/disqus.html" . }}
```

テーマのテンプレートを読んで理解して差し替えるのではなく、**テーマが用意した差し込み口に最小の記述を置く**。この距離感を守れているうちは、テーマの更新で困らない。

## なぜ config を分割するのか

Hugo は設定を 1 枚の `hugo.toml` に全部書いてもいいし、`config/_default/` 配下に分割してもいい。このサイトは分割している。

```
config/_default/
├── hugo.toml            # 出力・パーマリンク・言語宣言
├── params.toml          # Congo のテーマ設定
├── module.toml          # テーマの取得元
├── taxonomies.toml      # カテゴリ・タグの定義
├── languages.ja.toml    # 日本語のサイト情報・著者情報
├── languages.en.toml    # 英語版
├── menus.ja.toml        # 日本語のナビゲーション
└── menus.en.toml        # 英語版
```

分割してよかったのは、主に次の点だ。

ひとつは、**変更したいものがどこにあるか迷わない**こと。テーマの見た目をいじりたいなら `params.toml`、ナビを足したいなら `menus.*.toml` と決まっている。1 枚に全部あると、200 行のファイルを上から探すことになる。

もうひとつが本命で、**多言語のファイルが対になる**こと。`languages.ja.toml` と `languages.en.toml`、`menus.ja.toml` と `menus.en.toml` がそれぞれ同じ構造で並ぶので、片方だけ直して非対称になった事故に気づきやすい。1 枚のファイルの中で言語ごとのセクションが入り混じっていると、これが途端に難しくなる。

## なぜ GitHub Pages + Actions なのか

ホスティングは GitHub Pages、ビルドは GitHub Actions にしている。

いちばんは、費用がかからないこと。個人サイトの運用費を毎月払い続ける気になれなかった。10 年続けるつもりなら、固定費がゼロであることの価値は大きい。

運用ルールも単純で、`main` に入ったものがそのまま本番になる。ステージング環境はない。規則が 1 つしかないので、久しぶりに触っても「これは本番に出るのか」で迷わない。

PR ではビルドだけ走るようにしたのは、後から足した仕組みだ。`main` に入れる前に「少なくともビルドは通る」ことを確認できる。

---

ここまでが選定の理由。ここからは実装の中身に入る。

## ディレクトリ構成の全体像

リポジトリの全体はこうなっている。

```
tocky.net/
├── .devcontainer/
│   └── devcontainer.json     # Go 1.23 + Hugo extended 0.165.0
├── .github/workflows/
│   └── hugo.yml              # ビルドと GitHub Pages へのデプロイ
├── archetypes/
│   └── default.md            # hugo new のひな形
├── assets/images/            # ロゴ・プロフィール画像
├── config/_default/          # 分割した設定 (前述)
├── content/
│   ├── about/
│   │   ├── _index.ja.md
│   │   └── _index.en.md
│   └── posts/                # 記事
├── data/
├── layouts/partials/
│   └── comments.html         # 唯一のテンプレート上書き
├── static/
│   ├── css/user.css
│   └── favicon 各種
├── themes/
│   └── .gitkeep              # 空。テーマはここにない
├── go.mod                    # テーマのバージョンはここ
└── go.sum
```

見ての通り、**テーマの実体はどこにもない**。`go.mod` に書かれた `github.com/jpanther/congo/v2 v2.14.0` が、ビルド時に Go のモジュールキャッシュから読み込まれる。リポジトリをクローンして `hugo` を叩けば、必要なものは自動的に揃う。

## 設定ファイルの中身

### hugo.toml

サイト全体の骨格を決めている。

```toml
defaultContentLanguage = "ja"
disqusShortname = "tocky-net"

[outputs]
    home = ["HTML", "RSS", "JSON"]

[permalinks]
    posts = "/:year/:month/:slugorfilename/"

[languages]
    [languages.en]
        [languages.en.params]
            displayName = "🇺🇸"
            description = "Technology, software, development, cloud, and everything else I'm interested in"
    [languages.ja]
        [languages.ja.params]
            displayName = "🇯🇵"
            description = "テクノロジー、ソフトウェア、開発、クラウドなど私が興味のあることすべて"
```

`outputs` の `JSON` は検索インデックス用。Congo の全文検索はこれを読んでいるので、`enableSearch = true` にするならセットで必要になる。

`permalinks` は記事の URL 規則で、`/:year/:month/:slugorfilename/` にしている。この記事なら `/2026/08/blog-architecture/` になる。年月をパスに入れておくと、後で構成を変えたくなったときに衝突しにくい。

`[languages.*.params]` の 2 階層は、Hugo の仕様変更に追従した結果そうなっている。以前は言語ファイルの直下に `displayName` や `description` を書けたが、いまは `params` の下に置く。

### languages.ja.toml

```toml
locale = "ja"
label = "日本語"
title = "tocky.net"
copyright = "&copy; tocky.net"

[params.author]
    name = "Shin Tokiwa"
    image = "images/author.jpg"
    headline = "A developer who loves technology"
    bio = "..."
    links = [
        { github = "https://github.com/tocky" },
    ]
```

ここに、間違えやすい点が 2 つある。

ひとつは、著者情報を `[params.author]` の下に置くこと。トップレベルの `[author]` ではない。ここを間違えると Congo が警告を出し、バージョンによってはビルド自体が止まる。

もうひとつは、言語の指定に `locale` と `label` を使うこと。かつての `languageCode` / `languageName` は非推奨になっている。古い記事や古いテーマの例をそのままコピーすると、ビルド時に警告が出る。

### menus.ja.toml

```toml
[[main]]
    name = "Blog"
    pageRef = "posts"
    weight = 10

[[main]]
    name = "About"
    pageRef = "about"
    weight = 30

[[footer]]
    name = "Categories"
    pageRef = "categories"
    weight = 10

[[footer]]
    name = "Tags"
    pageRef = "tags"
    weight = 20
```

`url` ではなく `pageRef` で書いているのがポイントで、こうしておくと言語ごとのパス (`/posts/` と `/en/posts/`) を Hugo が解決してくれる。`weight` を 10 と 30 に飛ばしてあるのは、後から間に足せるようにするため。

### taxonomies.toml

```toml
category = "categories"
tag = "tags"
```

分類はカテゴリとタグの 2 つだけ。増やせるが、個人ブログで分類軸を増やしても運用しきれないので、この 2 つに留めている。

## コンテンツの持ち方

日本語が既定の言語で、`defaultContentLanguage = "ja"` によってルート (`/`) に出る。英語版は `/en/` の下に入る。

コンテンツファイルは **言語サフィックス**で言語を分ける。

```
content/about/_index.ja.md   → /about/
content/about/_index.en.md   → /en/about/
```

同じディレクトリに `.ja.md` と `.en.md` を並べると、Hugo がそれを同じページの翻訳同士だと認識し、言語切り替えのリンクを繋いでくれる。この記事のように片方の言語しかないページは、切り替えたときにトップに戻る挙動になる。

記事のひな形は `archetypes/default.md` にある。

```markdown
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
---
```

`hugo new posts/my-post.md` で、この形の下書きが生成される。既定が `draft: true` なので、書きかけが誤って公開されることはない。

## 日本語まわりの調整

`params.toml` の `customCss` で読み込んでいる `static/css/user.css` は、実質これだけだ。

```css
@import url('https://fonts.googleapis.com/css2?family=Ubuntu&display=swap');

html {
    font-family: "Ubuntu", "Helvetica Neue", "Helvetica", "Hiragino Sans", "Hiragino Kaku Gothic ProN", "Arial", "Yu Gothic Medium", "Meiryo", sans-serif;
    font-feature-settings: "palt";
}
```

やっていることは単純で、2 つだけ。

**欧文と和文でフォントを分ける。** 先頭に Ubuntu を置いて欧文に当て、その後ろに和文フォントを並べる。ブラウザは前から順に「その文字を持っているフォント」を探すので、欧文は Ubuntu、日本語は Hiragino や Yu Gothic という振り分けが自動的に成立する。

**`font-feature-settings: "palt"` で約物を詰める。** 日本語の句読点や括弧は、既定だと前後の余白が広すぎて間延びして見える。`palt` (プロポーショナルメトリクス) を有効にすると、文字ごとの適正な幅に詰まる。1 行入れるだけで日本語の見栄えがはっきり変わるので、和文を扱うサイトでは入れておいて損がない。

## GitHub Actions ワークフローの要点

`.github/workflows/hugo.yml` から、押さえておくべきところだけ。

### バージョンを全部固定する

```yaml
env:
  DART_SASS_VERSION: 1.102.0
  GO_VERSION: "1.23"
  HUGO_VERSION: 0.165.0
  TZ: Asia/Tokyo
```

Hugo も Go も Dart Sass も、`latest` を使わずバージョンを打っている。しかも `env` の 1 箇所に集約してあるので、上げるときはここだけ見ればいい。

そして**ローカル・devcontainer・CI の 3 つで Hugo のバージョンを揃えている**。`.devcontainer/devcontainer.json` も同じ 0.165.0 を指定している。手元では通るのに CI で落ちる、という一番つまらない事故がこれで消える。

### TZ を明示する

```yaml
TZ: Asia/Tokyo
```

これは日付ベースのパーマリンクを使っているサイトだと、地味に問題になる。GitHub Actions のランナーは既定で UTC なので、日本時間の朝 9 時より前に書いた記事は、CI 上では前日として扱われる。`/:year/:month/` を URL に含めている以上、これは URL が変わることを意味する。

### PR ではビルドだけ、デプロイは main だけ

```yaml
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
```

```yaml
deploy:
  if: github.event_name != 'pull_request'
```

PR でもビルドジョブは走るが、`deploy` ジョブは `pull_request` イベントでは実行されない。「マージ前にビルドが通ることは確認したい、でも公開はしたくない」を、条件 1 行で表現している。

### ビルドキャッシュを持ち越す

```yaml
- name: Build with Hugo
  run: |
    hugo build \
      --gc \
      --minify \
      --baseURL "${{ steps.pages.outputs.base_url }}/" \
      --cacheDir "${{ runner.temp }}/.cache/hugo"
```

`--cacheDir` を明示して `actions/cache` に載せている。画像処理の結果などが再利用されるので、記事が増えてからのビルド時間を左右する。`--gc` は使われなくなったキャッシュを掃除するオプション。

`--baseURL` に `steps.pages.outputs.base_url` を渡しているのは、GitHub Pages 側が持つ正しい公開 URL をビルドに教えるため。ここを間違えると、生成された HTML の中のリンクが全部おかしくなる。

## ハマりどころ

最後に、実際に引っかかったものを挙げておく。

### `.hugo_build.lock` で無限に待つ

これは今回この記事を書くにあたって実際に踏んだ。

Hugo はビルド中、プロジェクト直下に `.hugo_build.lock` というファイルを作って排他ロックを取る。通常は意識する必要がない仕組みだが、**何らかの理由で `hugo server` が初回ビルドの途中で固まると、ロックを掴んだままプロセスが残り続ける**。

この状態で別のビルドを走らせると、こうなる。

```
hugo: collected modules in 819 ms
Start building sites … 
hugo v0.165.0+extended+withdeploy darwin/arm64
```

ここから先に進まない。エラーも出ない。タイムアウトもしない。ただ黙って待ち続ける。

厄介なのは、原因が「いま動かしているコマンド」の外側にあることだ。何度実行し直しても同じところで止まるので、設定を疑ったりキャッシュを消したりと、見当違いの方向を延々と調べることになる。

正解はプロセスを見ることだった。

```sh
ps aux | grep hugo
```

数時間前に起動して固まったままの `hugo server` が残っていた。これを落としたら、次のビルドは 1 秒で終わった。

**ビルドが無言で止まったら、まず残存プロセスを疑う。** これを知っているだけで、次に踏んでもすぐ抜けられる。

### Congo は Hugo extended が必須

Congo が自分で宣言している。

```toml
[hugoVersion]
  extended = true
  min = "0.87.0"
```

通常版の Hugo ではビルドが通らない。Homebrew の `hugo` は extended 版なので普段は気づかないが、CI で自前でバイナリを取ってくる場合は、ファイル名が `hugo_extended_` で始まるものを選ぶ必要がある。

### テーマが宣言する最低バージョンを鵜呑みにしない

上の宣言をもう一度見てほしい。`min = "0.87.0"` と書いてある。

だが Congo v2.14.0 の CHANGELOG にはこうある。

> ⚠️ Required Hugo version is now 0.158.0 or later

**宣言されている値と、実際に必要なバージョンが食い違っている。** テーマの `module.toml` を見て「0.87.0 で動くのか」と判断すると、実際には動かない。

テーマを上げるときは、宣言よりも CHANGELOG を読んだほうがいい。とくに Congo の CHANGELOG は、破壊的変更に ⚠️ が付いていて追いやすい。v2.14.0 の項には、上の Hugo バージョン要件のほかに「著者設定の警告でサイトのビルドが止まる問題の修正」も並んでいて、前述した `[params.author]` の話がここに繋がっている。

## AI と一緒に組み直した

ひとつ書いておくと、この構成の刷新は AI と一緒に進めている。

依存の一括更新も、非推奨になった設定の移行も、この記事の下敷きになった調査も、Claude Code と組んでやった。git の履歴を見ると `Co-Authored-By: Claude` の付いたコミットがそのまま残っている。とくに「CHANGELOG を読んで、非推奨になった項目を洗い出して、順番に直す」といった、判断より調査量が支配的な作業は、任せると速い。

このあたりは主題が変わるので別の記事にする。仕事のほうでも AI をかなり実務に食い込ませているので、書くことは溜まっている。

## これから書くこと

そういうわけで、このブログはこういう構成で動いている。

次に書きたいのは、AI をどう実務に組み込んでいるかという話。それから、受託開発の会社を 18 年やってきて考えていること。技術と経営の両方から、興味を持ったことを書いていく。
