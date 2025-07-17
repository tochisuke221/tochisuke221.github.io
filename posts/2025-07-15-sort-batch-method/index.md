# [tochisuke docs](https://tochisuke221.github.io/)

## ソート順序を意識したバッチ処理

### 背景

find_eachのようにメモリ効率を意識しつつも、特定カラムでソートして処理を行いたいときがある。
Rails には find_each、find_in_batches、in_batches といったメモリ効率を意識したバッチ処理メソッドが用意されていますが、いずれもスコープで指定したorderを無視して、デフォルトでは主キー（primary_key）の昇順（または :order オプションで指定した cursor の順序）でしかバッチ処理を行わないという仕様になっている。

しかし、Rails8.0.2よりソート順序を維持した上でソートできるようになった。
内部実装を追いながら、どういう実装であるかをまとめておく。

### find_eachのcursorオプション
find_eachのcursorオプションはこのPRで実装された
https://github.com/rails/rails/pull/52384



### find_eachの内部実装

`find_each`メソッドの内部実装の根幹を担うのは`in_batches`メソッドである。

`in_batches`メソッドは下記の通りである

```ruby
    def in_batches(of: 1000, start: nil, finish: nil, load: false, error_on_ignore: nil, cursor: primary_key, order: DEFAULT_ORDER, use_ranges: nil, &block)
      cursor = Array(cursor).map(&:to_s)
      ensure_valid_options_for_batching!(cursor, start, finish, order)

      if arel.orders.present?
        act_on_ignored_order(error_on_ignore)
      end

      unless block
        return BatchEnumerator.new(of: of, start: start, finish: finish, relation: self, cursor: cursor, order: order, use_ranges: use_ranges)
      end

      batch_limit = of

      if limit_value
        remaining   = limit_value
        batch_limit = remaining if remaining < batch_limit
      end

      if self.loaded?
        batch_on_loaded_relation(
          relation: self,
          start: start,
          finish: finish,
          cursor: cursor,
          order: order,
          batch_limit: batch_limit,
          &block
        )
      else
        batch_on_unloaded_relation(
          relation: self,
          start: start,
          finish: finish,
          load: load,
          cursor: cursor,
          order: order,
          use_ranges: use_ranges,
          remaining: remaining,
          batch_limit: batch_limit,
          &block
        )
      end
    end
```
[該当箇所: Github](https://github.com/rails/rails/blob/60252302c77fb2a4885996a31b347b0db6aee6e1/activerecord/lib/active_record/relation/batches.rb#L259)

実際に分割取得している処理は、`batch_on_unloaded_relation`メソッドでしているようなので確認する。

```ruby

# 注意！！一部省略して記載
def batch_on_unloaded_relation(relation:, cursor:, order:, batch_limit:)
  batch_orders = cursor.zip(order) # 例: [[:created_at, :asc], [:id, :asc]]
  order_hash = batch_orders.to_h   

  relation = relation.reorder(order_hash).limit(batch_limit)
  relation.skip_query_cache!

  loop do
    values = relation.pluck(*cursor)
    values_size = values.size
    values_last = values.last

    break if values_size == 0

    # yield対象のrelationを構築して返す
    yielded_relation = relation.klass.where(cursor => values).order(order_hash)
    yield yielded_relation

    break if values_size < batch_limit
    break if values_last.nil? || values_last.any?(&:nil?) # nil安全

    # 次のバッチ用の条件を構築
    operators = order.map { |o| o == :desc ? :lt : :gt } # 最後のカーソルに対応
    relation = apply_cursor_condition(relation, cursor, values_last, operators)
  end
end

```

[該当箇所: Github](https://github.com/rails/rails/blob/60252302c77fb2a4885996a31b347b0db6aee6e1/activerecord/lib/active_record/relation/batches.rb#L426)

下記の部分に着目すると、シーク法を用いた分割取得をしていることがわかる

```ruby
  batch_orders = cursor.zip(order) # 例: [[:created_at, :asc], [:id, :asc]]
  order_hash = batch_orders.to_h   

  relation = relation.reorder(order_hash).limit(batch_limit)

  loop do
    # 処理
  end
```

流れとしては下記の通り
1. cursorに指定した列で受け取ったrelationをソートしなおす。
  - `relation = relation.reorder(order_hash).limit(batch_limit)`
2. loop内の処理で下記を行う
  - 3パターンのバッチ取得方法に分岐する（`load`／`use_ranges`／通常モード）  
    - **`load`がtrueのとき**  
      - relation.records からレコード配列を直接取得（すでにロード済み）  
      - `pluck(*cursor)` でカーソル列の値を取得し、最後の値（`values_last`）を記録  
      - `where(cursor => values)` でバッチに該当するrelationを構築し、すでに取得済みのrecordsを紐付ける

    - **use_rangesモードが有効なとき**  
      - `offset(batch_limit - 1).pick(*cursor)` で「バッチ末尾のカーソル値」を効率よく取得  
      - 最後の値が取れなければ `pluck(*cursor)` にフォールバックして再評価  
      - `apply_finish_limit` により、カーソル境界で切ったrelationを構築して yield する

    - **それ以外（通常モード）のとき**  
      - `pluck(*cursor)` でカーソル列だけ取得  
      - 最後の値（`values_last`）を使って、同様にバッチ対象のrelationを構築

3. 終了条件のチェック
  - `values_size == 0` なら終了（レコードが残っていない）
  - `values_last` に `nil` を含む場合は、select句に必要なカラムが不足しているため例外を発生
  - `yield yielded_relation` により、処理対象のバッチをブロックに渡す
  - バッチサイズが `batch_limit` より小さい場合は、次のバッチが存在しないため終了

4. limit_value が存在する場合の制御
  - `remaining`（残り処理可能件数）を減算
  - 0件なら終了、`remaining < batch_limit` なら次のクエリの `limit` を縮める

5. 次バッチのためのカーソル条件を構築
  - `batch_orders` を複製し、最後のカラムの順序に応じて `<` または `>` を決定
  - それ以前のカラムについては `<=` または `>=` を使用（複合条件）
  - 例：`(a > 1) OR (a = 1 AND b > 10)` のような形式
  - `batch_condition(relation, cursor, values_last, operators)` を呼び出して次の relation を更新

6. 次のループに進み、同様の処理を繰り返す



## 使用上の注意
一般にシーク法においてcursorに指定するカラムはユニークな列である必要があるが、複数カラムが一意であることが担保できていれば、問題なさそう

## シーク法について
シーク法についてはこの辺の資料がわかりやすかった
- [https://shopify.engineering/pagination-relative-cursors](https://shopify.engineering/pagination-relative-cursors)
- [https://www.slideshare.net/slideshow/p2d2-pagination-done-the-postgresql-way/22210863#8](https://www.slideshare.net/slideshow/p2d2-pagination-done-the-postgresql-way/22210863#8)
- [https://docs.gitlab.com/development/database/pagination_guidelines/](https://docs.gitlab.com/development/database/pagination_guidelines/)
