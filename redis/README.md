# Redis

## Commands

### Lists

- `RPUSH key value [value ...]`
    - O(1)
    - list の末尾に要素を追加
        - 要素は複数指定可能
- `RPUSHX key value`
    - O(1)
    - list の末尾に要素を追加
        - 要素は1つだけ指定可能
- `LPUSH key value [value ...]`
    - O(1)
    - list の先頭に要素を追加
        - 要素は複数指定可能
- `LPUSHX key value`
    - O(1)
    - list の先頭に要素を追加
        - 要素は1つだけ指定可能
- `LRANGE key start stop`
    - O(S+N)
        - S は list の開始位置の offset
        - N は list の長さ
    - start, stop を負の整数にした場合は、list の末尾から数える
        - -1 は list の末尾、-2 は list の末尾から 2 番目
- `RPOP key`
    - O(1)
    - list の末尾から値を取り出す
        - 取り出した値は list から消える
- `LPOP key`
    - O(1)
    - list の先頭から値を取り出す
        - 取り出した値は list から消える
- `RPOPLPUSH source destination`
    - O(1)
    - source の 末尾の値を取り出して destination の先頭に追加する
        - source から RPOP するので source の末尾の値は消える
- `LINDEX key index`
    - O(N)
        - N は list の長さ
            - ただし、list の先頭(0) と末尾(-1) は O(1) で取得可能
    - list の要素を返す
        - `LPOP`, `RPOP` と違い要素は消えない
- `LINSERT key BEFORE|AFTER pivot value`
    - O(N)
        - ただし、list の先頭への追加は O(1)
    - list の指定した要素の前後に追加する
- `LLEN key`
    - O(1)
    - list の長さを取得
- `LREM key count value`
    - O(N)
        - N は list の長さ
    - value を count 個削除する
        - 0 を 指定すればすべて削除する
- `LSET key index value`
    - O(N)
        - ただし、list の先頭と末尾に対する操作は O(1)
    - index の要素を value で上書きする
- `LTRIM key start stop`
    - O(N)
        - N は LTRIM で削除される要素数
    - start から stop で指定した要素以外を削除する
        - `LTRIM key 2 -1` とした場合、index 0, 1 の要素を削除
- `BLPOP key [key ...] timeout`
    - O(1)
    - ブロッキングする LPOP
        - 指定したキーが空の場合、キーに値が追加されるまでブロッキングする
            - キーが空でなければノンブロッキング
    - timeout を 0 にするとキーに値が追加されるまでブロッキングし続ける
        - timeout すると client にエラーが返る
    - トランザクションは使わないほうが良い
        - キューに溜めて即応答を返すので利点が損なわれるため
- `BRPOP key [key ...] timeout`
    - O(1)
    - ブロッキングする RPOP
        - 指定したキーが空の場合、キーに値が追加されるまでブロッキングする
            - キーが空でなければノンブロッキング
    - timeout を 0 にするとキーに値が追加されるまでブロッキングし続ける
        - timeout すると client にエラーが返る
    - トランザクションは使わないほうが良い
        - キューに溜めて即応答を返すので利点が損なわれるため
- `BRPOPLPUSH source destination timeout`
    - O(1)
    - ブロッキングする RPOPLPUSH
        - 指定したキーが空の場合、キーに値が追加されるまでブロッキングする
            - キーが空でなければノンブロッキング
    - timeout を 0 にするとキーに値が追加されるまでブロッキングし続ける
        - timeout すると client にエラーが返る
    - トランザクションは使わないほうが良い
        - キューに溜めて即応答を返すので利点が損なわれるため
