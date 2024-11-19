---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://www.jacobparis.com/content/cloudflare-remix/cover.png
# some information about your slides (markdown enabled)
title: RemixアプリをCloudflare PagesにデプロイしたらTTFBが3,800msになった
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true
fonts:
  # basically the text
  sans: 'Robot'
  # use with `font-serif` css class from windicss
  serif: 'Robot Slab'
  # for code blocks, inline code, etc.
  mono: 'Fira Code'
hideInToc: true
---

# RemixアプリをCloudflare PagesにデプロイしたらTTFBが3,800msになった

<!--
主催者である溝口さんからお声がけを頂き、少しだけではありますが発表、というか質問会のようなものをさせていただければと思います。

ということで、「」と題しまして、RemixとCloudflareで開発しながら現在進行中で困っていることを共有できたらなと思っています。

また解決策がわかっているわけではないのでなにか皆様のお力添えがあれば嬉しく思います。
-->

---
hideInToc: true
---

# アジェンダ

<Toc minDepth="1" maxDepth="2" />

<!--
まずは簡単ですがアジェンダになります。

今回のスライドは以前から触ってみたかった、MarkdownとVue.jsで書けるスライドを書くSlidevというものを使用しています。
-->

---

# 自己紹介

<div class="flex">
  <div class="w-[70%]">
    <ul>
      <li>
        若菜実農（さねあつ） <a href="https://github.com/saneatsu">@saneatsu</a>
        <li>Algoageという会社でフロントエンドエンジニアをしています</li>        
      </li>
      <li>
        <a href="https://chatboost.dmm.com/cv/">DMM チャットブーストCV</a>というLINEを使ったチャットボットを提供しています
        <li>会社ではNext.jsとReact Flowを使って社内向けプロダクトを開発しています</li>
      </li>      
    </ul>
  </div>
  <div class="w-[30%] ml-14 mt-10">
    <img 
      src="https://avatars.githubusercontent.com/u/16120550?s=400&v=4"
      alt="GitHubアイコン"
      width="180px"
      class="rounded-full"
    />
  </div>
</div>

<!--
最初に簡単な自己紹介をさせてください。

わかなさねあつ、といいます。
DMM傘下のAlgoageという会社でフロントエンドエンジニアをしています。
弊社では「DMMチャットブーストCV」というLINEを用いたチャットボットプロダクトを作っていて、中でも社内向けのプロダクトをNext.jsとReact Flowで作っています。

React Flowというのはノードベースのダイアログを構築できるライブラリです。

バックエンドに関しては、4年ほど前まではPythonなどを触っていたのですがもう完全に触っていません。
インフラを触る機会もあまりないという状態です。

がっつり自分一人でなにかやるのはCloudflareがはじめてくらいの感じです。
-->

---

# 対象となるアプリケーション

- 概要
  - 趣味で作っているキュレーションサイトみたいなもの
  - [ZennのScraps](https://zenn.dev/saneatsu?tab=scraps) に開発記録を書いています
- 技術スタック
  - Remix
  - Cloudflare Pages
  - shadcn/ui(Radix + Tailwind CSS)  
  - Prisma
  - Turso

<!--
アプリの技術スタックはこんな感じです。
RemixとCloudflareは完全に趣味で触っています。
アプリの概要はよくあるキュレーションサイトみたいな感じのものです。

アプリ自体は今年の2月ごろから空いている時間に少しずつ作り始めました。
Cloudflareも2月から完全に何もわからない状態から使い始めました。

Remixは主催の溝口さんが布教していてTypescriptで全部いけるのいいな、という点から。

Cloudflareを採用した理由は忘れましたが「RemixとCloudflareで銀の弾丸！」みたいな記事を見たのでしょう...。

ZennのScrapsに開発ログのようなものを書いています。
-->

---

# 発生している問題

- TTFBが3,800msかかっている
  - 推奨値は3G回線で600ms
- ページ内遷移の場合も訪れたことないページだと3,000msくらいかかってしまう  
  - 1回行ったことあるページの場合は早い  

<img 
  src="https://storage.googleapis.com/zenn-user-upload/10138bcf5a36-20241119.png"
  alt="Lighthouseの測定結果"
  width="700px"
/>

<!--
ここから本題に移ります。

問題は題名にもなっていた通り「どうもTTFBが遅いぞ」ということですね。

TTFBはTime to First Byteのことです、Webページを開こうとしてからダウンロードが始まるまで(最初の1バイトを受信するまで)の時間、つまりユーザーにとっての待ち時間です。
推奨とする値は0.6秒以内なので、3.8秒ということは6.5倍近くの時間がかかっていることになります。
3.8sかかったページではエンドポイントを3つほど叩いて60個ほどのデータを取得していました。
最初はなにかの立ち上げに時間がかかっているのかな？とも思っていたのですが、初めにアクセスしたページだけではなく、別のページに遷移する際にもかなり時間がかかっていました。

4秒近くかかってしまうと、体感「あれ？クリックしたよな？」となるくらいには遅いです。

半年くらい開発しておいて今更？？と思うかもしれません。
Cloudflare Pagesにデプロイするとどうもページの描画速度が遅いなとは思っていたのですが目をつむりつつ機能開発をし続けてしまっていました。
-->

---

# 計測用ページの作成

ゼロからRemix x Cloudflare Pagesのアプリを作って、以下3つのページで計測する。

リポジトリ: [saneatsu/remix-on-cf-pages](https://github.com/saneatsu/remix-on-cf-pages)

URL: https://remix-on-cf-pages.pages.dev/

<div class="mt-10"></div>

## 計測用のページ

1. [JSON Placeholder](https://jsonplaceholder.typicode.com/todos)を使ってTodoを200件取得するページ
1. [JSON Placeholder](https://jsonplaceholder.typicode.com/todos)を使ってTodoを1,000件取得するページ
1. DB(Turso)の2つのテーブルから5件ずつ、計10件のデータを取得するページ

<!-- ## 結果（Cloudflare Worker） -->
<!-- Lighthouseによる測定結果 -->

<!-- | URL    | TTFB    | -->
<!-- | --- | --- | -->
<!-- | `/json-placeholder-200`| 390 - 400 ms | -->
<!-- | `/json-placeholder-1000` | 510 - 520 ms | -->
<!-- | `/db-connect` | 1800 - 2330 ms | -->

---

# 結果
Lighthouseによる計測結果（各ページ3回測定）。

|  URL   | TTFB    |
| --- | --- |
| `/json-placeholder-200`| 240 - 430 ms |
| `/json-placeholder-1000` | 500 - 600 ms |
| `/db-connect` | **<span style="color:red">1970 - 2310 ms</span>** |

<!--
結果はこのようになりました。

基本的にCloudflare Pagesで時間がかかり、その中でもTursoにアクセスしているページでかなり時間がかかりました。

`/db-connect` では2000ms前後かかっています。
ここでは2つのテーブルから5件ずつ、合計10件取得していますが、1つのテーブルから5件だけ取得した場合では1200msくらいかかっていました。対象となるテーブルが増えるたびに800 - 1000msくらい増えていそうです。
-->

---

# 考えられる原因

- Cloudflareの設定？
  - 初回のアクセスだからCloudflareのキャッシュの設定ではないはず
- TursoのCold start？
  - Tursoは無料プランだとCold startが発生する
  - ということで月9ドルの有料プランにしてみたが計測結果が変わらなかった

<!--
初回の描画だからキャッシュの設定ではないはず。
-->

---

# 結論（追記）

- 結論、原因はORMとしてPrismaを使用していることで、Drizzleに移行することで問題を回避できました
- 詳細は以下の記事にまとめました
  - [Cloudflare Pages(Workers)でPrismaを使うとサーバーからデータを取得するのに時間がかかってしまう | Zenn](https://zenn.dev/saneatsu/articles/remix-cloudflare-ttfb-improvement)
