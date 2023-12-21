# RESTful WebApi 設計概念


本文為以下書籍的閱讀筆記，整理與紀錄一些我認為重要的部分

- [Web API 建構與設計](https://www.tenlong.com.tw/products/9789865020590)

<!--more-->

# Web Api 設計

## RESTful Api 所遵循的設計規則

* 資源是 URL 的一部分，例如 /users
* 每一個資源通常有兩個 URL，分別代表群體與個體
    * 群體：GET http://{domain}/api/users
    * 個體：GET http://{domain}/api/users/{userid}
* 使用資源名詞而非動詞
    * 行為：取得使用者資料
        * 應使用：GET {domain}/api/users/{userid}
        * 不應：  GET {domain}/api/getUserInfo/{userid}
* 利用 HTTP 方法表示功能行為
    | HTTP 方法 | URL: /users                                                           | URL: /users/{userid}    |
    |-----------|-----------------------------------------------------------------------|-------------------------|
    | Get       | 取得所有使用者資料                                                    | 取得指定 user id 的資料 |
    | POST      | 建立一位使用者 (資料以 Request Body 傳遞)                             | 不適用                  |
    | PUT       | 批次更新使用者 (資料以 Request Body 傳遞、資料必須為完整的 User 資料) | 指定更新 user id 的資料 |
    | PATCH     | 批次更新使用者 (資料以 Request Body 傳遞、資料為部分的 User 資料)     | 指定更新 user id 的資料 |
    | DELETE    | 刪除所有使用者                                                        | 刪除指定使用者          |
* 伺服器回傳標準的 HTTP 狀態碼來表示處理結果
    | 回應碼 | 執行結果                       |
    |--------|--------------------------------|
    | 2xx    | 成功                           |
    | 3xx    | 資源不存在、已被移除           |
    | 4xx    | 用戶端錯誤(缺少參數或太多請求) |
    | 5xx    | 伺服器錯誤                     |
* RESTful Api 回傳格式通常為 JSON 或 XML

## URL 資源關係

URL 的路徑要可以表達每個資源的關係

    GET {domain}/users/{userid}/name
    取得          某個使用者id   的名稱

    GET {domain}/unit/{unitid}/name
    取得          某個單位id   的名稱

## 非 CRUD 操作

以 HTTP 行為表示與資料的互動關係，以後續的資源欄位表示要進行的行為

| HTTP 方法 | 對應行為                                                 |
|-----------|----------------------------------------------------------|
| POST      | 驗證、比較等無更新資料的行為                             |
| PUT       | URL 即代表行為                                           |
| PATCH     | 依據 query string 或是 request body 內的資料決定行為結果 |

範例
| 預期行為                           | HTTP 方法 | URL                                         | 備註                                                    |
|------------------------------------|-----------|---------------------------------------------|---------------------------------------------------------|
| 驗證使用者資料資料                 | POST      | {domain}/users/verify                       | 驗證資訊會放置於 Http Request Body                      |
| 調整使用者狀態為有效或無效 (範例1) | PATCH     | {domain}/users/{userid}/status?invalid=true | 使用 query string 設定狀態                              |
| 調整使用者狀態為有效或無效 (範例2) | PATCH     | {domain}/users/{userid}/status              | 使用 HttpBody 設定狀態: Body Context = {"invalid"=true} |
| 設定使用者為無效使用者             | PUT       | {domain}/users/{userid}/status/invalid      |                                                         |
| 設定使用者為有效使用者             | PUT       | {domain}/users/{userid}/status/valid        |                                                         |

## C# .Net Core 3.1 後的 Route 設定

[參考資料](https://docs.microsoft.com/zh-tw/aspnet/core/mvc/controllers/routing?view=aspnetcore-5.0)

