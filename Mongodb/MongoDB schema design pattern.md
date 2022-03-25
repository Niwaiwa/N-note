###### tags: `MongoDB`

# MongoDB schema design pattern

## The Polymorphic Pattern(多態模式)

同一collection的document可存不固定key的資料

## The Attribute Pattern(屬性模式)

多種類似的同類型資料使用同一個key存成json array  


```
原始
{
    title: "Star Wars",
    director: "George Lucas",
    ...
    release_US: ISODate("1977-05-20T01:00:00+01:00"),
    release_France: ISODate("1977-10-19T01:00:00+01:00"),
    release_Italy: ISODate("1977-10-20T01:00:00+01:00"),
    release_UK: ISODate("1977-12-27T01:00:00+01:00"),
    ...
}
```
```
屬性模式
{
    title: "Star Wars",
    director: "George Lucas",
    …
    releases: [
        {
        location: "USA",
        date: ISODate("1977-05-20T01:00:00+01:00")
        },
        {
        location: "France",
        date: ISODate("1977-10-19T01:00:00+01:00")
        },
        {
        location: "Italy",
        date: ISODate("1977-10-20T01:00:00+01:00")
        },
        {
        location: "UK",
        date: ISODate("1977-12-27T01:00:00+01:00")
        },
        … 
    ],
    … 
}
```

## The Bucket Pattern(桶模式)

觀測資料之類時間序列數據可以以時間區間作為單個桶來儲存  
並可使用Trasaction統計計算  

```
原
{
   sensor_id: 12345,
   timestamp: ISODate("2019-01-31T10:00:00.000Z"),
   temperature: 40
}

{
   sensor_id: 12345,
   timestamp: ISODate("2019-01-31T10:01:00.000Z"),
   temperature: 40
}

{
   sensor_id: 12345,
   timestamp: ISODate("2019-01-31T10:02:00.000Z"),
   temperature: 41
}
```

```
桶模式
{
    sensor_id: 12345,
    start_date: ISODate("2019-01-31T10:00:00.000Z"),
    end_date: ISODate("2019-01-31T10:59:59.000Z"),
    measurements: [
       {
       timestamp: ISODate("2019-01-31T10:00:00.000Z"),
       temperature: 40
       },
       {
       timestamp: ISODate("2019-01-31T10:01:00.000Z"),
       temperature: 40
       },
       … 
       {
       timestamp: ISODate("2019-01-31T10:42:00.000Z"),
       temperature: 42
       }
    ],
   transaction_count: 42,
   sum_temperature: 2413
} 
```

## The Outlier Pattern(異常模式)

既定資料有可能突然超過限制故註記為異常資料  
之後將超過的資料以關聯形式儲存  

```
{
    "_id": ObjectID("507f1f77bcf86cd799439011")
    "title": "A Genealogical Record of a Line of Alger",
    "author": "Ken W. Alger",
    …,
    "customers_purchased": ["user00", "user01", "user02"]

}
```

```
{
    "_id": ObjectID("507f191e810c19729de860ea"),
    "title": "Harry Potter, the Next Chapter",
    "author": "J.K. Rowling",
    …,
   "customers_purchased": ["user00", "user01", "user02", …, "user999"],
   "has_extras": "true"
}
```

## The Computed Pattern(計算模式)

主要為提前計算資料  
其中又分定期計算資料、最後更新時間、隊列之類  

## The Subset Pattern(子集模式)

儲存特定的json array欄位資料  
可能只需要最近的幾筆, 更早之前的可以存到另一個位置作為子集來存取使用  
類似冷熱資料  

## The Extended Reference Pattern(擴展參考模式)

類似於SQL Table關係  
可將常使用的KEY關聯, 類似於SQL的反正規化, 減少查詢(join)次數  

## The Approximation Pattern(近似模式)

若資料不須絕對準確的數字, 只需要近似值的情況  
可以建立計數器來更新, 不需要每次都進DB更新資料  

## The Tree Pattern(樹模式)

不使用傳統的Table表示樹, 而使用保持有樹的結構且比較簡單的結構  
※此模式較複雜, 需要時再詳查

## The Preallocation Pattern(預分配模式)

如果資料為已知或固定的, 可以使用預分配模式簡化邏輯  
※此模式較複雜, 需要時再詳查

## The Document Versioning Pattern(文檔版本控制模式)

對文檔進行版控, 通常只搜尋最新版本  
前提是版本不會太多的情況, 契約書修訂等等

## The Schema Versioning Pattern(模型版本化模式)

添加schema_version的KEY讓程式可以區分當前版本來執行  
常用在不可停機或更新時間很長的情況  
