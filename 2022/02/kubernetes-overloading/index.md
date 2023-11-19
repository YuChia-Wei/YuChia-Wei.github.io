# Kubernetes Overloading


Kubernetes Node 的負載相關筆記

<!--more-->

# 資源設定單位

[[Kubernetes] 分配 & 管理 container 所使用到的計算資源](https://godleon.github.io/blog/Kubernetes/k8s-Scheduling-Manage-Compute-Resource-for-Container/)

## CPU

[Kubernetes officiel Document](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu)

CPU 資源設定在沒有附帶單位的狀況下，該數字視為 CPU Core 數量；例如 1 = 1 core。

單位 1000m = 1 (core)
最小為 100m (= 0.1 core)
如果資源設定要小於 1 core 的話，建議使用 100~1000m 的數值來設定，雖然支援小數點設定 (0.5 = 500m)，但是建議讓設定格式一致，減少小數點的出現次數。

## Memory

{{<admonition info >}}
目前常見的資源設定單位都會帶有 i 後綴的版本，例如 Mi (Mebibyte)，因為相比 MB 來說，MiB 的定義更為明確；MB 的定義混亂，有時被看作 1024 KB，有時又是 1000 KB。
{{</admonition >}}

[Kubernetes officiel Document](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory)

* Mi = Mebibyte [wiki](https://zh.wikipedia.org/wiki/Mebibyte)
    > 1 MiB = 2^20 bytes = 1024 kibibytes = 1048576bytes
* M = Megabyte [wiki](https://zh.wikipedia.org/wiki/Megabyte)

# 基礎能力負載

[HPA 設定參照](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

## 在要用於測試的 node 上加入標籤

為了避免在測試時影響到運行中的服務，可以在 node 上面加上標籤，來限制壓測時要使用的 node

{{<admonition warning >}}
2022-01-04 此節僅做筆記，尚未進行實機測試
{{</admonition >}}

* 設定 Label 的命令

    ```bash
    kubectl label node <node-name> <label>=<value> --overwrite
    ```

* 列出有 `node-role.kubernetes.io/worker=ci` 這個 label 的命令
    ```bash
    kubectl top node -l node-role.kubernetes.io/worker=ci
    ```

## Kubernetes 官網紀錄的資源限制

依據 [官網文件](https://kubernetes.io/docs/setup/best-practices/cluster-large/) 紀錄，Kubernetes v1.23 中的資源負載數量如下：

* 1 Node 最多 110 Pods
    備註：代表一個 Node 中，kubelet 工具可以操作的 pod 數量
* 1 Cluster 最多 5000 Nodes
* 1 Cluster 中總計最多 150,000 個 Pods
* 1 Cluster 中總計最多 300,000 個 Docker Containers

# Istio 網格能力負載

## Istio Sidecar 資源限制

使用 `kubectl describe pod` 命令查閱 pod，可以在 `istio-proxy:` 區段發現，Istio 所初始化的 Sidecar Container 所給予的資源為

```
Limits:
  cpu:     2
  memory:  1Gi
Requests:
  cpu:        100m
  memory:     128Mi
```

## 搭配 Istio 的其他影響

{{<admonition info >}}
以下影響以預設安裝不調整參數的 Istio 為主
{{</admonition >}}

* 因為 Sidecar 的增加，每一層 Sidecar 容器會微幅增加約 `0.5ms` 的延遲
* `使用默认istio-proxy的资源配置下, 最多可以抗住 1w 左右的HTTP请求并发.`
    此項直接引述參考文件中 [isito envoy sidecar的全面性能压力测试](http://xiaorui.cc/archives/7239) 的結論內容未做修正的原因是，文內的 `1w` 代表數量未知

# Horizontal Pod Autoscaler (HPA)

HPA 可以監測負載狀況，實現自動拓展 pod 來增加系統負載能力
倚賴 metrics server

# 壓力測試工具

{{<admonition warning >}}
2022-01-04 此節僅做筆記，尚未進行實機測試
{{</admonition >}}

https://www.dotblogs.com.tw/armycoding/2021/12/02/k6-introuction


## Docker-BusyBox

[BusyBox](https://hub.docker.com/_/busybox)

啟動測試容器
```bash
kubectl run --generator=run-pod/v1 -i --tty load-generator --image=busybox /bin/sh
```
## 官方的測試工具？

[kubernetes perf-tests](https://github.com/kubernetes/perf-tests)

## 可能可以用的工具

* https://github.com/wg/wrk
* https://github.com/giltene/wrk2
* https://github.com/rakyll/hey
* https://github.com/tsenart/vegeta

# 參考資料

* [認識負載測試與 k6](https://editor.leonh.space/2021/introduction-of-load-testing-and-k6/)
* [Kubernetes 動態建立 Jenkins Agent 壓力測試 (繁中翻譯版)](https://www.gushiciku.cn/pl/p99y/zh-tw)
* [Kubernetes 動態建立 Jenkins Agent 壓力測試 (原始來源)](https://www.chenshaowen.com/blog/the-stress-test-about-kubernetes-dynamically-creates-jenkins-agent.html)
* [isito envoy sidecar的全面性能压力测试](http://xiaorui.cc/archives/7239)
* [Kubernetes系列之Kubernetes的弹性伸缩（HPA）](https://blog.51cto.com/79076431/2475855)

