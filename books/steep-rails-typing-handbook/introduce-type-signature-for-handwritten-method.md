---
title: "自前で定義したメソッドの型シグネチャの導入"
---

# 4.1. この章でできるようになること

自前で定義したメソッドに対しても型による補完が効くようになります。

# 4.2. rbs-trace と rbs-inline の導入

既存の Rails アプリケーションに型を導入する場合、既に様々なメソッドが定義されていてそれらにも型をつける必要がありますが、それらに手で型をつけていくのは非常に大変です。

ここで役に立つのが [sinsoku/rbs-trace](https://github.com/sinsoku/rbs-trace) です。rbs-trace はテストコードから型を生成するための gem です。

参考: [Automatically generating types by running tests - RubyKaigi 2025](https://rubykaigi.org/2025/presentations/sinsoku_listy.html)

また、一緒に [soutaro/rbs-inline](https://github.com/soutaro/rbs-inline) も導入する必要があります。rbs-inline はコメント形式で rbs の型をソースコード上に書けるようにする gem です。

参考: [Embedding it into Ruby code - RubyKaigi 2024](https://rubykaigi.org/2024/presentations/soutaro.html)

以下を `Gemfile` に追加して `bundle install` を実行してください。

```ruby
group :development, :test do
  # 省略
  gem 'rbs-inline', require: false
  # 省略
end

group :test do
  # 省略
  gem 'rbs-trace'
  # 省略
end
```

次に rbs-trace の設定を `spec/rails_helper.rb` か `spec/support/rbs_trace.rb` に追加します。 自分は以下のように設定しています。(rbs-trace の README に tips 等が書いてあるので好きなように設定してください) 

```ruby
RSpec.configure do |config|
  # 省略

  # 環境変数を設定したときのみ rbs-trace を動かす
  if ENV.fetch('TRACE', '') in /\A.+/ => path_glob_or_flag
    # 環境変数 TRACE に glob を渡したときは glob に該当するファイルのみを対象に型を生成する
    options = (path_glob_or_flag in %r!\A/.+! => path_glob) ? { paths: Dir.glob("#{Dir.pwd}#{path_glob}") } : {}
    trace = RBS::Trace.new(**options) 

    config.before(:suite) { trace.enable }
    config.after(:suite) do
      trace.disable
      # :rbs_colon を save_comments に渡すことで TypeProf 等と記法をあわせる
      # 手で rbs-inline のコメント書くときは @rbs 記法を使うことで生成されたものと手で書いたものを分けられるメリットもある
      trace.save_comments(:rbs_colon)
      # .rbs を out_dir で指定した path 配下に出力する
      trace.save_files(out_dir: 'sig/trace/')
    end
  end
end
```

最後にテストを実行すると rbs-inline のコメントと rbs ファイルがガッと生成されます。

```zsh
$ bundle exec rspec spec
```

## 4.2.1. `rbs-inline` についての注意

rbs-inline は現在 rbs への統合が進められており、積極的に更新はされていません。まだ rbs-inline が統合された rbs はリリースされていないため、リリースされるまでは rbs-inline を使ってください。rbs の v4.0.0.dev.x で開発中の rbs を確認できます。
また、rbs-inline の prism のバージョンの制限により最新の rubocop だとバージョンが解決できません。rbs に統合されるまでは rbs-inline を fork して使うのが良いかもしれません。

参考: https://github.com/soutaro/rbs-inline/pull/207
参考: https://github.com/soutaro/rbs-inline/pull/199

# 4.3. YARD 資産を rbs に転用する

もし YARD の資産がある場合はそれを rbs に移行したいと考えるかもしれません。
その場合は以下が参考になります。

- [`ksss/orthoses-yard`](https://github.com/ksss/orthoses-yard)
  - [Railsの型ファイル自動生成における課題と解決](https://kaigionrails.org/2023/talks/ksss/)
- [前編：YARD から rbs-inline に移行しました - Timee Product Team Blog](https://tech.timee.co.jp/entry/2024/08/22/183127)
- [後編：YARD から rbs-inline に移行しました - Timee Product Team Blog](https://tech.timee.co.jp/entry/2024/08/23/180201)
- [AIを使ってYARDからrbs-inlineへ移行しました - kickflow Tech Blog](https://tech.kickflow.co.jp/entry/rbs-inline-with-ai)

# 4.4. rbs-inline での出力

最後に rbs-inline で型を出力します。
まずは `app` です。

```zsh
$ bundle exec rbs-inline --output app --opt-out
```

`sig/generated` に型定義が出力されます。出力先は `app` が除外されるため、`sig/generated` 配下に `app` ディレクトリを作成し、`sig/generated` 配下に生成された rbs ファイルとディレクトリを移動してください。
他のディレクトリも必要あれば生成し、同じように `sig/generated` 配下にディレクトリを切って移動させてください

```zsh
$ bundle exec rbs-inline --output lib --opt-out
```
