# [tochisuke docs](https://tochisuke221.github.io/)

## カスタム例外クラスを作る上での注意点

結論、「StandardErrorを直接継承してはいけない」

## なぜか？

RubyやRailsでは、下記のように `StandardError` を直接継承したカスタム例外をよく見る。

```ruby
# よくあるパターン（悪くはないが…）
class ZipcloudAPIError < StandardError; end
```

この実装自体は“動く”のですが、ライブラリ化・再利用・保守性 を考えると不十分。

上記のように直接、StandardErrorを継承した場合、下記のような問題点がある。


- 例外の識別ができない
  - どのライブラリから発生したエラーか判別できず、他の StandardError と混ざってしまう。

- 拡張性がない
  - 今後エラーを増やしたくなっても、共通の親クラスがないため整理できない。

- 利用者が正しく rescue できない
  - rescue MyLib::Error のような安全な補足ができず、意図しない例外まで握りつぶす危険がある。

- 他ライブラリとの衝突リスク
  - 例外名が衝突したり、Rails 内部の例外と見分けがつかなくなる。


## 対策
ライブラリでは、必ず 独自の親クラス（エラールート） を作り、すべての例外をそこから継承させるべき。
つまりStandardErrorを直接継承しない。

（エラーに名前空間を与えるアイデア）

```ruby
# lib/my_lib/error.rb
module MyLib
  class Error < StandardError; end
end

# lib/my_lib/errors/connection_error.rb
module MyLib
  class ConnectionError < Error; end
end

raise MyLib::ConnectionError, "API unreachable"
```

```ruby
begin
  client.fetch
rescue MyLib::Error => e # <= ここをStandardErrorとすると、全く別の例外を捕捉する可能性がある
  logger.error("MyLib error: #{e.message}")
end
```
