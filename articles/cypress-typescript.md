---
title: "Cypress で TypeScript を利用する"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "TypeScript", "Cypress", "E2E"]
published: true
---

## 環境構築

前回の環境を引き続き利用します。  
まずは TypeScript のセットアップをします。

```sh
npm install typescript
npx tsc --init --types cypress --lib dom,esnext --target esnext --module esnext
```

js ファイルを ts ファイルに置換します。

```sh
cd cypress
find . -name '*.js' | xargs rename 's/js/ts/'
```

Cypress を起動します。

```sh
npx cypress open
```

あとは適当なテストを選択すると、テストが実行されると思います。
ひとまず TS で動くようになりました。

## Additional Setting

とりあえずは動くようになりましたが、まだ不十分なのでいくつか設定が必要です。

### plugins/index.ts の修正

`cypress/plugins/index.ts`を開いてみると、おそらく`module.exports`の部分でエラーになっていると思うので以下のように修正します。

- `tsconfig.json`

```json
{
  ...
  "types": ["cypress", "node"]
  ...
}
```

- `cypress/plugins/index.ts`

```typescript
module.exports = (
  on: Cypress.PluginEvents,
  config: Cypress.PluginConfigOptions
) => {
  // `on` is used to hook into various events Cypress emits
  // `config` is the resolved Cypress config
};
```

### カスタムコマンドを TypeScript で書いてサジェストできるようにする

前回はカスタムコマンドを js で書いてみましたが、TypeScript で書くこともできます。
型を書けば`cy.`からサジェストできるようになるので、今回は一例を示します。
ログインを行う処理がコメントアウトされているので、外して型を書きます。

```typescript
Cypress.Commands.add("login", (email: string, password: string) => {});
```

この状態で適当な spec.ts を開いて`cy.login`と書いてもそんなものはないので怒られます。ちょっと面倒ですが、Cypress の namespace に生やして上げる必要があります。

```typescript
Cypress.Commands.add("login", (email: string, password: string) => {});
declare global {
  namespace Cypress {
    interface Chainable<Subject> {
      login(email: string, password: string): Chainable<Subject>;
    }
  }
}
export {};
```

再度 spec.ts を開いて`cy.login`と入力してみると無事サジェストされます。
