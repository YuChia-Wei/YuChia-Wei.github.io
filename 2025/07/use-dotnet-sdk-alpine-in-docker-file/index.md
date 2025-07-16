# 在 dockerfile 內使用 alpine 的 dotnet sdk image 建置 dotnet 服務


在我以前的工作環境中，由於建置機採用地端管理，且有儲存容量等限制，如何利用 docker 的 build cache 來增加儲存空間的利用率成了必須思考的議題。

也因為我在規劃 **包含 opentelemetry auto instrumentation** 的 base image 時，也會需要同時發布多種不同 linux 版本的 image，因此，這邊特別研究了多種不同 linux 發行版的 dotnet build 參數並應用在 dockerfile 之中，來讓一份 dockerfile 可以發布多種不同的成品，並且使用相同的建置階段。

這邊使用的 github repo 為 [YuChia-Wei/otel-dotnet-auto-instrumentation](https://github.com/YuChia-Wei/otel-dotnet-auto-instrumentation)

<!--more-->

## 利用建置變數控制 dotnet 版本與 os 版本

dockerfile 可以利用變數來控制許多部份，在這邊就利用了變數來控制建置的 dotnet 版本以及要建置的 os 版本，最終可以達到利用以下的 docker build 命令來變換建置成品的 dotnet 版本與 os 類型。

```shell
docker build --build-arg dotnetVersion=8.0 --build-arg releaseOsType=linux -t my-app:latest .
```

### OS 參數

- linux = debian / ubuntu
- linux-musl = alpine

### dockerfile - 建置階段

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG dotnetVersion=8.0
ARG releaseOsType=linux
WORKDIR /src
COPY ["OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins/OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins.csproj", "OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins/"]
RUN dotnet restore "OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins/OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins.csproj" \
COPY . .
WORKDIR "/src/OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins"
RUN dotnet build "OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins.csproj" \
    -c Release \
    -o /app/build \
    -f net${dotnetVersion} \
    --os ${releaseOsType} \
    --arch x64

FROM build AS publish
ARG dotnetVersion=8.0
ARG releaseOsType=linux
RUN dotnet publish "OpenTelemetry.AutoInstrumentation.AspNetCore.Plugins.csproj" \
    -c Release \
    -o /app/publish \
    -f net${dotnetVersion} \
    --os ${releaseOsType} \
    --arch x64
```

### 參考資料

- [dotnet build 參考文件](https://learn.microsoft.com/zh-tw/dotnet/core/tools/dotnet-build)
- [dotnet publish 參考文件](https://learn.microsoft.com/zh-tw/dotnet/core/tools/dotnet-publish)
- [os rid 參考文件](https://learn.microsoft.com/zh-tw/dotnet/core/rid-catalog)
- [os type list (rid list)](https://github.com/dotnet/sdk/blob/main/src/Layout/redist/PortableRuntimeIdentifierGraph.json)

