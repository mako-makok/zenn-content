---
title: "kotlin-resultを半年使ってみて"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "kotlin-result"]
published: true
published_at: 2023-09-11 12:00
publication_name: "loglass"
---

# はじめに

ログラスの小林([@mako-makok](https://twitter.com/mako_makok))です。

この記事は毎週必ず記事がでるテックブログ **["Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6)** の **4 週目** の記事です！
1 年間連続達成まで **残り 49 週** となりました！

Kotlin でのエラーハンドリングの改善に向けて、`kotlin-result`というライブラリを導入したのですが、使い始めて約半年経過しました。
今回は使ってみて実際にどうだったかを振り返ってみます。

# What kotlin-result

https://github.com/michaelbull/kotlin-result

Rust の`Result`や、Scala の `Either` など、関数型の概念を取り入れた言語には例外ではなく、失敗する可能性のある処理は成功と失敗の型をシグネチャで表現できるようになっています。
`kotlin-result` は それらの表現を Kotlin でも利用できるようにしたライブラリです。

内部の実装を見てみるとそれらは代数的データ型で実装されています。

```kotlin
public sealed class Result<out V, out E> { ... }
public class Ok<out V>(public val value: V) : Result<V, Nothing>() { ... }
public class Err<out E>(public val error: E) : Result<Nothing, E>() { ... }
```

`Ok` は成功値、 `Err` 失敗値を表すクラスで、それぞれ `Result` を継承しています。

利用側は `Result` のインタフェースを使用し、`Ok` か `Err` の型を確定させないとそれぞれ値を取り出すことができません。

```kotlin
enum class ErrorCode {
    ZERO_DIVISION_ERROR
}

fun div(a: Int, dividedBy: Int): Result<Int, ErrorCode> {
    return if (dividedBy == 0) {
        return Err(ErrorCode.ZERO_DIVISION_ERROR)
    } else {
        Ok(a / dividedBy)
    }
}

fun main() {
    when (val result = div(10, 0)) {
        is Ok -> println(result.value)
        is Err -> throw ArithmeticException()
    }
}
```

これが `Result` の基本になっており、`koltin-result` では `Result` と、それらに付随する拡張関数が多数提供されています。

```kotlin
// findの結果がnullであれば NotFoundException に変換する
userRepository.find().toResultOr(::NotFoundException)

val value = div(10, 0)
    // 成功値のIntをBigDecimalに変換する
    .map { it.toBigDecimal() }
    // エラーコードを例外に変換する
    .mapError {
        when (it) {
            ErrorCode.ZERO_DIVISION_ERROR -> ArithmeticException()
        }
    }
    //  ResultがOkであれば値を取り出し、ErrであればThrowableをthrowする
    .getOrThrow()
```

他にも便利な API がありますので、ぜひ使ってみてください。
より具体の話はこちらの記事がとても良くまとまっており、大変参考にさせていただきました。

[kotlin-result 入門](https://blog.nnn.dev/entry/2023/06/22/110000)

そもそもなぜ Kotlin にこのような代数的に処理できる機構が必要なのか、については以下のスライドをご覧ください。

[Kotlin における型の世界と エラーハンドリング / Type World and Error Handling in Kotlin](https://speakerdeck.com/makomakok/type-world-and-error-handling-in-kotlin)

# Kotlin 標準の Result について

Kotlin には標準で Result が付属しています。

Kotlin 1.5 以前は戻り値に指定できないという制約があったのですが、それ以降のバージョンでは普通に利用できます。

標準の`Result`は似たようなことができるのですが、いくつかの問題があります。

```kotlin
fun div(a: Int, dividedBy: Int): Result<Int> {
    // 戻り値がkotlin.Resultは成功値の型の指定しかできないので、失敗値の型が消失する
    return if (dividedBy == 0) {
        // failureにはThrowableしか投入することができない
        return Result.failure(ArithmeticException())
    } else {
        Result.success(a / dividedBy)
    }
}

fun main() {
    val value = div(10, 0)
        .map { it.toBigDecimal() }
        .getOrThrow()
}
```

# 実際 kotlin-result をプロダクションコードで利用してみて

新規開発部分は積極的に `Result` を利用するようにしています。
基本パターンマッチ前提のコードベースになるので、定性的にはなりますがエラーのハンドリングやパターンの組み合わせの不備によって発生する不毛なバグは減ったように感じます。

そしてこれが一番大きなところですが、一括更新系の機能で複数行のエラーを出すことがかなり容易になりました。

Loglass のプロダクトの性質上、ファイルアップロードによるデータ追加/更新がかなり多いです。

また、扱うデータ量も多く、間違っている行が複数ある場合に著しく UX を損ねる結果にもなるので、自前実装で機構を作らなくてもサクッと作れるのはとても良かったです。

下記のコードは一例です。

```kotlin
data class UserName(
    val value: String,
) {
    companion object {
        fun of(value: String): Result<UserName, IllegalArgumentException> {
            return value.let {
                if (it.isBlank()) {
                    Err(IllegalArgumentException("ユーザー名は必須です"))
                } else {
                    Ok(it)
                }
            }.andThen {
                if (it.length > 20) {
                    Err(IllegalArgumentException("ユーザー名は20文字以内で入力してください"))
                } else {
                    Ok(it)
                }
            }.map {
                UserName(it)
            }
        }
    }
}

data class UserCode(
    val value: String,
) {
    companion object {
        fun of(value: String): Result<UserCode, IllegalArgumentException> {
            return value.let {
                if (it.isBlank()) {
                    Err(IllegalArgumentException("コードは必須です"))
                } else {
                    Ok(it)
                }
            }.andThen {
                if (it.length > 20) {
                    Err(IllegalArgumentException("コードは20文字以内で入力してください"))
                } else {
                    Ok(it)
                }
            }.map {
                UserCode(it)
            }
        }
    }
}

data class User(
    val name: UserName,
    val code: UserCode,
)

data class Row(
    val userName: String,
    val code: String,
)

fun List<Row>.toUsers(): List<Result<User, List<IllegalArgumentException>>> {
    return this.map { row ->
        val userNameResult = UserName.of(row.userName)
        val userCodeResult = UserCode.of(row.code)

        zip(
            { userNameResult },
            { userCodeResult },
            { userName, userCode -> User(userName, userCode) }
        ).mapError { getAllErrors(userNameResult, userCodeResult) }
    }
}

fun main(rows: List<Row>) {
    rows
        .toUsers()
        .partition()
        .let { (oks, errs) ->
            if (errs.isNotEmpty()) {
                throw IllegalArgumentException(
                    errs
                        .flatten()
                        .map { it.message }
                        .joinToString("\n")
                )
            } else {
                userRepository.save(oks)
            }
        }
}
```

これらは同じチームの [ゆいとさん](https://twitter.com/Yuiiitoto)がより詳しく書かれたスライドもありますので、こちらを御覧ください。

[B2B SaaS あるある！ 一括処理のエラーハンドリングを Kotlin で関数型的に処理する / Kotlin Functional Multi Error Handling](https://speakerdeck.com/yuitosato/kotlin-functional-multi-error-handling)

# kotlin-result 以外の選択肢

`kotlin-result` 以外の選択肢としては[Arrow](https://arrow-kt.io/)が挙げられます。

`Arrow`はより関数型の思想を反映したライブラリとなっており、`arrow-core`や`arrow-core-retrofit`のような複数のモジュールがあります。

`Either` や`Nel`など便利なクラスやコレクションを内包した`arrow-core`、ほかは `retrofit` など様々なユースケースに対応したモジュールとなっています。
`Arrow`は多様なユースケースを叶えられる厚いライブラリであり、選定をする際非常に悩みましたが、下記の理由で `kotlin-result` を選択しました。

- Result(Either)以外の機構も入ってくるため、学習コストが高すぎると判断した
- 万が一 Arrow へ移行することになっても、`typealias`による補完と機械的な置換で済みそう

大前提、`kotlin-result` の貢献度合いは高く Kotlin 標準では成し得なかった体験をもたらしています。

しかしながら、開発をしながら若干片手落ち感が残る部分もあります。

一例ですが、成功値の合成は不自由なく行えるものの、`Arrow`に比べてエラーの合成が少し弱いと感じる部分があります。

前節のコードでは`zip`してから`getAllErrors`でエラーを全取得していました。

```kotlin
fun List<Row>.toUsers(): List<Result<User, List<IllegalArgumentException>>> {
    return this.map { row ->
        val userNameResult = UserName.of(row.userName)
        val userCodeResult = UserCode.of(row.code)

        zip(
            { userNameResult },
            { userCodeResult },
            { userName, userCode -> User(userName, userCode) }
        ).mapError { getAllErrors(userNameResult, userCodeResult) }
    }
}
```

この方法だと`User`にプロパティを追加した際、`getAllErrors`の引数に追加し損ねると意図したエラー値にならなくなります。

`Arrow` には`zipOrAccumlate`というメソッドがあり、エラー値を accmlate してくれます。(※以下は`kotlin-result`での実装イメージです)

```kotlin
fun List<Row>.toUsers(): List<Result<User, List<IllegalArgumentException>>> {
    return this.map { row ->
        zipOrAccumlate(
            { UserName.of(row.userName) },
            { UserCode.of(row.code) },
            { userName, userCode -> User(userName, userCode) }
        )
    }
}
```

`kotlin-result`はかなり API が豊富ですが、`Arrow`はそれを上回る勢いで充実しています。

# kotlin-result の pros/cons

ここまで `kotlin-result` と標準の`Result`、`Arrow`との比較について記載しましたが、最後に pros/cons をつらつら書いて終わります。

## pros

- エラーの型を明示的にできるため、メソッドの振る舞いをすべてシグネチャで表現できるようになる
- シグネチャで表現できるので、ハンドリングはパターンマッチになり、基本的にハンドリング漏れ/リカバリ処理の漏れがなくなる
- ライブラリが軽量
- 付属している拡張関数が豊富。流れるようなインタフェースでエラーハンドリング処理を書くことができ、開発体験が良い

# cons

- Result や Either は関数型の概念になるため、一定学習コストは高くなってしまう
- 利用者に書き方の強制ができないため、throw がどうしても混在してしまう
- エラーハンドリングをリッチに行おうとするとどうしてもコードの見通しが悪くなってしまう
- (`Arrow`と比べて)API の豊富さには劣る

# 終わりに

身も蓋もないですが、ライブラリ選定はプロジェクトの状況によって変わります。

数ある中でも、`kotlin-result`は少ない導入コストで、Kotlin の課題を解決する OSS だと私は感じました。

今後は`kotlin-result`自体にも貢献していきたいです。

# 参考文献

- https://github.com/michaelbull/kotlin-result
- https://blog.nnn.dev/entry/2023/06/22/110000
- https://arrow-kt.io/
- https://speakerdeck.com/yuitosato/kotlin-functional-multi-error-handling
