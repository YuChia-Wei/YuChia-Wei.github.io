# Istio Install - Use Helm


使用 Helm 安裝 Istio

<!--more-->

# 官方參考資料

* 用官方工具安裝 : https://istio.io/latest/docs/setup/install/istioctl/
* 用 HELM 安裝 : https://istio.io/latest/docs/setup/install/helm/

# Use HELM

{{<admonition warning >}}
2022-01-10
OS: CentOS 7 - Kernel: 3.10
Helm 中的 istio gateway 版本安裝再 kernel = 3.10 的 linux 上面會發生以下問題，建議使用 istioctl 工具安裝
> container init caused "open /proc/sys/net/ipv4/ip_unprivileged_port_start: no such file or directory"
{{</admonition >}}

* 更新 HELM 資料
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```
* 建立 istio namespace
```bash
kubectl create namespace istio-system
```
* 安裝核心
```bash
helm install istio-base istio/base -n istio-system
```
* 安裝 istio discovery chart
```bash
helm install istiod istio/istiod -n istio-system --wait
```
* 安裝 Istio Ingress Gateway (做為主要的系統進入點，替代 k8s ingress gateway)
```bash
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress --wait
```

{{<admonition info >}}
如果在安裝 Istio Ingress Gateway 出現下列訊息
> Error: INSTALLATION FAILED: timed out waiting for the condition

請移除 namespace istio-ingress 上面的 istio-injection=enabled label 後再重新執行以下命令

```bash
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress --wait
```
{{</admonition >}}

# Set SiteCar
```bash
kubectl label namespace default istio-injection=enabled --overwrite
kubectl get namespace -L istio-injection
```

{{<admonition warning >}}
如果車掛不上去，或是掛上去之後發現無法部屬，請檢查 kube-apiserver 這個 pod 的環境變數
當 api server 有掛 proxy 的話，會無法掛車

REF (SCH): https://istio.io/latest/zh/docs/ops/common-problems/injection/#automatic-sidecar-injection-fails-if-the-Kubernetes-API-server-has-proxy-settings
REF (ENG): https://istio.io/latest/docs/ops/common-problems/injection/#automatic-sidecar-injection-fails-if-the-kubernetes-api-server-has-proxy-settings

有兩種解法 (建議方法 2) ：

重啟方式的參考 : https://stackoverflow.com/a/51763158

1. 調整環境變數並重啟 (刪除) pods/kube-apiserver，讓 apiserver 套用設定
    * 設定環境變數
    ```bash
    export NO_PROXY=NO_PROXY=istio-sidecar-injector.istio-system.svc,.svc
    ```
    * 刪除 kube-apiserver (因為 api server 是由 k8s 系統管理，刪除後會自動重建)
    ```bash
    kubectl delete po kube-apiserver-<node name> -n kube-system
    ```
    * 如果有定義開機 (登入) 後自動建立 PROXY 的環境變數，將 NO_PROXY 設定加入
    ```bash
    vim ~/.bash_profile

    # 把 NO_PROXY 設定檔加到環境變數
    export NO_PROXY=NO_PROXY=istio-sidecar-injector.istio-system.svc,.svc

    source ~/.bash_profile
    ```

2. 編輯 api server 的設定檔後重啟
    * 開啟設定檔
    ```
    vim /etc/kubernetes/manifests/kube-apiserver.yaml
    ```
    * 找到 env 節點並加入 NO_PROXY 設定
    ```yaml
    env:
    - name: HTTPS_PROXY
      value: <your proxy>
    - name: NO_PROXY
      value: istio-sidecar-injector.istio-system.svc,.svc
    ```
    * 刪除 kube-apiserver (因為 api server 是由 k8s 系統管理，刪除後會自動重建)
    ```
    kubectl delete po kube-apiserver-<node name> -n kube-system
    ```
{{</admonition >}}

# 解除安裝 Istio

```bash
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system --wait
kubectl delete namespace istio-system
```

