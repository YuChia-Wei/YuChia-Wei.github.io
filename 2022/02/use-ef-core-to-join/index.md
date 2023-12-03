# Use EF-Core to Join


EF Core 中使用 Join

<!--more-->

**以下操作均使用 EF Core 5.0**
**以下程式碼範例皆為 EFCore.ToList() 執行前的 IQueryable 操作，也就是仍在組合 SQL 語法的階段**
**不管使用哪種寫法，皆應盡可能減少巢狀操作**

# Use LinQ Join

## Linq Join 的參考資料
* [LINQ-to-SQL-Query-Tips-INNER-JOIN-and-LEFT-JOIN)](https://blog.miniasp.com/post/2010/10/14/LINQ-to-SQL-Query-Tips-INNER-JOIN-and-LEFT-JOIN)
* [[LINQ+EF]如何做單一條件JOIN以及複合條件JOIN以及多個table的JOIN](https://dotblogs.com.tw/kevinya/2015/10/23/153677)

## 多欄位匹配
* 寫法是 Call Method 形式的 Linq，而非參考資料中使用的關鍵字形式的寫法
* Key 為多欄位的 Key，對應關係如下
    * tableA.SectName = tableB.SessionName
    * tableA.DistName = tableB.AreaName

```cs
private IQueryable<OutPutEntity> JointToOutPutEntity(
            IQueryable<TableA> tableADataList,
            IQueryable<TableB> tableBDataList)
{
    return tableADataList.Join(tableBDataList,
                                // Join 的第二個參數為 TableA 中要用來處理匹配判斷的欄位
                                tableA => new ValueTuple<string, string>( tableA.SectName,    tableA.DistName),
                                // Join 的第三個參數為 TableB 中要用來處理匹配判斷的欄位
                                tableB => new ValueTuple<string, string>( tableB.SessionName, tableB.AreaName),
                                (tableA , tableB) => new OutPutEntity
                                {
                                    LandCode = tableA.LandCode,
                                    CurrentValue = tableA.CurrentValue,
                                    CurrentPrice = tableA.CurrentPrice,
                                    SectName = tableB.SessionId,
                                    DistName = tableB.ZipCode,
                                });
}

```

另一種寫法，如果要將 Join 好的 Table 欄位全數輸出的話可以使用此寫法

```cs
private IQueryable<OutPutEntity> JointToOutPutEntity(
            IQueryable<TableA> tableADataList,
            IQueryable<TableB> tableBDataList)
{
    return tableADataList.Join(tableBDataList,
                                // Join 的第二個參數為 TableA 中要用來處理匹配判斷的欄位
                                tableA => new { tableA.SectName,    tableA.DistName},
                                // Join 的第三個參數為 TableB 中要用來處理匹配判斷的欄位
                                tableB => new { tableB.SessionName, tableB.AreaName},
                                (tableA , tableB) => new { tableA, tableB });

    //=====非 Method 寫法如下=======
    return from tableAData in tableADataList
                 join tableBData in tableBDataList on new
                     {
                         fieldA = tableAData.SectName, fieldB = tableAData.DistName
                     }
                     equals new
                     {
                         fieldA = tableBData.SessionName, fieldB = tableBData.AreaName
                     }
                 select new { tableAData, tableBData };

}
```

## 單欄位匹配
* 寫法是 Call Method 形式的 Linq，而非參考資料中使用的關鍵字形式的寫法
* Key 為單一欄位的 Key，對應關係如下
    * tableA.SectName = tableB.SessionName

```cs
private IQueryable<OutPutEntity> JointToOutPutEntity(
            IQueryable<TableA> tableADataList,
            IQueryable<TableB> tableBDataList)
{
    return tableADataList.Join(tableBDataList,
                                tableA => tableA.SectName,
                                tableB => tableB.SessionName,
                                (tableA , tableB) => new OutPutEntity
                                {
                                    LandCode = tableA.LandCode,
                                    CurrentValue = tableA.CurrentValue,
                                    CurrentPrice = tableA.CurrentPrice,
                                    SectName = tableB.SessionId,
                                    DistName = tableB.ZipCode,
                                });
}

```

# Select To Object

```cs
private IQueryable<OutPutEntity> JointToOutPutEntity(
            IQueryable<TableA> tableADataList,
            IQueryable<TableB> tableBDataList)
{
    return tableADataList.Select(tableAData => new OutPutEntity()
                                {
                                    A = tableAData.SomeField,
                                    B = tableBDataList.FirstOrDefault()?.SomeField
                                    TableBOneRow = tableBDataList.FirstOrDefault() //TableBOneRow 的型別為 TableB

                                    //下面這種寫法會無法使用
                                    //TableBOneRows = tableBDataList //TableBOneRow 的型別為 IEnumerable<TableB>
                                });
}
```

# Use LinQ SelectMany

當需要組合多個 Table 時，可使用 SelectMany 來處理。
當使用 SelectMany 後，可以建立極為複雜的可測試查詢 (因為依賴了 IQueryable 與 IEnumerable，而非 SQL)，但是會有一定的開發、維護難度，需要注意。

```cs
private IQueryable<OutPutEntity> JointToOutPutEntity(
            IQueryable<TableA> tableADataList,
            IQueryable<TableB> tableBDataList)
{
    return tableADataList.SelectMany(tableAData => tableBDataList.Where(tableBData => tableBData.BId == tableAData.AId)
                                                                 .DefaultIfEmpty(), // 這個設定很重要，會影響 Sql 組成與底下的輸出設定
                                     (tableAData, tablaBData) => new { tableAData, tableBData });
}
```

