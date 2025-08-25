---
title: "周辺知識ついて学ぶ"
---

# 8.1. rbs/steep について学ぶ

- rbs
  - [ドキュメント](https://github.com/ruby/rbs/tree/master/docs)
  - まずは [Syntax](https://github.com/ruby/rbs/blob/master/docs/syntax.md) を読むと良いでしょう。
- steep
  - [ドキュメント](https://github.com/soutaro/steep?tab=readme-ov-file#docs)
- rbs-inline
  - [ドキュメント](https://github.com/soutaro/rbs-inline/wiki)
  - まずは [Syntax guide](https://github.com/soutaro/rbs-inline/wiki/Syntax-guide) を読むと良いでしょう。
- その他
  - [新機能ラッシュ！ RBS最新情報をキャッチアップ | gihyo.jp](https://gihyo.jp/article/2024/01/ruby3.3-rbs)

# 8.2. コミュニティで質問する

- [ruby-jp](https://ruby-jp.github.io/) の Slack Workspace
  - 型についての質問なら #type チャンネルがおすすめです。
- [Asakusabashi.rbs](https://asakusa-bashi-rbs.connpass.com/)
  - 型に興味を持つ人のための Rubyコミュニティ
  - 第一木曜と第三木曜の晩にオンラインで開催されています。

# 8.3. Ruby の他の型解析について学ぶ

Ruby の型解析を行う場合、rbs/steep 以外にも選択肢があります。
それらを触ることから「こういうの rbs/steep にもほしい！」というアイディアを得られたりするかもしれません。

## 8.3.1. rbi/sorbet/tapioca

- [rbi](https://github.com/Shopify/rbi)
  - rbs に当たる型シグネチャ
  - [ドキュメント](https://sorbet.org/docs/rbi)
- [sorbet](https://github.com/sorbet/sorbet)
  - steep に当たる型解析機
  - [ドキュメント](https://sorbet.org/docs/overview)
  - [Playground](https://sorbet.run/)
  - [tapioca](https://github.com/Shopify/tapioca)
    - 位置づけとしては orthoses が近いです
    - 参考記事
      - [Tapioca の DSL compiler のしくみ](https://qiita.com/tomoasleep/items/a9b8a7a7bbdab9bc7a44)
- [rbi-central](https://github.com/Shopify/rbi-central)
  - gem_rbs_collection に当たるもの

## 8.3.2. typeprof

- [リポジトリ](https://github.com/ruby/typeprof)
- [Playground](https://mame.github.io/typeprof-playground/)
- 資料・動画
  - [Good first issues of TypeProf - RubyKaigi 2024](https://rubykaigi.org/2024/presentations/mametter.html)
  - [Writing Ruby Scripts with TypeProf - RubyKaigi 2025](https://rubykaigi.org/2025/presentations/mametter.html)

# 8.4. 他の言語の型について学ぶ

他の言語の型を触ってみることで、より理解を深められたり「こういうの Ruby の型にもほしい！」というアイディアを得られたりするかもしれません。

- PHP
  - [zonuexe/phpstan-typing-tutorial](https://github.com/zonuexe/phpstan-typing-tutorial)
- TypeScript
  - [サバイバルTypeScript](https://typescriptbook.jp/)

# 8.5. 型システムについて学ぶ

- [型システムのしくみ TypeScriptで実装しながら学ぶ型とプログラミング言語](https://www.lambdanote.com/products/type-systems)
