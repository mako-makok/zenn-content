---
title: "なんとなく使わないFlyway"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java", "Database"]
published: true
---

# 動機

Java で DB マイグレーションするんだったら Flyway みたいな感じで何となく使っていたけど、エラーが出たときとかは雰囲気で対処していた。  
また Java で書ける性質もあり、誰かがゴリゴリにカスタマイズした Flyway を使っていたりしたので基本動作がよくわからなくなっていた。  
そんな中業務でマイグレーションの状態を破壊してしまったりしたので、いい加減ちゃんと知ろうと思って調べてみた。

# Flyway とは

Redgate 社が開発しているマイグレーション管理・実行ツール。  
フリープランと有料プランがあり、フリープランの実装は GitHub で`Apache License v2`として公開されている。  
有料プランになると追加でいろいろなコマンドが使えたり、Redgate 社から直接サポートを受けることができる。
個人的には以下の 3 点が目を引いた。

- 10 年前にリリースされた DB のリリースのサポート(通常は 5 年以内)
- Dry-run
- Undo

他にも様々な機能があるため、詳しくは[公式の Download + pricing](https://flywaydb.org/download)を参照。

また DB マイグレーションの管理の他に、Preview ではあるが`Flyway Hub`という GitHub と直接連携して CI 上でマイグレーションの整合性をチェックしてくれる機能があったりする。

# 基本的な使用方法

特定のディレクトリに特定の命名規則に沿った形で SQL を置いて`flyway migrate`を実行すると、整合性のチェックが行われた後に問題がなければ SQL が実行される。  
整合性のチェックには`flyway_shema_history`というメタテーブルが利用されている。  
このテーブルは初めて`flyway migrate`か、後述する`flyway baseline`というコマンドを利用したときに作成される。  
スキーマは以下のようになっている。

```sql
create table flyway_schema_history
(
    installed_rank integer                 not null -- 実行済みSQLの順序
        constraint flyway_schema_history_pk
            primary key,
    version        varchar(50),
    description    varchar(200)            not null,
    type           varchar(20)             not null, -- FlywayはSQL以外でもマイグレーションを実行できるため、どんな形式のマイグレーションが実行されたかが記載されている
    script         varchar(1000)           not null, -- マイグレーションに利用されたファイルのパス.
    checksum       integer,                          -- チェックサム. SQLファイルの整合性チェックに利用される.
    installed_by   varchar(100)            not null,
    installed_on   timestamp default now() not null,
    execution_time integer                 not null,
    success        boolean                 not null -- マイグレーションが成功したかどうか.
);


create index flyway_schema_history_s_idx
    on flyway_schema_history (success);
```

今回は利用できるコマンドについて詳しく調べてみた。
Flyway には 7 つのコマンドがある。前述の通り`undo`は有料版限定である。

## migrate

### 利用方法

```
flyway migrate
```

## 実行内容

特定のディレクトリ配下の SQL ファイルを再帰的に探索し、順に SQL が実行される。  
通常のマイグレーションの他にリピータブルマイグレーションというマイグレーションが実行される度に SQL が実行される仕組みがある。

デフォルトでは以下の仕様でマイグレーションが行われる。  
ちなみにプリフィックス・サフィックス・セパレータはオプションにより変更可能である。
それ以外の文字は何でも良い。

### 通常のマイグレーション

過去実行された SQL ファイルのチェックサムと、現在のチェックサムを検証した上で、相違がなければ未適用の SQL を実行する。

- プリフィックス: `V`
- セパレータ: `__`
- サフィックス: `.sql`

### リピータブルマイグレーション

チェックサムに**変更があった場合**マイグレーションが実行される。

- プリフィックス: `R`
- セパレータ: `__`
- サフィックス: `.sql`

### 細かな仕様

プリフィックスとセパレータの間はバージョンとして扱われ、`flyway_schema_history.version`に記録されている。  
このバージョンは`flyway migrate -target=<version>`で渡すことで指定したバージョン以降のマイグレーションだけを実行することができる。

## clean

スキーマ内のすべてのオブジェクトが削除される

### 利用方法

```
flyway clean
```

## info

これまでに行ったマイグレーションとそのステータスを確認できる。
よく目にするステータスはこれら

- Pending
  - クラスパスに含まれているが`flyway_schema_history`に未登録
- Success
  - クラスパスに含まれており、`flyway_schema_history`にレコードが登録済み、かつ`flyway_schema_history.success=true`の場合
- Failed
  - 何らかの理由で SQL の適用に失敗している
    - チェックサムの検証でコケている
    - SQL の実行でエラーになる

ステータスの一覧は[こちら](https://flywaydb.org/documentation/concepts/migrations#migration-states)

### 利用方法

```
flyway info
```

## validate

クラスパス内に存在するマイグレーションファイルと、現在の DB にあたっているマイグレーションとのチェックサムを比較し、差異がある場合はエラーを吐く

### 利用方法

```
flyway validate
```

## undo

`flyway migrate`と同じ要領で undo 用のマイグレーションファイルが実行される。  
migrate がコケたり間違って当てたりしたときに直してくる魔法のようなコマンドだと思っていたのだが、そんなことはない。  
また、リピータブルマイグレーションに対して undo するような SQL を実行することもできない。  
以下のルールに沿って作成された SQL が実行される。

- プリフィックス: `U`
- セパレータ: `__`
- サフィックス: `.sql`

### 利用方法

```
flyway undo
```

## baseline

baseline は flyway で管理がされていない DB に対して、自分が今どこにいるかを定義するコマンド。

### 利用方法

```
flyway baseline -baselineVersion=1.0.0
```

例えば DB とマイグレーションファイルが以下の状態であるとする

- マイグレーションファイル
  - V1\_\_foo.sql
  - V2\_\_bar.sql
- DB A
  - V1 相当の DDL があたっているが、flyway で管理されていない

このときに`flyway baseline -baselineVersion=1.1.0`を実行すると以下のようになる。

```
+------------+------------+--------------------------------------------------------------------------------+----------+---------------------+----------+
| Category   | Version    | Description                                                                    | Type     | Installed On        | State    |
+------------+------------+--------------------------------------------------------------------------------+----------+---------------------+----------+
|            | 1.1        | << Flyway Baseline >>                                                          | BASELINE | 2022-08-25 01:34:07 | Baseline |
| Versioned  | 2          | V2__bar                                                                        | SQL      |                     | Pending  |
+------------+------------+--------------------------------------------------------------------------------+----------+---------------------+----------+
```

## repair

`flyway info`で ステータスが Failed となるマイグレーションファイルを`Pending`にする

### 利用方法

```
flyway repair
```

## check

レポートを作成してくれるらしい。

> Requirements  
> .NET 6 is required in order to generate reports. You can download it from here.  
> sqlfluff is required for Code Analysis (-code). You can install it by running pip3 install sqlfluff==1.2.1.

とあり、生成するのは大変である。

### 利用方法

```
flyway check
```

## snapshot

DB のスナップショットを取ることができる。
現在は有料版でのみ β 版の提供らしいが、将来的に無償版でも利用できるようになるらしい。

### 利用方法

```
flyway snapshot
```

# その他機能

## Dry run

flyway migrate を実行すると走る SQL を 1 つのファイルにまとめて出力する機能。
`flyway migrate -dryRunOutput=dryrun.sql`とすると、以下のように出力される。  
(引用元: https://flywaydb.org/documentation/tutorials/dryruns#doing-a-dry-run)

```sql
---====================================
-- Flyway Dry Run (2018-01-25 17:19:17)
---====================================

SET SCHEMA "PUBLIC";

-- Executing: validate (with callbacks)
------------------------------------------------------------------------------------------
-- ...

-- Executing: migrate (with callbacks)
------------------------------------------------------------------------------------------
-- ...

-- Executing: migrate -> v3 (with callbacks)
------------------------------------------------------------------------------------------

-- Source: ./V3__Couple.sql
---------------------------
create table COUPLE (
    ID int not null,
    PERSON1 int not null references PERSON(ID),
    PERSON2 int not null references PERSON(ID)
);
INSERT INTO "PUBLIC"."flyway_schema_history" ("installed_rank","version","description","type","script","checksum","installed_by","execution_time","success") VALUES (2, '3', 'Couple', 'SQL', 'V3__Couple.sql', -722651034, 'SA', 0, 1);
-- ...
```

## Flyway Hub

公式によると以下のことが可能

- マイグレーションファイルの構文チェック
- マイグレションファイルの実行順序のチェック
- バージョン番号のコンフリクトの検知
- GitHub で管理しているマイグレーションファイルから新しい環境を作る

これは必要なのか・・・？

Flyway Hub 公式サイト: https://flywaydb.org/documentation/hub/overview.html

# 参考資料

- 公式ドキュメント
  - https://flywaydb.org/
- GitHub
  - https://github.com/flyway/flyway
