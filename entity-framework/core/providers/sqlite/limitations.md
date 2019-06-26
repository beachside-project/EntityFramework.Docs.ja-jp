---
title: SQLite データベース プロバイダー - 制限事項 - EF Core
author: rowanmiller
ms.date: 04/09/2017
ms.assetid: 94ab4800-c460-4caa-a5e8-acdfee6e6ce2
uid: core/providers/sqlite/limitations
ms.openlocfilehash: eaa7d5b1496172e4f3821433a1cd098ee7e8b737
ms.sourcegitcommit: 9bd64a1a71b7f7aeb044aeecc7c4785b57db1ec9
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 06/26/2019
ms.locfileid: "67394799"
---
# <a name="sqlite-ef-core-database-provider-limitations"></a>SQLite EF Core データベース プロバイダーの制限事項

SQLite プロバイダーでは、いくつかの移行の制限があります。 これらの制限事項のほとんどは、基になる SQLite データベース エンジンの制限事項の結果し、EF に固有ではありません。

## <a name="modeling-limitations"></a>モデリングの制限事項

(Entity Framework のリレーショナル データベース プロバイダーによって共有)、一般的なリレーショナル ライブラリは、モデリングの概念はほとんどのリレーショナル データベース エンジンに共通の Api を定義します。 これらの概念のいくつかは、SQLite プロバイダーによってサポートされていません。

* スキーマ
* シーケンス
* 計算列

## <a name="query-limitations"></a>クエリの制限事項

SQLite は、次のデータ型をネイティブにサポートしていません。 EF Core が、これらの型と等しいかどうかのクエリの値を読み書きできます (`where e.Property == value`) もサポートされています。 ただし、比較のようなその他の操作と、順序付けすると、クライアントで評価が必要です。

* DateTimeOffset
* Decimal (10 進数型)
* TimeSpan
* UInt64

代わりに`DateTimeOffset`DateTime 値を使用することをお勧めします。 複数のタイム ゾーンを処理する際は、値を保存し、し、適切なタイム ゾーンに変換する前に UTC に変換するをお勧めします。

`Decimal`型には、精度の高いレベルが用意されています。 有効桁数のレベルを必要としない場合をお勧め代わりに double を使用します。 使用することができます、[値コンバーター](../../modeling/value-conversions.md)クラスでの 10 進数の使用を続行します。

``` csharp
modelBuilder.Entity<MyEntity>()
    .Property(e => e.DecimalProperty)
    .HasConversion<double>();
```

## <a name="migrations-limitations"></a>移行の制限事項

SQLite データベース エンジンは、多数の他のリレーショナル データベースの大部分でサポートされているスキーマの操作をサポートしていません。 サポートされていない操作のいずれかの SQLite データベースに適用しようとしたかどうか、`NotSupportedException`がスローされます。

| 操作            | サポートされていますか。 | バージョンが必要です。 |
|:---------------------|:-----------|:-----------------|
| AddColumn            | ✔          | 1              |
| AddForeignKey        | ✗          |                  |
| AddPrimaryKey        | ✗          |                  |
| AddUniqueConstraint  | ✗          |                  |
| AlterColumn          | ✗          |                  |
| CreateIndex          | ✔          | 1              |
| CreateTable          | ✔          | 1              |
| DropColumn           | ✗          |                  |
| DropForeignKey       | ✗          |                  |
| DropIndex            | ✔          | 1              |
| DropPrimaryKey       | ✗          |                  |
| DropTable            | ✔          | 1              |
| DropUniqueConstraint | ✗          |                  |
| RenameColumn         | ✔          | 2.2.2            |
| RenameIndex          | ✔          | 2.1              |
| RenameTable          | ✔          | 1              |
| EnsureSchema         | ✔ (操作なし)  | 2.0              |
| DropSchema           | ✔ (操作なし)  | 2.0              |
| 挿入               | ✔          | 2.0              |
| 更新               | ✔          | 2.0              |
| 削除               | ✔          | 2.0              |

## <a name="migrations-limitations-workaround"></a>移行の制限の回避策

回避策をいくつかは、移行テーブルを実行するコードを手動で記述してこれらの制限の再構築します。 テーブルのリビルドには、既存のテーブルの名前変更、新しいテーブルの作成、新しいテーブルへのデータのコピー、および古いテーブルの削除が必要です。 使用する必要があります、`Sql(string)`メソッドを次の手順の一部を実行します。

参照してください[その他の種類のテーブル スキーマ変更を行う](http://sqlite.org/lang_altertable.html#otheralter)詳細については、SQLite ドキュメント。

将来的に、EF がサポートこれらの操作の一部裏では、テーブルの再構築のアプローチを使用しています。 できます[当社の GitHub プロジェクトでこの機能を追跡](https://github.com/aspnet/EntityFrameworkCore/issues/329)します。
