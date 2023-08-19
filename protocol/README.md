# Scarlet Search Protocol

## 0. 説明
Scarlet Search Protocolは、分散型検索エンジンのためのプロトコルです。
httpをベースにしています。

大企業の検索エンジンによるWebの支配から逃れるために作られます。

## 1. サーバーの処理
### basePath
Scarlet Search Protocolを実装したサーバーには、`basePath`が存在します。
これは、Scarlet Search Protocol が動作するURLのベースパスを示します。

例、サーバーのドメインは、`example.com`とします:
- `basePath: '/'`: `http(s)://example.com/*`で動作する
- `basePath: '/scarlet'`: `http(s)://example.com/scarlet/*`で動作する
### aboutページ
インスタンスの情報を取得するページです。`{basePath}/about`に存在します。

以下の条件に従うレスポンスをする必要があります。
- `Content-Type` は `application/json`であり、文字コードはUTF-8であること
- JSONでレスポンスすること
- `Access-Control-Allow-Origin`のヘッダーは`*`であること

JSONデータは、以下のようになTypeScriptインターフェイスで表せます。
```typescript
interface Data {
  basePath: string
  instanceId: string
}
```
- `basePath`は、前述したbasePathです。
- `instanceId`は、インスタンスのIDです。`{ドメイン名}{basePath}`になります。最後の`/`は省略できます。
  - 例:
    - `example.com`がサーバーのドメインで、basePathが`/scarlet`の場合は、`example.com/scarlet`になります。
    - `example.com`がサーバーのドメインで、basePathが`/`の場合は、`example.com`になります。(最後の`/`は省略できるため)
### get-instancesページ
そのインスタンスが知っているインスタンスたちを返します。`{basePath}/get-instances`に存在します。

以下の条件に従うレスポンスをする必要があります。
- `Content-Type` は `application/json`であり、文字コードはUTF-8であること
- JSONでレスポンスすること
- `Access-Control-Allow-Origin`のヘッダーは`*`であること

JSONデータは、以下のようになTypeScriptインターフェイスで表せます。
```typescript
interface Data {
  instances: {
    instanceId: string
  }[]
}
```
- `instances`: そのインスタンスが知っているインスタンスたちの配列です。何個でも構いません。
  - `instanceId`: 知っている一つのインスタンスのIDです。
### 検索
検索結果を取得するには、`{basePath}/search`のパスに送信します。
以下の条件に従う必要があります。
- `POST`メソッドであること
- `Content-Type` は `application/json`であり、文字コードはUTF-8であること
- `Access-Control-Allow-Origin`のヘッダーは`*`であること
- bodyは下のJSONデータに従うこと

JSONデータは、以下のようになTypeScriptインターフェイスで表せます。
```typescript
interface Data {
  query: string
  language: string | null
  safe: 0 | 1 | 2
}
```
- `query`は、検索したい文字列を代入します。一般的な検索エンジンのボックスに入れる内容です。
- `language`は、ユーザーの言語を示します。ISO 639-1で定義された言語コード(`ja`など)、またはISO 3166で定義された国コードをハイフンで繋げたもの(`ja-jp`)が代入されます。
不明または言語を指定しない場合は、`null`が入ります。
- `safe`は、セーフサーチの強さです。
  - 0: オフ
  - 1: 標準
  - 2: 厳格
  と示します。

以上の3つのデータをどう処理するかは、インスタンスに委ねられます。

リクエストをクライアントから正しい形式で送信すると、サーバー側から検索結果が返ってきます。

検索結果のレスポンスは以下の条件でなければいけません。
- `Content-Type` は `application/json`であり、文字コードはUTF-8であること
- bodyは下のJSONデータに従うこと

JSONデータは、以下のようになTypeScriptインターフェイスで表せます。
```typescript
interface Data {
  result: {
    score: number
    title: string
    iconUrl: string
    description: string
    url: string
    thumbnail: string | null
  }[]
}
```
- `result`には、結果の配列が入ります。一つの結果につき一つです。いくつ結果を返しても構いません。
  - `score`には、検索結果のスコアが入ります。詳しくは後述します。
  - `title`には、検索結果のタイトルが入ります。
  - `iconUrl`には、ウェブサイトのアイコンのURLが入ります。何のアイコンにするかはインスタンスの任意です。
  - `description`には、ウェブサイトの説明が入ります。クエリによってインスタンスが変更しても良いでしょう。
  - `url`には、そのウェブサイトのURLが入ります。
  - `thumbnail`には、ウェブサイトのサムネイルが入ります。ない場合は`null`です。画像検索などに使えるでしょう。
#### `score`について
検索結果のウェブサイトのスコアです。
結果の合計が、結果の数になるようにします。

例えば:

`https://example.com`と`https://example.net`が結果の場合、結果の数は2個なので、スコアの合計は2.0にする必要があります。
`https://example.com`の方を上位にしたい場合、`https://example.com`のスコアを`1.5`、`https://example.net`のスコアを`0.5`にできます。

スコアの合計が結果の数になっていれば、どんなスコアでも構いません。インスタンスにスコア付けは任されます。

クライアント側は、スコアの合計が結果の数になっているかどうかを確認します。浮動小数点数の誤差を考慮するため、
```
(スコアの合計) + 0.1
```
を上限とするようにします。

