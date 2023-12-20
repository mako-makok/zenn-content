---
title: "ログラスのバックエンド技術スタック2023"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "Java", "SpringBoot", "ORM", "OpenAPI"]
published: true
publication_name: "loglass"
---

![](/images/open-loglass-tech-stack-2023.title.png)

:::message
この記事はログラスアドベントカレンダーの 20 日目の記事です！
:::

# はじめに

こんにちは、ログラスの小林([@mako-makok](https://twitter.com/mako_makok))です！

昨日は[@asa_kossy](https://twitter.com/asa_kossy)さんの[「ログラスのプロダクトマネージャーチームが今年取り組んだこと、いま苦労していること 2023」](https://note.com/asa_kossy/n/nc85eca4a8066)でした。

ログラス社はありがたいことに、 CTO 協会主催の「開発者体験が良い」イメージのある企業で 25 位にランクインさせていただいております。
この結果は非常に光栄だと思っており、自分が想像する要因としては DDD、スクラム、技術的投資の 3 点だと思っています。

https://zenn.dev/yuitosato/articles/9db2a0fe90313e
https://levtech.jp/media/article/interview/detail_304/
https://speakerdeck.com/urmot/loglass-technical-investment
https://note.com/go_nambu/n/n9b15c88dcf10

これらは継続的に活動しています。
ライブラリバージョンアップは欠かさずやりますし、DDD に関しては社内で DDD という言葉はもうほぼ使われていないレベルで浸透しており、機能追加の際はドメインエキスパートと会話つつ、モデルと実装を行き来して開発しております。

この認知は非常に嬉しい結果ではありますが、実際は開発者体験に課題を感じる部分はまだまだあります。
もちろん都度改善を回してはいるものの、開発者体験は青天井なので常に高みを目指したくなってしまいます。

今回は、現状のログラスの技術スタックをご紹介し、

- ログラスってまだまだ改善の余地がいっぱいあるな、いっちょ俺が直したる
- ライブラリの選定に迷っているけど参考になった

と感じていただけるといいな、と思っております。

# Loglass シリーズのアプリケーション特性

[「Loglass 経営管理」](https://www.loglass.jp/)をはじめとした Loglass シリーズ（以下 Loglass）は、経営管理領域の中でも、特に予算・実績の管理と予実対比に強みがあるアプリケーションです。
多くの場合、**各企業で Excel で管理されている予算データ**と**会計システム等に存在している実績データ**を取り込み、それぞれ**同じ単位で集計し対比**することで経営分析を行えます。

このことから、アプリケーションとしては以下のような特性があります。

- データ量は BtoB のアプリケーションにしては多め
  - つまりデータ取り込みは大量データを扱う
  - 特に実績は会計システムから出力されるレコードを保持する必要がある
- 大量のデータをバックエンドで集計してクライアント側で大量データを表示する
  - バックエンド・フロントエンドともにパフォーマンスがボトルネックになることが多い
  - 逆にトラフィックが急に増えてスパイクするようなことは現状ない
- 予算は Excel を利用していることが多く、xlsx ファイルをアップロードする機会が多い
  - CSV アップロードはほぼ無い
  - 正規化されていないデータをアップロードするケースもある

# 技術スタックのご紹介

- フレームワーク: SpringBoot
- 言語: Kotlin
- O/R マッパ: jooq, Exposed
- データベース: PostgreSQL
- マイグレーション: Flyway
- テスト: jUnit5, mockk
- lint・formatter: ktlint
- API ドキュメント: springdoc-openapi
- クライアントコード生成: openapi-generator

## SpringBoot

https://spring.io/projects/spring-boot

SpringBoot は、Spring Framework という Web 開発のためのモジュールを、一定ひとまとめにしたフレームワークです。
枯れたフレームワークのため、機能開発という面に関しては基本的に何をするにも困りません。

特徴的なのはそのモジュールの多さで、トランザクションや認証はもちろんのこと、モジュラモノリスまでサポートされています。
開発に必要なものはほぼ Spring Framework のモジュールとして用意されていますし、ドキュメントもとても充実しています。

### 良い点

ドキュメントが充実している・アプリケーション開発に必要なモジュールのラインナップが充実している他に、
Aspect Oriented Programming(以下 AOP)という概念があり、それを利用することでフレームワークの利用者はアプリケーションのロジックを書くことに専念できます。

https://docs.spring.io/spring-framework/reference/core/aop.html

例えば、SpringBoot でトランザクションのハンドリングは以下のようなコードで行うことができます。

- `@Transactional` をつけることにより、自動的にメソッドの開始時にトランザクションが開始さる
- 自動的にメソッドの終了後にトランザクションが終了される
- 例外が発生するとロールバックが行われる

```java
@Transactional
public class FindUserUsecase {

	public User findUser(String id) {
		// ...
	}
}
```

AOP は自前で定義が可能です。
例えば、下記コードでは `com.xyz.dao` パッケージ以下に定義されているメソッドが正常に終了(例外が発生していなければ) `doAccessCheck` が実行されます。

```java
@Aspect
public class AfterReturningExample {

    @AfterReturning("execution(* com.xyz.dao.*.*(..))")
    public void doAccessCheck() {
        // ...
    }
}
```

他にも

- あるアノテーションがついていたら
- 特定の例外が throw されたら

と、かなり細かくハンドリングできるのでロギングなど共通処理を抜き出して隠蔽することが可能になっています。
SpringBoot にはここでは紹介しきれないくらいたくさんの便利な機能があるので、気になる方はぜひドキュメントや実際に触ってみてください。

### 気になる点

とはいえ SpringBoot も万能ではなく、他のフレームワークや言語と比べて気になる部分も存在します。

#### 起動が遅い

- SpringBoot の起動時は、DI コンテナに全コンポーネントを登録する処理が走るため、アプリケーションが大きくなると相対的に起動時間が長くなる
- 数秒の差だが、開発の中ではもちろん何回も起動するので、地味にストレスになる
- チューニング手法はいくつかあるが、手をつけられていない

#### コンテナとの相性に問題があるとされている

- JVM 自体の起動が遅い/JVM はリソース消費が激しいというのが主な理由
- もちろんチューニングすれば一定解決はする

https://speakerdeck.com/hhiroshell/jvm-on-kubernetes

- 最近は GraalVM など、ハイパフォーマンスなランタイムやネイティブイメージへのビルドも可能になっているため、クラウドネイティブ化への投資は進んでいる
- とはいえ Loglass は BtoB の SaaS で、アプリケーションのトラフィックが急に増え、オートスケールが走ることはほぼないので問題にはなっていない

#### AOP の弊害として、処理がブラックボックス化されるかつ実装が追いかけづらくなる

AOP は便利ですが、共通処理が増えてくると共通処理同士がかち合って意図せぬ挙動になる/そもそも AOP のコード自体が追いづらいという問題があります。

例えば Loglass ではテナント間でデータが漏洩しないように、AOP で Row Level Security 関連の事前設定を行っています。

https://www.slideshare.net/koichiromatsuoka/postgresql-springaop

実装者が意識せずとも、セキュリティが担保される素晴らしい仕組みですが、今まで JVM に触れてこなかったメンバーに説明する際のコストはどうしてもトレードオフになってしまいます。

## Kotlin

https://kotlinlang.org/

Kotlin はとてもバランスの取れた言語で、言語仕様もシンプルかつ専用の API も非常に便利で、書き心地がとても良いです。

また[アップデートの周期が非常に早く](https://kotlinlang.org/docs/roadmap.html)、次バージョンではコンパイラの刷新によってパフォーマンス含め大幅に改善されるとのことなので、今後の進化も楽しみです。
https://kotlinlang.org/docs/roadmap.html

### 良い点

※ Java との比較が中心です

#### Kotlin 特有の高い表現力を活用できる

- 拡張関数

```kotlin
fun MutableList<Int>.swap(index1: Int, index2: Int) {
  val tmp = this[index1] // 'this' corresponds to the list
  this[index1] = this[index2]
  this[index2] = tmp
}

fun main() {
  // あたかも特定のデータの振る舞いとするかのように記述可能
  mutableListOf(1, 2, 3).swap(0, 2) // => [3, 2, 1]
}
```

- Java に比べて記述量が少ない

```kotlin
// Streamへの変換/終端操作/itでのelementへのアクセスが可能
listOf(1, 2, 3).map { it * it }
```

- 非同期処理([coroutine](https://kotlinlang.org/docs/coroutines-overview.html))
  - 所謂軽量スレッドで、I/O 待ちや、複数スレッドでの並列処理が可能になります

https://kotlinlang.org/docs/coroutines-basics.html#scope-builder-and-concurrency

#### mockk が便利

Kotlin 製のモックライブラリですが、非常に便利です。
テストの頁で詳しく記載します。

### 気になる点

#### SpringBoot との相性問題はある

- GitHub で kotlin で Issue を検索するといくつかある

https://github.com/spring-projects/spring-boot

- 非同期処理や AOP でバグは何度か実際に踏みました

#### Java に比べて(体感)ビルドが遅い

- これは Kotlin 特有の箇所が追加でかかってくることが原因か
  - 具体的にベンチマークを取ったわけでは無いので杞憂の可能性
- 上述した通り、K2 コンパイラという新コンパイラに刷新される予定のため、バージョンアップにより速度向上は期待できる

#### Kotlin には隠れたコストがあるため、Java で書くよりコストが高くつく傾向にある

- Kotlin のオブジェクトは immutable が基本になっているため、カジュアルに変数へ展開していると意図せずメモリを消費することがある
  - こちらに関連して、GC のタイミングが一部 Java と違うので、大量データを捌くときなどは注意
- こちらの記事がとてもよくまとまっていたので、参考にさせていただきました

https://retheviper.github.io/posts/kotlin-hidden-cost-1/

## マイグレーション

Flyway を利用しています。
Loglass の DB スキーマは SQL をバージョニングしています。
実行は Gradle の公式のプラグイン付属のスクリプトを利用しています。( `flywayMigrate` )
以前 Flyway の記事を書かせていたので、こちらもよろしければ御覧ください。
https://zenn.dev/mako_makok/articles/use-flyway-migration

Flyway の気になる部分は特別ないのですが、複数人で開発しているとマイグレーションスクリプトのバージョン番号がコンフリクトして修正の手間が発生します。
この部分に関しては GitHub のマージキューやプロテクト、CI など仕組みで解決していきたいと思っています。

## O/R マッパ

`jooq` を使っています。
https://www.jooq.org/

jooq は専用の API を扱う必要があるものの、SQL ライクに記述でき、単純なクエリなどはほぼ copilot が書いてくれるためとても体験が良いです。

```java
DSLContext create = DSL.using(connection, dialect);

create.select(AUTHOR.FIRST_NAME, AUTHOR.LAST_NAME, count())
      .from(AUTHOR)
      .join(BOOK).on(BOOK.AUTHOR_ID.eq(AUTHOR.ID))
      .where(BOOK.LANGUAGE.eq("DE"))
      .and(BOOK.PUBLISHED_IN.gt(2008))
      .groupBy(AUTHOR.FIRST_NAME, AUTHOR.LAST_NAME)
      .having(count().gt(5))
      .orderBy(AUTHOR.LAST_NAME.asc().nullsFirst())
      .limit(2)
      .offset(1)
      .forUpdate()
      .fetch();
```

### 良い点

#### code generator も付属しており、DB を立ち上げておくとスキーマを読んでコードを自動生成してくれる

- サンプルの `AUTHOR` や `BOOK` などが自動生成されるイメージ
- コードの自動生成することによって、タイプセーフに SQL を書くことができる
  - あるカラムを drop したときコード生成の結果削除されるので、そのカラムの利用箇所はコンパイルエラーになる

#### マイグレーションとコード自動生成の相性から生まれる体験が良い

実運用としては

1. 前述の Flyway の`flywayMigrate`
2. jooq の自動コード生成
3. テストデータの流し込み

をひとまとめにしたスクリプトを用意することで、DB からコード自動生成まで一気通貫で行うようにしています。

### 気になる点

概ね体験は良いのですが、課題点としては自動生成コードを Git に commit しているので、自動生成漏れが発生して、merge 後に気づく問題があります。

Flyway 同様、GitHub のマージキューやプロテクト、CI など仕組みで解決していきたいと思っています。

#### Exposed からの乗り換え

Exposed は Kotlin で書かれた JetBrains 製の O/R マッパ です。そのため Kotlin と非常に相性は良いのですが、

- 公式を含めてもドキュメントがかなり少ない
- Exposed の SpringBoot プラグインが Exposed のメンテナによって管理されているため、SpringBoot の追従に遅れてしまう
  - プラグインで SpringBoot のバージョンがロックされてしまう

などの問題があります。

やはり flyway でバージョニングされたスキーマ + jooq の自動コード生成がとても便利なため、現在は jooq への置き換えを行っており、Exposed で書かれたコード割合は減少しています。

## テスト

テストランナーは `jUnit5` 、モックは `mockk` を利用しています。

https://junit.org/junit5/
https://mockk.io/

jUnit5 はパラメータ化テストが若干書きづらいという課題があり、それを解決するため一時期 `kotest` という、Kotlin 製のテストライブラリを検討した時期があったのですが、

- 乗り換えコストが大きい
- 得られるメリットはパラメータ化テストが一定書きやすくなる
- IntelliJ に専用のプラグインを入れる必要がある

といった理由で見送りました。
とはいえ圧倒的に書き味は良くなるので、初期選定時などでは候補として有力です。
https://kotest.io/

mockk は非常に強力で、少ない記述量でモックを書くことができます。
static な関数のモックや柔軟な値キャプチャもでき、非常に高機能です。

### 良い点

#### jUnit5

- 枯れており、ドキュメントが豊富かつ様々な記事もある
- エクステンションが豊富で、少し込み入ったことをするときに既にライブラリがある

#### mockk

- 少ない記述量
- static な関数のモック、キャプチャなど基本的になんでもモックと検証ができる

### 気になる点

#### jUnit5

パラメータ化テストが非常に高機能ですが、他のライブラリと比べると若干書きづらい部分があります。
https://qiita.com/oohira/items/5030182af29a30166868

ちなみに kotest だとかなり簡潔に記述できます。
https://kotest.io/docs/proptest/property-test-functions.html

#### mockk

- なんでもできる反面、簡単に実装の詳細をテストできてしまう
- そもそもモック自体使い所はかなり絞ったほうが良い
- テストしづらければ interface を変えることを検討した方が良い
  - テストし辛いコードが出現したときに mockk で解決してしまいがち
- 直近のアップデートでパフォーマンス劣化？ が起きていそう(CI でメモリが溢れてたまに落ちてしまう)

https://github.com/mockk/mockk/issues/997

## lint

`ktlint` を利用しています。
https://pinterest.github.io/ktlint/1.0.1/

Gradle プラグインを入れており、 `ktlintFormat` を実行しフォーマットをかけ、CI でチェックを行っています。
特殊な設定もしておらず、基本的に生で使っています。

### 気になる点

`detekt` に移行するかはかなり迷っています。
https://detekt.dev/

detekt だとフォーマッティング以外にもカスタムチェックや IntelliJ でのチェックなどがあり、便利なのですが Kotlin のバージョンがロックされやすいのがトレードオフになります。

もちろん ktlint にも通ずることではあります。
しかし detekt は ktlint と比べて機能が豊富(detekt formatter 自体 ktlint のラッパー)な分、体感 Kotlin のバージョンアップ時に detekt が壊れてしまうことが多い気がしています。
特に Kotlin はバージョンアップ頻度が高いので、非常に迷うところです。
折衷案として、現在はランタイムを分けて detekt を利用しており、CI で non null assertion をしている箇所を怒るなどをしております。

https://zenn.dev/loglass/articles/51958a02455a10

## ドキュメント

`springdoc-openapi` を利用しています。
https://springdoc.org/

SpringBoot を立ち上げると自動的に Swagger が立ち上がります。
以前まで `SpringFox` を利用していましたが、開発が止まっているためこちらに乗り換え済みです。
https://springfox.github.io/springfox/

Loglass ではバックエンドの Controller の実装を正として開発を進めており、yml を書くことはしていないです。
こちらは特に気になる部分もなく、快適に使わせていただいています。

## 自動コード生成

`openapi-generator` を利用し、 `typescript-axios` で実行しています。
https://openapi-generator.tech/

SpringBoot を起動した状態で専用の npm script を実行すると、API のスキーマを読みに行きクライアントコードを自動生成しています。

### 良い点

- 様々な generator があり、そこから選択して利用できる
- TypeScript から扱いやすい型で生成してくれる(enum が union types で生成される)

### 気になる点

そもそも自動生成のフローを見直したほうがいいのでは？ という気もしており、悩ましい部分です。

- クライアントコードが 1 ファイルにまとまるため、数万行のファイルになり重い
- namespace が区切られていないので、controller に生やすメソッドは全てユニークにする必要が出てくる
  - 例えばリソースの名称が users であれば、コントローラーのメソッドは `create/find/list/delete/update` など単純なものにしたい
  - 他の controller の名称とメソッド名がかぶると list1, list2…のような連番でクライアントコードが生成されてしまう
- SpringBoot を起動しないとクライアントコードを生成できない
- メンテナンスされていない generator がいくつかある

# フロントエンド

今回はバックエンドが中心のため割愛しますが、フロントエンドに関してはいくつかの解説記事がありますので、ぜひそちらを参照ください。

## フレームワーク、ステート管理、フォームについて

https://zenn.dev/yuitosato/articles/9db2a0fe90313e

## バリデーションライブラリについて

https://zenn.dev/loglass/articles/abe85e10e229f3

# さいごに

Loglass の細かいライブラリ群まで深掘らせていただきました。

このように、ログラスの開発はまだまだ多くの伸びしろがあり、フロントエンド/バックエンドともに改善することでグッと開発者体験を上げていけると思っています。

「いっちょ直したる！」「そここうすると良くなるから教えてあげてもいいよ」という方がいらっしゃいましたらぜひお話させてください。

https://hrmos.co/pages/loglass/jobs/1813462408235663423
https://hrmos.co/pages/loglass/jobs/1813462408235663423
