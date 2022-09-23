---
title: "Cypressで環境変数を指定する方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "TypeScript", "Cypress", "E2E"]
published: true
---

## 独自の環境変数を設定する

Cypress では、実行時に環境変数を利用することができます。  
例えば E2E テスト内で利用するユーザーを変えるために、ID/Password を外から渡したいなどの場面で役に立つかもしれません。  
環境変数は `cypress.json` に記述したり、実行時に指定することができます。以下は `cypress.json` を利用した例です。

```json
{
  "baseUrl": "https://example.com",
  "env": {
    "id": "foo",
    "password": "bar"
  }
}
```

テスト内では`Cypress.env`で値を取得できます。

```ts
const id: string = Cypress.env("id");
const password: string = Cypress.env("password");
```

## 環境変数を切り替える

テストを動かすとき、実行ごとに変数を渡したい場合があります。  
例えばログイン用の ID/Password を変えたり、テスト対象の環境を変えるなどです。  
環境変数を渡す方法はいくつかあります。

### config file を渡す

cypress は起動時、特に何も指定しないとプロジェクトルートの`cypress.json`が使用されます。
CLI 上で行うこともできますが、定型化するとなるとファイルを切り替えたいということもあるかと思います。その場合は`-config-file`オプションを使用します。
例えば、以下のようなファイルを用意します。

- cypress.dev.json

```json
{
  "env": {
    "id": "hoge",
    "password": "fuga"
  }
}
```

起動時に`-config-file`で`cypress.dev.json`を指定すると、`id=hoge, password=fuga`が使用されます。

```sh
cypress run -config-file cypress.dev.json
```

### 実行時に直接指定する

何らかの理由で直接環境変数を利用したい場合があるかもしれません。その場合は`env`オプションで渡すことができます。

```sh
# 複数のenvをセットする場合はカンマ区切りにする
# envオプションはconfig fileの設定値に上書きされる仕様のため、特に指定しなければconfig fileの値が使用される
cypress run --env "id=hoge,password=fuga"
```
