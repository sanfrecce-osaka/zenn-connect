---
title: "DSL により動的に定義されるメソッドの型シグネチャの導入"
---

# 3.1. この章でできるようになること

DSL により動的に定義されるメソッドに対しても型による補完が効くようになります。

# 3.2. 導入方法の選択

gem の型シグネチャは `rbs collection` でインストールできますが、これはあくまで静的なもののみです。 DSL 経由で動的に定義されるメソッドは別途対応が必要になります。
例えば以下のようものです。

```ruby
class User < ApplicationRecord
  scope :active, -> { where(active: true) } # <= scope により定義されるメソッドの型定義
end
```

これに対しては手段が 2つ あります。

- [ksss/orthoses-rails](https://github.com/ksss/orthoses-rails)
- [pocke/rbs_rails](https://github.com/pocke/rbs_rails)

どちらかを選択して導入してください。

rbs_rails を利用した事例が多いのですが、個人的には orthoses-rails を推しています。
orthoses-rails はサポートされている DSL が豊富で、基本的に orthoses-rails の提供する rake タスクのみの実行なので手数が少なくすみます。

参考: [Railsの型ファイル自動生成における課題と解決 by Yuki Kurihara - Kaigi on Rails 2023](https://kaigionrails.org/2023/talks/ksss/)

## 3.2.1. orthoses-rails での導入

以下を `Gemfile` に追加して `bundle install` を実行してください。

```ruby
group :development do
  # 省略
  gem 'orthoses-rails'
  # 省略
end
```

インストールできたら以下のコマンドを実行してください。

```zsh
$ bin/rails generate orthoses:install
```

すると `lib/tasks/orthoses/rails.rake` が生成されます。

```ruby
# frozen_string_literal: true

namespace :orthoses do
  task :rails do
    # Phase to load libraries
    require Rails.root / 'config/application'
    require 'orthoses/rails'

    # You can choose logger level
    Orthoses.logger.level = :warn

    # DSL for Orthoses.
    Orthoses::Builder.new do
      use Orthoses::CreateFileByName,
          to: 'sig/orthoses', # Write to this dir. (require)
          depth: 1,           # Group files by module name path depth. (default: nil)
          rmtree: true        # Remove all `to` dir before generation. (default: false)

      # Complement missing const name.
      use Orthoses::MissingName

      # You can use other publicly available middleware.
      # `Orthoses::YARD` is available at https://github.com/ksss/orthoses-yard.
      # By using this middleware, you can add the capability
      # to generate type information from YARD documentation.
      # use Orthoses::YARD,
      #   parse: ['{app,lib}/**/*.rb']

      # You can load hand written RBS.
      # use Orthoses::LoadRBS,
      #   paths: Dir.glob(Rails.root / "sig/hand-written/**/*.rbs")

      # Middleware package for rails application.
      use Orthoses::Rails::Application

      # Application code loaded here is the target of the analysis.
      run Orthoses::Rails::Application::Loader.new
    end.call
  end
end
```

orthoses の rake タスクを読む際に覚えておくとよいのは以下の点です。

- `Orthoses::Builder.new` に渡すブロックの中では `Orthoses::Builder#run` と `Orthoses::Builder#use` を使用できる
- `Orthoses::Builder.new` でブロックが評価され `use` で middleware の登録が行われる
- `Orthoses::Builder#call` が呼び出されるとブロックの中の `run` から遡って(== `use` の呼び出し順と逆順で)、登録されている middleware の `call` が実行されていく

それでは以下のコマンドを実行してください。

```zsh
$ bundle exec rails rails orthoses:rails
```

実行が完了すると `Orthoses::CreateFileByName` の `to` で指定した path の配下に型定義が生成されます。

## 3.2.2. rbs_rails での導入

TODO

# 3.3. Rails 以外の gem の DSL で定義されるメソッドの型を生成する

DSL は Rails 以外が提供しているものもありますが、これらの型シグネチャに関しては別途 gem を導入するか、自分で型を書く必要があります。例としては以下のようなものがあります。

- orthoses-**
  - [ksss/orthoses-paranoia](https://github.com/ksss/orthoses-paranoia)
  - [ksss/orthoses-config](https://github.com/ksss/orthoses-config)
  - [ksss/orthoses-fiddle](https://github.com/ksss/orthoses-fiddle)
- orthoses 以外
  - [tk0miya/rbs_config](https://github.com/tk0miya/rbs_config)
  - [tk0miya/rbs_draper](https://github.com/tk0miya/rbs_draper)
  - [tk0miya/rbs_discard](https://github.com/tk0miya/rbs_discard)
  - [tk0miya/rbs_shrine](https://github.com/tk0miya/rbs_shrine)
  - [tk0miya/rbs_active_hash](https://github.com/tk0miya/rbs_active_hash)
  - [tk0miya/rbs_devise](https://github.com/tk0miya/rbs_devise)

方針としては以下のようになるかと思います。

- orthoses-rails を利用した場合
  - orthoses-** がある場合はそれを利用する
  - ない場合は取り急ぎ orthoses 以外の gem を利用するか [ksss/orthoses](https://github.com/ksss/orthoses) を使って middleware を書く
- rbs_rails を利用した場合
  - orthoses 以外の gem を利用する
  - 型定義生成用の rake タスクを書く

gem を使う場合は各 gem の  README を参照してください。
ここでは自前で orthoses の middleware や rake タスクを書く場合の方法を説明します。

## 3.3.1. orthoses を使って middleware を書く

orthoses は rack アーキテクチャを元にしています。

参考: [RBS generation framework using Rack architecture - RubyKaigi 2022](https://rubykaigi.org/2022/presentations/_ksss_.html)

実装例としては [ksss/orthoses-paranoia](https://github.com/ksss/orthoses-paranoia/blob/v0.2.0/lib/orthoses/paranoia.rb#L6-L38) が参考になります。

```ruby
module Orthoses
  class Paranoia
    def initialize(loader)
      @loader = loader
    end

    def call
      acts_as_paranoid = CallTracer::Lazy.new # <= CallTracer::Lazy のインスタンスを生成する
      # 1. CallTracer::Lazy#trace の引数に DSL の文字列(クラスメソッドは `.`、インスタンスメソッドは `#` で表現する) とブロックを渡す
      # 2. ブロックの中で @loader.call を呼び出す
      # 3. 先に実行された他の middleware での型定義を格納した store(Hash) が返り値で返ってくるのでそれを受けとる
      store = acts_as_paranoid.trace("ActiveRecord::Base.acts_as_paranoid") do 
        @loader.call
      end

      # captures は CallTracer::Capture の配列で、CallTracer::Capture は method と argument を持つ Struct
      acts_as_paranoid.captures.each do |capture|
        # Utils.module_name(capture.method.receiver) で DSL を呼び出しているクラスが取得できる
        base_name = Utils.module_name(capture.method.receiver) or next
        paranoia_class_methods = "#{base_name}::ParanoiaMethods"
        next if store.key?(paranoia_class_methods)
        # header は module/class/interface の定義を渡せる
        store[paranoia_class_methods].header = "module #{paranoia_class_methods}"
        # store の key には型定義を追加する module/class の名前の文字列、<< の引数にはメソッドの型定義の文字列を渡す
        store[paranoia_class_methods] << "def with_deleted: () -> #{base_name}::ActiveRecord_Relation"
        store[paranoia_class_methods] << "def only_deleted: () -> #{base_name}::ActiveRecord_Relation"
        store[paranoia_class_methods] << "alias deleted only_deleted"
        store[paranoia_class_methods] << "def paranoia_scope: () -> #{base_name}::ActiveRecord_Relation"
        store[paranoia_class_methods] << "alias without_deleted paranoia_scope"

        store[base_name] << "extend #{paranoia_class_methods}"
        store["#{base_name}::ActiveRecord_Relation"] << "include #{paranoia_class_methods}"
        store["#{base_name}::ActiveRecord_Associations_CollectionProxy"] << "include #{paranoia_class_methods}"
      end

      store
    end
  end
end
```

定義した middleware は `lib/tasks/orthoses/rails.rake` の中で use してください。

```ruby
      use Orthoses::Paranoia # <= 追加

      # Middleware package for rails application.
      use Orthoses::Rails::Application

      # Application code loaded here is the target of the analysis.
      run Orthoses::Rails::Application::Loader.new
```

## 3.3.2. 型定義生成用の rake タスクを書く

TODO
