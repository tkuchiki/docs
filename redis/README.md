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
- `RPOPLPUSH srclistkey destlistkey`
    - O(1)
    - srclistkey の 末尾の値を取り出して destlistkey の先頭に追加する
        - srclistkey から RPOP するので srclistkey の末尾の値は消える
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
