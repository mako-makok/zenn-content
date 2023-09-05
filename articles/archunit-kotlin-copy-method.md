---
title: "ArchUnitでKotlinのdata classのcopyメソッドを禁止する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin", "ArchUnit"]
published: false
publication_name: "loglass"
---

# はじめに

ログラスの小林([@mako-makok](https://twitter.com/mako_makok))です。
ご存知の方も多いと思いますが、Kotlin の data class に自動的に生える copy method に対して、ArchUnit を使ったアプローチをご紹介します。

# Kotlin の data class の概要

Kotlin には data class という機能があります。
https://kotlinlang.org/docs/data-classes.html

data class で宣言するだけで自動的に`equals`, `hashCode`, `toString`, `componentN`をよしなに実装してくれます。
なので、以下のようなことができるようになります。

```kotlin
data class Clazz(
    val foo: String,
    val bar: Int
)

fun main() {
    // これはtrueになる
    Clazz("foo", 1) == Clazz("foo", 1)

// componentNが実装されているので、分解宣言ができる
    val (foo, bar) = Clazz("foo", 1)
}
```

# data class の不都合な点

data class の実装されるメソッドの中に、copy メソッドがあります。
copy メソッドはある data class のプロパティを一部だけ入れ替えてインスタンスを作ることができるメソッドです。

```kotlin
data class Clazz(
    val foo: String,
    val bar: Int
)

fun main() {
    // Clazz("foo", 2)となる
    Clazz("foo", 1).copy(bar = 2)
}
```

非常に便利な反面、このメソッドがあることによってファクトリ関数を無視してインスタンスを作ることが可能になってしまいます。
以下の実装だとコンストラクタは隠すことができますが、copy メソッドでメールアドレスのルールを突破できます。

```kotlin
data class MailAddress private constructor (
    val value: String
) {
    companion object {
        fun of(localPart: String, domain: String): MailAddress {
            return MailAddress("$localPart@$domain")
        }
    }
}

fun main() {
    MailAddress.of("a", "b").copy(value = "broke through")
}
```

# 回避策

いくつか回避策はあります。

## 実装を interface で隠す

```kotlin
sealed interface MailAddress {

    val value: String

    private data class MailAddressData(
        override val value: String
    ): MailAddress

    companion object {
        fun of(localPart: String, domain: String): MailAddress {
            return MailAddressData("$localPart@$domain")
        }
    }
}

fun main() {
    // MailAddressはinterfaceなので、copyメソッドは呼び出せない
    MailAddress.of("a", "b").copy(value = "broke through")
}
```

この実装だと MailAddress 自体は interface となっています。そのため、data class によって生成されるメソッドは外では一切利用できません。
data class のメリットを享受しつつ、ファクトリメソッドを経由しないとインスタンスを生成できないようになっています。
これは目的を達成するという意味では完璧ですが、2 点ほど問題があります。

- 拡張関数を使用しなければ`internal`が利用できなくなる
- 実装が冗長になる

sealed された class や interface では`internal`を利用できません。
https://kotlinlang.org/docs/sealed-classes.html#inheritance-in-multiplatform-projects

`internal`を利用したい場合、拡張関数を利用する必要があります。

```kotlin
sealed interface MailAddress {

    val value: String

    private data class MailAddressData(
        override val value: String
    ): MailAddress

    companion object {
        fun of(localPart: String, domain: String): MailAddress {
            return MailAddressData("$localPart@$domain")
        }
    }
}

internal fun MailAddress.localPart(): String {
    return value.substringBefore("@")
}
```

実装が冗長になる点については、`MailAddress`程度であれば問題ありませんが、例えば、以下のようにパターンマッチを実現する実装を目指すと段々とカオスになります。

```kotlin
sealed interface MailAddress {
    val value: String
}
sealed interface ValidatedMailAddress: MailAddress {
    private data class ValidatedMailAddressData(
        override val value: String
    ): ValidatedMailAddress
}
sealed interface UnValidatedMailAddress: MailAddress {
    private data class UnValidatedMailAddress(
        override val value: String
    ): ValidatedMailAddress
}

fun main(mailAddress: MailAddress) {
    when (mailAddress) {
        is ValidatedMailAddress -> {}
        is UnValidatedMailAddress -> {}
    }
}
```

個人的には、上記の方法がデフォルトの機能だけを利用して不正なインスタンス生成を許さないという目的を満たすには良いと感じています。

## data class を使わない

元も子もないですが、思い切って data class を利用しないというのも 1 つの手です。

```kotlin
class MailAddress private constructor (
    val value: String
) {
    companion object {
        fun of(localPart: String, domain: String): MailAddress {
            return MailAddress("$localPart@$domain")
        }
    }

    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is MailAddress) return false

        return value == other.value
    }

    override fun hashCode(): Int {
        return value.hashCode()
    }
}
```

この実装でも目的は達成されますが、Kotlin の良さが失われてしまいます。
また頻繁に改修が入るクラスであれば、プロパティの拡張で`equals`や`hashCode`の実装漏れで同値チェックがすり抜けてしまう危険性もあります。

# ArchUnit で copy メソッドを抑制する

この記事の本題です。

## ArchUnit とは

`ArchUnit`は主に Java 向けのライブラリで、Java コードのアーキテクチャをテストできます。
Java 向けですが、Kotlin でも利用できます。
https://www.archunit.org/

使い方はシンプルで、archunit を依存に追加してテストを書き始めることができます。

```kts
dependencies {
    testImplementation("com.tngtech.archunit:archunit:1.1.0")
}
```

ユースケースは公式で公開されていますので、ぜひ御覧ください。
https://www.archunit.org/use-cases

ArchUnit の実態は内部でバイトコードの解析とリフレクションでソースコードを解析しており、利用者側は提供されているユーティリティでモジュール内のパッケージ/クラス/メソッドの依存関係をテストできます。
パッケージからメソッド名、あとは引数に至るまで取れてしまうので、要は機械的にできそうなソースコードのチェックは大抵できる代物となっています。

## テストの実装

下記をコピーしてパッケージ名などを埋めていただくと試すことができます。

```kotlin
class DomainDataClassCannotUseCopyMethod {

    @Test
    fun `ドメイン層のデータクラスに付属するcopyメソッドを呼び出すことはできない`() {
        val targetPackages = arrayOf(
            // 解析対象のパッケージを列挙する
            "com.example.domain..",
            "com.example.infrastructure..",
        )
        val allApplicationClasses = ClassFileImporter()
            .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
            .importPackages(*targetPackages)

        methods()
            .should(BanDomainLayerDataClassCopyCondition)
            .check(allApplicationClasses)
    }

    private object BanDomainLayerDataClassCopyCondition :
        ArchCondition<JavaMethod>("ドメイン層に定義されたdata classのcopyは、同じクラス内でしかコールすることができない") {
        override fun check(method: JavaMethod, events: ConditionEvents) {
            // Kotlinで自動的に実装されるメソッドにはメタデータの関係上サフィックスに$defaultが付与される
            if (method.name != "copy\$default") {
                return
            }

            // domainレイヤー以外のcopyは許容する
            if (!method.owner.packageName.startsWith("{your domain package}")) {
                return
            }

            val temporaryAllowMethods = listOf(
                "com.example.Main.main()",
            )

            method.callsOfSelf.forEach { caller ->
                if (temporaryAllowMethods.contains(caller.origin.fullName)) {
                    return@forEach
                }

                // this.copyは許容する
                if (method.owner != caller.originOwner) {
                    events.add(
                        SimpleConditionEvent.violated(
                            /* correspondingObject = */ method,
                            /* message = */buildErrorMessage(caller = caller),
                        ),
                    )
                }
            }
        }

        private fun buildErrorMessage(caller: JavaMethodCall): String {
            val callerClassAndMethod =
                "${caller.origin.owner.name.replace("${caller.origin.owner.packageName}.", "")}.${caller.origin.name}"
            val calledClassAndMethod = "${caller.target.owner.simpleName}.${caller.name}"
            return "$callerClassAndMethod calls $calledClassAndMethod"
        }
    }
}

```
