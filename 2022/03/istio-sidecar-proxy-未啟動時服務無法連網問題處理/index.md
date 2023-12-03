# Istio Sidecar Proxy 未啟動時服務無法連網問題處理


前段時間在工作上碰到服務部署後沒辦法連上網路，但是容器重啟後就一切正常。

在查過資料後發現原來是因為 Istio Sidecar Proxy 尚未完成啟動時，後面的服務會無法連上網路的問題，這篇記錄解決辦法。

<!--more-->

# 更新 Istio 的 ConfigMap

在 Istio Config 中的 mesh 設定中加入以下設定

```yaml
holdApplicationUntilProxyStarts: true
```

最後加完預設的內容應該會長這樣

```yaml
defaultConfig:
  holdApplicationUntilProxyStarts: true
  discoveryAddress: istiod.istio-system.svc:15012
  proxyMetadata: {}
  tracing:
    zipkin:
      address: zipkin.istio-system:9411
enablePrometheusMerge: true
rootNamespace: istio-system
trustDomain: cluster.local
```

# Reference

* [stackoverflow:connection-refused-error-in-outbound-request-in-k8s-app-container-istio](https://stackoverflow.com/questions/63481969/connection-refused-error-in-outbound-request-in-k8s-app-container-istio)
* [Istio Github Issues#33911](https://github.com/istio/istio/issues/33911#issuecomment-877010732)
* [Istio Github Issues#11130](https://github.com/istio/istio/issues/11130)
* [知乎-关于Istio Proxy生命周期的两点注意事项](https://zhuanlan.zhihu.com/p/369301902)

