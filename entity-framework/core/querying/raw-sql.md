---
title: 生 SQL クエリ - EF Core
author: rowanmiller
ms.date: 10/27/2016
ms.assetid: 70aae9b5-8743-4557-9c5d-239f688bf418
uid: core/querying/raw-sql
ms.openlocfilehash: 91592ea9f7c73f10446993282c1874c852000871
ms.sourcegitcommit: c9c3e00c2d445b784423469838adc071a946e7c9
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/18/2019
ms.locfileid: "68306545"
---
# <a name="raw-sql-queries"></a>生 SQL クエリ

Entity Framework Core を使用すると、リレーショナル データベースを操作するときに生 SQL クエリにドロップ ダウンすることができます。 この方法は、実行するクエリが LINQ を使用して表現できない場合や、LINQ クエリを使用すると非効率的な SQL が生成される場合に役立ちます。 生 SQL クエリは、エンティティ型か、モデルの一部である[クエリ型](xref:core/modeling/query-types) (EF Core 2.1 以降) を返すことができます。

> [!TIP]  
> この記事の[サンプル](https://github.com/aspnet/EntityFramework.Docs/tree/master/samples/core/Querying)は GitHub で確認できます。

## <a name="basic-raw-sql-queries"></a>基本的な生 SQL クエリ

*FromSql* 拡張メソッドを使用して、生 SQL クエリに基づいた LINQ クエリを開始できます。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSql("SELECT * FROM dbo.Blogs")
    .ToList();
```

生 SQL クエリを使用してストアド プロシージャを実行できます。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogs")
    .ToList();
```

## <a name="passing-parameters"></a>パラメーターを渡す

SQL を受け取る API の場合と同様に、SQL インジェクション攻撃から保護するには、ユーザー入力をパラメーター化することが重要です。 SQL クエリ文字列にパラメーターのプレースホルダーを含めて、追加の引数としてパラメーター値を指定することができます。 指定したパラメーター値は自動的に `DbParameter` に変換されます。

次の例では、ストアド プロシージャに 1 つのパラメーターを渡しています。 これは `String.Format` 構文のように見えますが、指定された値はパラメーターにラップされ、生成されたパラメーター名は `{0}` プレースホルダーが指定された場所に挿入されます。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogsForUser {0}", user)
    .ToList();
```

これは同じクエリですが、EF Core 2.0 以降でサポートされている文字列補間構文を使用しています。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSql($"EXECUTE dbo.GetMostPopularBlogsForUser {user}")
    .ToList();
```

また、DbParameter を構築し、それをパラメーター値として指定することもできます。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = new SqlParameter("user", "johndoe");

var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogsForUser @user", user)
    .ToList();
```

これにより、SQL クエリ文字列で名前付きパラメーターを使用できるようになります。これはストアド プロシージャに省略可能なパラメーターがある場合に便利です。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = new SqlParameter("user", "johndoe");

var blogs = context.Blogs
    .FromSql("EXECUTE dbo.GetMostPopularBlogs @filterByUser=@user", user)
    .ToList();
```

## <a name="composing-with-linq"></a>LINQ による作成

データベース内で SQL クエリを作成できる場合は、LINQ 演算子を使用して初期の生 SQL クエリ上に作成することができます。 `SELECT` キーワードで始まる SQL クエリを作成できます。

次の例では、テーブル値関数 (TVF) から選択し、LINQ を使用してフィルター処理と並べ替えを実行する生 SQL クエリを使用します。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSql($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Where(b => b.Rating > 3)
    .OrderByDescending(b => b.Rating)
    .ToList();
```

## <a name="change-tracking"></a>変更追跡

`FromSql()` を使用するクエリは、EF Core 内の他の LINQ クエリとまったく同じ変更追跡ルールに従います。 たとえば、クエリでエンティティ型を予測する場合、既定で結果は追跡されます。  

次の例では、テーブル値関数 (TVF) から選択し、.AsNoTracking() の呼び出しを使用して変更追跡を無効にする生の SQL クエリを使用しています。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Query<SearchBlogsDto>()
    .FromSql($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .AsNoTracking()
    .ToList();
```

## <a name="including-related-data"></a>関連データを含める

`Include()` メソッドを使用して、他の LINQ クエリと同様に、関連データを含めることができます。

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSql($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Include(b => b.Posts)
    .ToList();
```

## <a name="limitations"></a>制限事項

生 SQL クエリを使用する場合、注意が必要な制限事項がいくつかあります。

* SQL クエリは、エンティティ型またはクエリ型のすべてのプロパティのデータを返す必要があります。

* 結果セットの列名は、プロパティがマップされている列名と一致する必要があります。 これは EF6 と異なる点です。EF6 では、生 SQL クエリのプロパティ/列のマッピングは無視され、結果セットの列名はプロパティ名と一致する必要がありました。

* SQL クエリに関連データを含めることはできません。 ただし、多くの場合、`Include` 演算子を使用して関連データを返すクエリを作成することができます (「[関連データを含める](#including-related-data)」を参照してください)。

* このメソッドに渡された `SELECT` ステートメントは、一般的にコンポーザブルである必要があります。EF Core がサーバー上で追加のクエリ演算子を評価する必要がある場合 (たとえば、`FromSql` の後に適用される LINQ 演算子を変換する場合)、指定された SQL はサブクエリとして扱われます。 これは、渡される SQL に、次のようなサブクエリでは無効な文字またはオプションを含めてはならないことを意味します。
  * 末尾のセミコロン
  * SQL Server では、末尾のクエリ レベル ヒント (例: `OPTION (HASH JOIN)`)
  * SQL Server で、`SELECT` 句の `OFFSET 0` または `TOP 100 PERCENT` を伴わない `ORDER BY` 句

* `SELECT` 以外の SQL ステートメントは、自動的に非コンポーザブルと認識されます。 その結果、ストアド プロシージャのすべての結果が常にクライアントに返され、`FromSql` の後に適用されたすべての LINQ 演算子はメモリ内で評価されます。

> [!WARNING]  
> **生 SQL クエリには常にパラメーター化を使用する:** ユーザーの入力を検証するだけでなく、生 SQL クエリ/コマンドで使用される値には常にパラメーター化を使用してください。 `FromSql` や `ExecuteSqlCommand` のような生 SQL 文字列を受け取る API では、値をパラメーターとして簡単に渡すことができます。 FormattableString を指定できる `FromSql` と `ExecuteSqlCommand` のオーバーロードでも、SQL インジェクション攻撃からの保護に役立つ方法での文字列補間構文の使用が許可されます。 
> 
> 文字列連結または補間を使用して、クエリ文字列のいずれかの部分を動的に構築する場合、または動的 SQL としてユーザー入力を実行できるステートメントまたはストアド プロシージャにその入力を渡す場合は、SQL インジェクション攻撃から保護するためにご自身で入力を検証する必要があります。
