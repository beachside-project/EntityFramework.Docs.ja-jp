---
title: 移行履歴テーブルのカスタマイズ-EF6
author: divega
ms.date: 10/23/2016
ms.assetid: ed5518f0-a9a6-454e-9e98-a4fa7748c8d0
ms.openlocfilehash: eb19f367611a86f685557a6741a5f2f0bad6b718
ms.sourcegitcommit: cc0ff36e46e9ed3527638f7208000e8521faef2e
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/06/2020
ms.locfileid: "78415683"
---
# <a name="customizing-the-migrations-history-table"></a>移行履歴テーブルのカスタマイズ
> [!NOTE]
> **EF6 以降のみ** - このページで説明する機能、API などは、Entity Framework 6 で導入されました。 以前のバージョンを使用している場合、一部またはすべての情報は適用されません。

> [!NOTE]
> この記事では、基本的なシナリオでの Code First Migrations の使用方法を理解していることを前提としています。 そうでない場合は、続行する前に[Code First Migrations](~/ef6/modeling/code-first/migrations/index.md)を読む必要があります。

## <a name="what-is-migrations-history-table"></a>移行履歴テーブルとは

移行履歴テーブルは、データベースに適用された移行の詳細を格納するために Code First Migrations によって使用されるテーブルです。 既定では、データベース内のテーブルの名前は MigrationHistory \_\_され、データベースへの最初の移行を適用するときに作成されます。 Entity Framework 5 では、アプリケーションが Microsoft Sql Server データベースを使用していた場合、このテーブルはシステムテーブルでした。 これは Entity Framework 6 で変更され、移行履歴テーブルはシステムテーブルとしてマークされなくなりました。

## <a name="why-customize-migrations-history-table"></a>移行履歴テーブルをカスタマイズする理由

移行履歴テーブルは Code First Migrations によってのみ使用され、手動で変更すると移行が中断される可能性があります。 ただし、既定の構成が適切ではなく、テーブルをカスタマイズする必要がある場合があります。次に例を示します。

-   <sup>サードパーティの</sup>移行プロバイダーを有効にするには、列の名前やファセットを変更する必要があります。
-   テーブルの名前を変更するには
-   \_\_MigrationHistory テーブルには、既定以外のスキーマを使用する必要があります。
-   コンテキストの特定のバージョンの追加データを格納する必要があるため、テーブルに列を追加する必要があります。

## <a name="words-of-precaution"></a>予防策

移行履歴テーブルを変更することは強力ですが、過剰な作業を行わないように注意する必要があります。 EF runtime では、現在、カスタマイズされた移行履歴テーブルにランタイムとの互換性があるかどうかは確認されません。 そうでない場合は、実行時にアプリケーションが中断したり、予期しない動作が発生したりする可能性があります。 これは、データベースごとに複数のコンテキストを使用する場合にさらに重要になります。この場合、複数のコンテキストで同じ移行履歴テーブルを使用して、移行に関する情報を格納できます。

## <a name="how-to-customize-migrations-history-table"></a>移行履歴テーブルをカスタマイズする方法

開始する前に、最初の移行を適用する前にのみ、移行履歴テーブルをカスタマイズできることを把握しておく必要があります。 ここで、コードにします。

最初に、システムから派生したクラスを作成する必要があります。このクラスは、 履歴コンテキストクラスは DbContext クラスから派生するため、移行履歴テーブルを構成することは、fluent API での EF モデルの構成とよく似ています。 OnModelCreating メソッドをオーバーライドし、fluent API を使用してテーブルを構成するだけです。

>[!NOTE]
> 通常、EF モデルを構成するときは、base を呼び出す必要はありません。DbContext. OnModelCreating () に空の本文があるため、オーバーライドされた OnModelCreating メソッドから OnModelCreating () が行われました。 これは、移行履歴テーブルを構成する場合には当てはまりません。 この場合、OnModelCreating () オーバーライドで最初に行うことは、base を呼び出すことです。OnModelCreating ()。 これにより、移行履歴テーブルが既定の方法で構成されます。これにより、上書きする方法を調整できます。

たとえば、移行履歴テーブルの名前を変更し、"admin" というカスタムスキーマに格納するとします。 さらに、DBA は MigrationId 列の名前を移行\_ID に変更したいと考えています。  これを実現するには、履歴コンテキストから派生した次のクラスを作成します。

``` csharp
    using System.Data.Common;
    using System.Data.Entity;
    using System.Data.Entity.Migrations.History;

    namespace CustomizableMigrationsHistoryTableSample
    {
        public class MyHistoryContext : HistoryContext
        {
            public MyHistoryContext(DbConnection dbConnection, string defaultSchema)
                : base(dbConnection, defaultSchema)
            {
            }

            protected override void OnModelCreating(DbModelBuilder modelBuilder)
            {
                base.OnModelCreating(modelBuilder);
                modelBuilder.Entity<HistoryRow>().ToTable(tableName: "MigrationHistory", schemaName: "admin");
                modelBuilder.Entity<HistoryRow>().Property(p => p.MigrationId).HasColumnName("Migration_ID");
            }
        }
    }
```

カスタム履歴コンテキストを準備したら、[コードベースの構成](https://msdn.com/data/jj680699)を使用して登録することにより、EF に認識させる必要があります。

``` csharp
    using System.Data.Entity;

    namespace CustomizableMigrationsHistoryTableSample
    {
        public class ModelConfiguration : DbConfiguration
        {
            public ModelConfiguration()
            {
                this.SetHistoryContext("System.Data.SqlClient",
                    (connection, defaultSchema) => new MyHistoryContext(connection, defaultSchema));
            }
        }
    }
```

これは非常に簡単です。 これで、パッケージマネージャーコンソールに移動し、移行、追加、および更新を行うことができます。 これにより、履歴コンテキストの派生クラスで指定した詳細に従って、構成された移行履歴テーブルがデータベースに追加されます。

![データベース](~/ef6/media/database.png)
