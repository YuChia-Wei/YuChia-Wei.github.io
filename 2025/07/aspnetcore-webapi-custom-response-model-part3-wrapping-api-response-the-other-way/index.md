# 幫 AspNetCore WebApi 包上自己的 response model，Part 3 : 其他包裝方法


先前在撰寫 wrapping response action filter 的時候其實有利用 ChatGPT 進行輔助，在與 ChatGPT 對話的過程中也有討論到使用 Middleware 處理回應包裝的部分，我想利用這一篇記錄一下當時的對話內容以及結論。
</br>
</br>
因此，本文大部分內容都是由 AI 產出，我再進行整理與調整。
</br>
</br>
{{<admonition warning >}}
本文中所使用的方式我並沒有在正式環境中使用，僅是因為在先前與 ChatGPT 討論的過程中有針對這種做法進行討論，然後覺得這段適合保留下來而有這一篇文章。
{{</admonition >}}

<!--more-->

在 ASP.NET Core 中統一包裝 API 回應內容是許多開發團隊會採用的工程實踐，特別是當我們希望所有回傳結果都有一致格式、攜帶追蹤資訊（如 traceId）、以及能明確標示成功與否的情況下。

我在實務中已經採用 **Action Filter** 作為主要的實作方式，這篇文章會補充說明除了 Action Filter 外，還有哪些可行方案，以及每種方式的特性與限制，幫助團隊選擇最適合的整合點。

---

## ✅ 已採用方案：使用 Action Filter 包裝 API 回應

這裡不贅述實作細節，但使用 ActionFilterAttribute 實作的優點包括：

* 可以在 `OnActionExecuted` 中取得 `Result` 並重新包裝
* 可讀取 `HttpContext`（例如 traceId header）
* 容易註冊為全域 filter，或用 `[ServiceFilter]`、`[TypeFilter]` 套用到特定 controller/action
* 搭配泛型 `ApiResponse<T>` 可讓 Swagger 產生正確文件

不過，這種方式只能處理 Action 成功執行後的回應，例外處理仍需額外補強。

---

## 🚀 替代方案：使用 Middleware 包裝 API Response

另一種常見方式是在 ASP.NET Core 的 Middleware pipeline 中進行包裝。
這種方式具有較低階、全域、可掌控例外與所有回應內容的特性。

### 優點：

* 能處理所有 HTTP 回應，包含例外（Exception）
* 能完整攔截並改寫 Response.Body
* 與 Action 無關，不需依賴 MVC 或控制器邏輯

### 缺點：

* 需複製/暫存 Response.Body（MemoryStream）
* 無法透過泛型直接建立 Swagger 文件
* 需自行判斷何時不包裝（如 Swagger, health check, 圖片等）

### 範例程式碼（簡化版）：

```csharp
public class ApiResponseMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var originalBody = context.Response.Body;
        using var tempStream = new MemoryStream();
        context.Response.Body = tempStream;

        try
        {
            await next(context);

            tempStream.Seek(0, SeekOrigin.Begin);
            var bodyText = await new StreamReader(tempStream).ReadToEndAsync();
            tempStream.Seek(0, SeekOrigin.Begin);

            var wrapped = new ApiResponse<object>
            {
                Success = context.Response.StatusCode is >= 200 and < 300,
                Data = JsonConvert.DeserializeObject<object>(bodyText),
                TraceId = context.TraceIdentifier
            };

            context.Response.ContentType = "application/json";
            var result = JsonConvert.SerializeObject(wrapped);
            await context.Response.WriteAsync(result);
        }
        catch (Exception ex)
        {
            context.Response.StatusCode = 500;
            var error = new ApiResponse<string>
            {
                Success = false,
                Message = "Internal Server Error",
                Data = ex.Message,
                TraceId = context.TraceIdentifier
            };
            await context.Response.WriteAsync(JsonConvert.SerializeObject(error));
        }
        finally
        {
            tempStream.Seek(0, SeekOrigin.Begin);
            await tempStream.CopyToAsync(originalBody);
        }
    }
}
```

> 註：實務上建議加入判斷 `Request.Path` 是否為 `/swagger`, `/health`, 檢查 `Content-Type` 是否為非 `application/json`，以略過非 API 呼叫。

---

## 📚 結語：如何選擇？

| 技術             | 能處理例外                | 能包裝非 Controller | Swagger 支援  | 可讀取 traceId |
|------------------|---------------------------|---------------------|---------------|----------------|
| Action Filter    | ❌（需配 ExceptionFilter） | ❌                   | ✅（搭配泛型） | ✅              |
| Exception Filter | ✅（Controller 例外）      | ❌                   | ❌             | ✅              |
| Middleware       | ✅                         | ✅（全域）           | ❌             | ✅              |

實務建議：

* 若你的應用為純 API，建議 **ActionFilter + ExceptionFilter** 為主體，**Middleware 做例外補強**
* 若你想確保所有 response 都有一致格式、包含例外情境、或不依賴 MVC，可考慮 **Middleware 全包裝策略**

未來可持續觀察 ASP.NET Core 在 `ProblemDetails`、`IResultFilter`、`UseExceptionHandler` 等其他機制上的整合演進。

