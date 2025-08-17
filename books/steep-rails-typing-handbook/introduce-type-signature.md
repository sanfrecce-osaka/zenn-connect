---
title: "静的に定義される型シグネチャの導入"
---

# 2.1. この章でできるようになること

`irb` や `rails console`、エディタでの入力補完をより強力にすることができます。

# 2.2. rbs の導入

以下を `Gemfile` に追加して `bundle install` を実行してください。

```ruby
gem 'rbs'
```

次に `rbs collection` を導入し、利用している gem の型を [ruby/gem_rbs_collection](https://github.com/ruby/gem_rbs_collection) からインストールします。gem_rbs_collection については [Community-driven RBS repository - RubyKaigi 2024](https://rubykaigi.org/2024/presentations/p_ck_.html) を参照してください。

それでは以下のコマンドを実行してください。

```zsh
$ bundle exec rbs collection init
```

すると `rbs_collection.yml` が以下のように生成されます。

```yaml
# Download sources
sources:
  - type: git
    name: ruby/gem_rbs_collection
    remote: https://github.com/ruby/gem_rbs_collection.git
    revision: main
    repo_dir: gems

# You can specify local directories as sources also.
# - type: local
#   path: path/to/your/local/repository

# A directory to install the downloaded RBSs
path: .gem_rbs_collection

# gems:
#   # If you want to avoid installing rbs files for gems, you can specify them here.
#   - name: GEM_NAME
#     ignore: true
```

設定に関しては[ドキュメント](https://github.com/ruby/rbs/blob/master/docs/collection.md)を参照してください。
ここではそのまま以下のコマンドを実行してください。

```zsh
$ bundle exec rbs collection install
```

`.gem_rbs_collection` 配下に利用している gem の型シグネチャがインストールされます。
`.gem_rbs_collection` 配下はコミットする必要がないので `.gitignore` に追加しておきましょう。

# 2.3. repl_type_completor の導入

`irb` や `rails console` での入力補完を強化するために、`repl_type_completor` を導入します。
Gemfile に以下を追加して `bundle install` を実行してください。

```ruby
gem 'repl_type_completor'
```

`.irbrc` に以下を追加しておくか

```ruby
IRB.conf[:COMPLETOR] = :type
```

環境変数 `IRB_COMPLETOR` に `type` を設定しておくことで `rails console` の入力補完でも型が利用されます。

```zsh
$ IRB_COMPLETOR=type bin/rails console
```

参考: [IRBのアップデート  〜補完とデバッグ機能の強化 | gihyo.jp](https://gihyo.jp/article/2024/01/ruby3.3-irb)
