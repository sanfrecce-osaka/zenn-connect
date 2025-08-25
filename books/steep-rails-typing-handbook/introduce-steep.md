---
title: "型検査の導入"
---

# 5.1. この章でできるようになること

steep gem を使って Rails アプリケーションに対して型検査ができるようになります。

# 5.2. steep の導入

`Gemfile` に以下を追加して `bundle install` を実行してください。

```ruby
group :development do
  # 省略
  gem 'steep'
  # 省略
end
```

次に以下を実行します。

```zsh
$ bundle steep init
```

すると以下の `Steepfile` が生成されます。

```ruby
# D = Steep::Diagnostic
#
# target :lib do
#   signature "sig"
#   ignore_signature "sig/test"
#
#   check "lib"                       # Directory name
#   check "path/to/source.rb"         # File name
#   check "app/models/**/*.rb"        # Glob
#   # ignore "lib/templates/*.rb"
#
#   # library "pathname"              # Standard libraries
#   # library "strong_json"           # Gems
#
#   # configure_code_diagnostics(D::Ruby.default)      # `default` diagnostics setting (applies by default)
#   # configure_code_diagnostics(D::Ruby.strict)       # `strict` diagnostics setting
#   # configure_code_diagnostics(D::Ruby.lenient)      # `lenient` diagnostics setting
#   # configure_code_diagnostics(D::Ruby.silent)       # `silent` diagnostics setting
#   # configure_code_diagnostics do |hash|             # You can setup everything yourself
#   #   hash[D::Ruby::NoMethod] = :information
#   # end
# end

# target :test do
#   unreferenced!                     # Skip type checking the `lib` code when types in `test` target is changed
#   signature "sig/test"              # Put RBS files for tests under `sig/test`
#   check "test"                      # Type check Ruby scripts under `test`
#
#   configure_code_diagnostics(D::Ruby.lenient)      # Weak type checking for test code
#
#   # library "pathname"              # Standard libraries
# end
```

`sig` には型シグネチャのファイルが格納場所を設定します。
`ignore_signature` は型検査に使う型シグネチャから除外する対象を指定できます。
`rbs-trace` で `save_files` を呼び出している場合、テストディレクトリ(e.g. `/spec`、`/test`) の型シグネチャも出力されているため、`ignore_signature` で除外しておくと良いです。
`check` で型検査の対象となるファイルやディレクトリを指定できます。
`ignore` は `check` の対象から除外するファイルやディレクトリを指定します。
`configure_code_diagnostics` で型検査のレベルを以下から選択できます。(指定しない場合は `default`)

- `default`
- `strict`
- `lenient`
- `silent`

また `configure_code_diagnostics` にブロックを渡すと型エラーの種類ごとにエラーのレベルを調整できます。指定できるレベルは以下の通りです。

- `:error`
- `:warning`
- `:information`
- `:hint`
- `nil`

最後に `library` ですが、`Gemfile` や `Gemfile.lock` に記載のない暗黙的にインストールされた gem がある場合のみ指定します。 以前は標準ライブラリや `Gemfile`・`Gemfile.lock` に記載されている gem をこの `library` で指定する必要がありましたが、現在は `gem_rbs_collection` ができたことにより不要になっています。

cf. https://github.com/soutaro/steep/blob/v1.10.0/guides/src/gem-rbs-collection/gem-rbs-collection.md#1-remove-unnecessary-library-configurations

特にこだわりがない場合は今回は以下のように設定してみましょう。(`rspec` を使っている場合)

```ruby
target :lib do
  signature "sig"
  ignore_signature "sig/trace/spec"
  check "app/models/**/*.rb"
end
````

# 5.3. 型定義の問題の解消

ここまでで一度以下を実行してみてください。

```zsh
$ bundle exec steep check 
```

大量のエラーが出て圧倒されるかと思いますが、1つずつ解消していきましょう。

steep の Diagnostic ID には大別して RBS と Ruby の2種類のカテゴリがあります。
前者は型定義のファイル自体に問題がある場合に、後者は型検査された Ruby のコードに問題がある場合に発生します。 後者はエラーが発生していてもエラーを抑制する手段があるのですが、前者は必ず解消しきらないと型検査をパスすることができません。

エラーを解消しては `bundle exec steep check` を実行するのサイクルを繰り返し、 RBS のエラーをゼロにすることをまずは目指しましょう。

RBS のエラーが 0 になったら 5.4. に進んでください。

ここから解消方法を紹介していきます。

## 5.3.1. ケース1: gem_rbs_collection でまだ型が提供されていない

**Diagnostic ID: RBS::UnknownTypeName**

```zsh
sig/generated/app/models/user.rbs:29:2: [error] Cannot find type `AASM`
│ Diagnostic ID: RBS::UnknownTypeName
│
└   include AASM
    ~~~~~~~~~~~~
```

未定義の定数の型がある場合は `orthoses` や `rbs-inline` が自動で定義してくれるのですが、gem 側はその範疇外になります。
例えば 2025/08/24 時点では以下のようなものがあげられます。

- `aasm`
- `pagy`
- `dry-monads`

これらは [ruby/gem_rbs_collection](https://github.com/ruby/gem_rbs_collection) にまだ型が定義されていないことによるものです。
まず `steep check` を通すところまでなら gem の全てのメソッドに型をつける必要はなく、最低限エラーが出ているモジュール・クラスの型を追加してください。
手書きの型定義を追加する場合、自動生成された型定義のあるディレクトリとはディレクトリを分けるのが通例です。 またディレクトリ名は `sig/handwritten/` とされることが多いです。
モジュール・クラスの型を定義する際は以下のいずれかのスタイルで定義してください。

```ruby
module Pagy
end

# ワンラインで定数名を書く場合は Pagy を先に定義しておく必要がある
module Pagy::Backend
end
```

```ruby
module Pagy
  module Backend
  end
end
```

余裕があれば [ruby/gem_rbs_collection](https://github.com/ruby/gem_rbs_collection) にコントリビュートしてもらえると助かります。

参考: [gem_rbs_collection へのコントリビュートから始める Ruby の型の世界](https://speakerdeck.com/sanfrecce_osaka/contributing-gem-rbs-collection)

## 5.3.2. ケース2: rbs-trace が出力した型に対する対応

### 5.3.2.1. プロダクションコードには存在しない型の修正

**Diagnostic ID: RBS::UnknownTypeName**

```zsh
sig/generated/app/models/user.rbs:29:18: [error] Cannot find type `RSpec::Mocks::InstanceVerifyingDouble`
│ Diagnostic ID: RBS::UnknownTypeName
│
└   def some_method: (RSpec::Mocks::InstanceVerifyingDouble) -> untyped
                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

#### 5.3.2.1.1. RSpec::Mocks

参考: https://github.com/sinsoku/rbs-trace/issues/23

`rbs-trace` はテストコードから型を生成するため、`rspec` を利用している場合は生成結果に `RSpec::Mocks` の型が含まれてしまいます。
これらを除外する必要があるので以下のように置換します。(上から順番に行ってください)

1. `/ \| RSpec::Mocks::[\w:]+/` => 削除 
2. `/Rspec::Mocks::[\w:]+ \| /` => 削除
3. `/Rspec::Mocks::[\w:]+/` => `untyped`

複数の型の Union になっている場合は削除し、単一の型になっている場合は `untyped` に置き換えていく、という流れです。 

#### 5.3.2.1.2. ActiveRecord_AssociationRelation

`ActiveRecord_AssociationRelation` という型は存在しないため、置き換える必要があります。
`rbs_rails`・`orthoses-rails` のどちらを使っている場合でも以下のように置換してください。

```diff
- ActiveRecord_AssociationRelation
+ ActiveRecord_Associations_CollectionProxy
```

### 5.3.2.2. 一部型への generics の追加

**Diagnostic ID: RBS::InvalidTypeApplication**

```zsh
sig/generated/app/models/user.rbs:29:18: [error] Type `::CSV::Table` is generic but used as a non generic type
│ Diagnostic ID: RBS::InvalidTypeApplication
│
└   def some_method: (CSV::Table) -> untyped
                      ~~~~~~~~~~
```

`rbs-trace` がジェネリクスに対応しているのは v0.6.0 時点では以下の 3つ のみです。

- `Array`
- `Range`
- `Hash`

これら以外のモジュール・クラスには手動でジェネリクスを付与する必要があります

参考: https://github.com/sinsoku/rbs-trace/issues/21

例えば以下のようなものがあります。

- `CSV::Table`
- `ActiveSupport::HashWithIndifferentAccess`

`rbs-inline` のコメント・型定義ファイルともに置換してください。

- `CSV::Table` => `CSV::Table[untyped]`
- `ActiveSupport::HashWithIndifferentAccess` => `ActiveSupport::HashWithIndifferentAccess[untyped, untyped]`

## 5.3.3. ケース3: 重複した型定義の削除

**Diagnostic ID: RBS::DuplicateDefinition**

```zsh
sig/generated/app/models/user.rbs:29:18: [error] Non-overloading method definition of `some_method` in `User` cannnot be duplicated
│ Diagnostic ID: RBS::DuplicateDefinition
│
└   def some_method: () -> String
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

型定義が重複しているため、重複した型定義を削除する必要があります。
これには `rbs subtract` を利用します。
`rbs subtract` に関しては以下を参照してください。

[Let's write RBS! - RubyKaigi 2023](https://rubykaigi.org/2023/presentations/p_ck_.html#day3)

`orthoses-rails` を利用している場合は以下を実行してください。

```zsh
$ bundle exec rbs subtract --write sig/generated sig/trace
$ bundle exec rbs subtract --write sig/orthoses sig/trace
$ bundle exec rbs subtract --write sig/trace sig/generated
```

# 5.4. `steep:ignore` コメントの追加

ここまで来たら RBS のエラーが 0 になっているはずです。(もしそれ以外にも RBS のエラーがあれば Pull Request や Issue で教えてもらえれば 🙏)
しかし、ここまで来ても `steep check` を実行すると 100 件を超える多くのエラーが残っているはずです。
これを 0 にするまで型検査が利用できないのは途方もない作業です。
また steep には rubocop の `rubocop:disable` コメントと同じように `steep:ignore` コメントが用意されていますが、これを手動で各所に埋め込んでいくのも大変です。
そこで `steep-expectations` オプションを利用します。
以下を実行してみてください。

```zsh
$ bundle exec steep check --steep-expectations
```

実行後に `steep_expectations.yml` が生成されるかと思います。
このファイルには `steep check` で発生しているエラーの種類と発生している場所が記録されています。
このファイルから `steep:ignore` コメントを挿入する箇所を特定することができそうです。
以下のスクリプトを `lib/commands/steep_command.rb` として追加してください。

```ruby
require 'rails/command'

module Commands
  class SteepCommand < Rails::Command::Base
    namespeace :steep
    
    def todo
      YAML.safe_load(File.open(Rails.root.join('steep_expectations.yml'))).map(&:deep_symbolize_keys).each_with_object(Hash.new { |hash, key| hash[key] = [] }) do |expectation, memo|
        expectation[:diagnostics].each do |diagnostic|
          diagnostic => { range: { start: { line: current_start_line }, end: { line: current_end_line } } }
          if memo[expectation[:file]].last in { start_line: last_start_line, end_line: last_end_line, **nil }
            case current_end_line
            # 同一の範囲
            in ^last_end_line
              # no op
            # 連続した行
            in ^(last_end_line + 1)
              memo[expectation[:file]].last[:end_line] = current_end_line
            # 含まれている行
            in ^(...last_end_line)
              # no op
            # 後続の連続していない行
            in ^((last_end_line + 2)..)
              memo[expectation[:file]].push(start_line: current_start_line, end_line: current_end_line)
            end
          else
            memo[expectation[:file]].push(start_line: current_start_line, end_line: current_end_line)
          end
        end
      end => ignore_ranges
      ignore_ranges.each do |file, ranges|
        ranges.each do |range|
          case range
          in { start_line:, end_line: ^start_line }
            range.merge!(comment: ["# steep:ignore\n"])
          else
            range.merge!(comment: ["# steep:ignore:start\n", "# steep:ignore:end\n"])
          end
        end

        written_lines = []
        File.open(file) do |f|
          lines = f.readlines
          is_writing = false
          lines.reverse.flat_map.with_index do |line, i|
            if is_writing
              range = ranges.find { |range| range[:start_line] == (lines.size - i) }
              if range
                case range
                in { comment: [start_comment, _] }
                  indent = line.match(/\A +/, 0) || ''
                  is_writing = false
                  [line, "#{indent}#{start_comment}"]
                end
              else
                line
              end
            else
              range = ranges.find { |range| range[:end_line] == lines.size - i }
              if range
                case range
                in { comment: [comment] }
                  if Prism.parse(line.chomp).comments.none?
                    ["#{line.chomp} #{comment}"]
                  else
                    indent = line.match(/\A +/, 0) || ''
                    ["#{indent}# steep:ignore:end\n", line, "#{indent}# steep:ignore:start\n"]
                  end
                in { comment: [_, end_comment] }
                  indent = line.match(/\A +/, 0) || ''
                  is_writing = true
                  ["#{indent}#{end_comment}", line]
                end
              else
                line
              end
            end
          end => results
          written_lines = results.reverse
        end
        File.open(file, 'w+') { |f| f.write(written_lines.join) }
      end
    end
  end
end
```

とりあえず動けばいいだろうという雑なスクリプトですが、確認したところでは大体うまく動いています。
が、ヒアドキュメント周辺でエラーがある場合はコマンド実行後に手動で修正する必要があります。
では以下を実行してください。

```zsh
$ bin/rails steep:todo
```

これで `steep:ignore` コメントが挿入されているはずです。
最後に以下を実行して型検査が green になっていれば導入完了です 🎉

```zsh
$ bundle exec steep check
```
