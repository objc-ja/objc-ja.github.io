# objc.io 日本語訳

## 概要

[objc.io](http://www.objc.io)の翻訳を公開しています。[こちら](http://objc-ja.github.io)から翻訳記事を閲覧できます。

## 記事の書き方

GitHub Pages、Jekyll、Jekyll Bootstrapを利用しています。記事を書くにはまずJekyllをインストールしてください。

```
$ gem install jekyll
```

### 公開前のドラフト

`_drafts`ディレクトリの下に`article-title.md`のようなファイル名でファイルを作ります。`article-title`の部分が記事のURLになります。記事の中身は以下のテンプレートにそって記述してください。

```
---
layout: post
title: "軽量なView Controller"
category: issue-1
original:
  title: "Lighter View Controllers"
  url: http://www.objc.io/issue-1/lighter-view-controllers.html
  author: "Chris Eidhof"
translator:
  name: "@gonsee"
  url: http://twitter.com/gonsee
---
{% include JB/setup %}

以下、記事の中身
```

上記ファイルを用意して、リポジトリのルートで

```
$ jekyll serve --drafts
```

とすると、[http://localhost:4000](http://localhost:4000)でサイトの見え方を確認できます。修正を反映するには再度`jekyll serve`するか、`--watch`オプションをつけて自動更新を有効にしておきます。

記事を書いたらcommitしてpushします。ドラフトの状態では記事は表に公開されません。この状態でレビューしましょう。

### 記事の公開

レビュー後の記事は`_posts`ディレクトリの下に`YYYY-MM-DD-article-title.md`のようなファイル名にリネームして置きます。日付部分が記事の投稿日になります。

```
$ jekyll serve
```

して、[http://localhost:4000](http://localhost:4000)にアクセスすれば公開後の状態を確認できます。問題なければpushしてください。記事が公開されます。