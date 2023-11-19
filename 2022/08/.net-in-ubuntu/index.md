# .net in ubuntu


Ubuntu 22.04 直接支援 .net 6 了!!!

<!--more-->

## Ubuntu 22.04 原生支援 .net 6

微軟官方 BLOG: https://devblogs.microsoft.com/dotnet/dotnet-6-is-now-in-ubuntu-2204/

Ubuntu BLOG: https://ubuntu.com/blog/install-dotnet-on-ubuntu

文章提到使用 ubuntu 為基底的 docker image for .net 的容量很小，未來也許可以評估服務打包時改用相關 image base 來降低成品容量~

不過目前縮小化的 ubuntu 原生支援 .net 的容器還在測試版本，還需要等等

以下提供目前微軟官方的容器儲存庫 (amd 64 - aspnet core runtime) 中，各 OS 版本的 image base 容量

https://mcr.microsoft.com/en-us/product/dotnet/aspnet/tags

| .net 6 container tag           | size    | os version               |
| ------------------------------ | ------- | ------------------------ |
| 6.0-jammy-amd64                | 87.3 MB | Ubuntu 22.04 LTS         |
| 6.0-bullseye-slim-amd64        | 87.6 MB | Debian 11                |
| 6.0-windowsservercore-ltsc2022 | 45.5 MB | windows server core 2022 |
| 6.0-alpine-amd64               | 45.3 MB | alpine 3.16              |
| 6.0-nanoserver-ltsc2022        | 43.9 MB | windows nano server 2022 |

Ubuntu 官方的 docker image repo 中，原生支援 .net 的 image base `目前還在測試中`

|                | container tag  | size     | os version | link                                                |
| -------------- | -------------- | -------- | ---------- | --------------------------------------------------- |
| dotnet-runtime | 6.0-22.04_edge | 48.65 MB | 22.04 LTS  | https://hub.docker.com/r/ubuntu/dotnet-runtime/tags |
| dotnet-aspnet  | 6.0-22.04_edge | 58.55 MB | 22.04 LTS  | https://hub.docker.com/r/ubuntu/dotnet-aspnet/tags  |
| dotnet-deps    | 6.0-22.04_edge | 5.3 MB   | 22.04 LTS  | https://hub.docker.com/r/ubuntu/dotnet-deps/tags    |

## docker file

https://github.com/dotnet/dotnet-docker/blob/main/samples/dotnetapp/Dockerfile.chiseled

https://github.com/dotnet/dotnet-docker/blob/main/samples/aspnetapp/Dockerfile.chiseled
