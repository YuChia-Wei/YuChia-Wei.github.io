# Istio 設定 http request header 來讓內部容器使用 https


當網路架構在以下狀況時

```
Client (get `https://my.domain.com`) -> LB (A10) -> Istio Ingress (TLS Terminating) -> Service (http request schema is http)
```

由於 Service 端收到的 Http Request 中的協定可能因為憑證管理的方式或其他原因，是會被 A10 或是 Istio Ingress 給過濾掉的。

在這種情況下，可能會導致定義在服務內的一些服務出現錯誤；我實務上碰到的案例是 OAuth OpenIdConnect 的 Redirect Url 在 .Net 6 裡面其實是無法強制複寫成 https 的，加上 A10 那端有被設定強轉，使用 http 的網址並不會通，這時候就會出現錯誤。

以下提供在 Istio 中的解決方式，文章末端也有一些我覺得可能有幫助的一些資料來源讓大家參考。

<!--more-->

前面有提到，我自己碰到的狀況如下

由於憑證是設定在 LB (A10) 上面的，當 http request 經過 LB 之後就是一律使用 http (non SSL) 協定進行連線，這導致 Istio Ingress 永遠只會收到 `http://my.domain.com` 這樣的非 https 網址，再往下進入服務後，.Net 服務所收到的要求更只會進入 80 port。

而由於我公司使用 OAuth 2.0 Code Flow，在使用 OpenIdConnect 設定連線時，微軟套件是無法直接修改整段 Redirect Url 的；儘管修改了，在這樣的架構下也會認證失敗。

在經過幾天的搜尋後找到條有用的 github issue

* [istio github issue](https://github.com/istio/istio/issues/7964#issuecomment-499775214)

發現只要在 http header 上加入 `x-forwarded-proto: https` 之後，OpenIdConnect 的套件就會使用 https 去設定 redirect url！

稍做調整後的 Istio Virtual Service yaml 如下 (其他必要資料請自行調整)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: --
  labels:
    app.kubernetes.io/name: --
    app.kubernetes.io/part-of: --
  namespace: --
spec:
  gateways:
    - istioGateway
  hosts:
    - host.example.com
  http:
    - headers:
        request:
          set:
            x-forwarded-proto: https
      match:
      - uri:
          prefix: /
      route:
      - destination:
          host: service-name.namespace.svc.cluster.local
          subset: v1
          port:
            number: 8080
      retries:
        attempts: 3
        perTryTimeout: 10s
      timeout: 30s

```

設定好之後果然 OAuth 就通了！可喜可賀~ 可口可樂~~

其他過程中查到的覺得可能有幫助的資料如下

* [istio-doc](https://istio.io/latest/docs/reference/config/networking/virtual-service/#Headers)
* [其它相關設定的 istio doc 頁面](https://istio.io/latest/docs/ops/configuration/traffic-management/network-topologies/)
* [istio doc - show source ip](https://istio.io/latest/blog/2020/show-source-ip/)
* [nginx 設定參考 (尚未驗證)](https://stackoverflow.com/a/70681462)
* [微軟的相關說明-.netcore 3.1 (參考用)](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-3.1)
* [微軟的相關說明-.net 6 (參考用)](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-6.0)
* [直接從 C# 服務中用程式碼設定的做法 (尚未驗證)](https://stackoverflow.com/a/63920369)

=====

未來是希望可以將身分認證等功能全數移往 istio / kubernetes 上的 resource 上設定，降低軟體中對這些網路設定的倚賴，這感覺才有用到 istio 的功能，不然我都不知道我為什麼要使用 istio 了。

