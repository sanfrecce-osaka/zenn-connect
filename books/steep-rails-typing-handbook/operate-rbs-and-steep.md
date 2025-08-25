---
title: "型を運用する"
---

# 6.1. rubocop

rbs 自体の rubocop は [ksss/rubocop-on-rbs](https://github.com/ksss/rubocop-on-rbs) があります。
Cop のドキュメント: https://github.com/ksss/rubocop-on-rbs/blob/main/docs/modules/ROOT/pages/cops.adoc

また、rbs-inline で `#:` 形式だといくつかの Cop のオプションを設定しておいたほうがよいです

- [Layout/LeadingCommentSpace](https://docs.rubocop.org/rubocop/cops_layout.html#allowrbsinlineannotation_-true-layoutleadingcommentspace)
- [Lint/SelfAssignment](https://docs.rubocop.org/rubocop/cops_lint.html#allowrbsinlineannotation_true-lintselfassignment)

# 6.2. `rbs collection` の更新について

現状は Gemfile や Gemfile.lock に gem が追加されても、追加された gem の型は自動で追加されず、手動で `bundle exec rbs collection install` を実行する必要があります。

# 6.3. エディタとの統合

TODO

# 6.4. CI に組み込む

TODO
