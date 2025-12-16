---
title: "一行パターンマッチを使って nil を消すテクニック"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Ruby', 'Tech']
published: true
---

## はじめに

これは Ruby/Rails Advent Calendar 2025 の 17日目 の記事です。

@[card](https://qiita.com/advent-calendar/2025/ruby)

北陸Ruby会議01 で [onk](https://x.com/onk/) さんと飲んでいた際に話して、意外と皆知らないテクニックなんだなーと感じたので記事にしてみました。
この記事では一行パターンマッチを使って、いつも書いているコードをお手軽により堅牢かつリーダブルにする手法を紹介します。

@[card](https://regional.rubykaigi.org/hokuriku01/)

パターンマッチの用語や文法については細かくは説明しないのでドキュメントを参照してください。

日本語

@[card](https://docs.ruby-lang.org/ja/3.4/doc/spec=2fpattern_matching.html)

英語

@[card](https://docs.ruby-lang.org/en/3.4/syntax/pattern_matching_rdoc.html)

一部 RBS を使っていますがこちらもドキュメントを参照してください。

@[card](https://github.com/ruby/rbs/blob/v3.9.5/docs/syntax.md)

## 一行パターンマッチを使った表明

引数やメソッドの返り値が、型としては `nil` や複数の型を返しうる場面はよくあるのではないでしょうか？
例えば認証に失敗していたり認証切れになっていたりした場合の `current_user` の返り値が `nil` になっている実装とかはありそうですね。(その前段で例外にしときなさいよ、という話はありますがあくまで例なのでご容赦を 🙏)
RBS で表現すると以下のような型ですね。

```ruby.rbs
# 複数の型
User | AdminUser
# nil を含む型
User? # User | nil と等価
```

このようなとき、表明を使うと便利です。
表明に関しては [t_wada](https://x.com/t_wada) さんのスライドや達人プログラマーをどうぞ。

@[speakerdeck](68b40ca9983a48dc9efec540ab98f88d?slide=69)
@[speakerdeck](68b40ca9983a48dc9efec540ab98f88d?slide=75)

表明は PHP では [assert](https://speakerdeck.com/twada/php-conference-2016?slide=70)、TypeScript では [assertion functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions) がありますが Ruby には表明用のメソッドや機能はありません。
しかし、一行パターンマッチを使うことでお手軽に表明を表現することができます。

```ruby
user => User
```

Ruby の一行パターンマッチは左辺が右辺にマッチしなければ `NoMatchingPatternError` が raise されます。
これにより `user` に `nil` が入ることは想定していないことをコードの読み手に伝えることができます。
表明を使っている箇所は例外が発生することは想定外の事態のはずで、Rails アプリケーションの場合ならエラーハンドリングはせず 500 でいいはずです。
このような場合に一行パターンマッチでサクッと表明を表現できるのは実に便利です。
一方、表明は起こり得ないケースで利用する手法なので、ライブラリを書く際の引数チェックのようなケース(ユーザーが引数として誤った型のオブジェクトを渡しうるケース) では表明ではなく `ArgumentError` や `TypeError`、カスタム例外クラス を raise する選択をするのが良いと思います。

@[speakerdeck](68b40ca9983a48dc9efec540ab98f88d?slide=76)

## 一行パターンマッチを使った Array#find・Array#first・Array#last・Array#[] 

`Array#find`・`Array#first`・`Array#last`・`Array#[]` などは要素が見つからなければ `nil` を返します。
Rails(ActiveRecord) でデータベースからレコードを取得する場合だと `!` つきのメソッドを使って値を取得することで `nil` になることがないようにするでしょう。

```ruby
user = User.find_by!(role: 'admin')
```

が、Ruby 自体や ActiveSupport には同じようなメソッドは用意されていません。
ここでも一行パターンマッチが役に立ちます。
まずは `Array#find` ですが、要素に対しての評価が `===` で十分なら以下のように書けます。

```ruby
# users の型は Array[User | AdminUser]
users => [*, AdminUser => admin_user, *]
pp admin_user # #<AdminUser id: 1>
```

単純な `===` での評価以上のことをしたい場合は `Proc` を使うことで実現できます。(`Proc#===` で `Proc` が評価される)

```ruby
users => [*, -> (user) { user.role == 'admin' } => admin_user, *]
```

また、上記は `admin?` のようなインターフェースが実装されていれば以下のようにピン演算子を使って書くこともできます。(`map(&:to_s)` のような書き方でおなじみの `Symbol#to_proc` ですね)

```ruby
users => [*, ^(:admin?.to_proc) => admin_user, *]
```

同じように `nil` を除外した `Array#first` や `Array#last` を表現することができます。 

```ruby
users => [first_user, *]
```

```ruby
users => [*, last_user]
```

他にも `nil` を除外した `Array#[]` も表現可能ですし、 

```ruby
users => [_, second_user, *]
```

それぞれを組み合わせて同時に変数に束縛したり型を明示したりもできます。

```ruby
users => [first_user, second_user, *other_users, AdminUser => last_user]
````

## 一行パターンマッチを使った Hash#[]

`Hash#[]` も一行パターンマッチで表現することで `nil` を除外できます。(key が `Symbol` の `Hash` の場合のみ)

```ruby
hash => { name: }
```

これくらいなら `Hash#fetch` を使うのと同じですが、一行パターンマッチなら複数の要素をまとめて変数に束縛できたり型を明示してコードの読み手のためにより多くの情報を残したりできます。

```ruby
hash => {
  name:,
  role: :admin | :member => role,
  age: 0..30,
  books: [*, { title: /[Rr]uby( on Rails)?/ => ruby_book_title }, *],
  **others
}
```

## 最後に

12/23 担当の [maimux2x](https://x.com/maimux2x) さんもパターンマッチの右代入について書かれるっぽいので楽しみですね。
パターンマッチはいいぞ。
