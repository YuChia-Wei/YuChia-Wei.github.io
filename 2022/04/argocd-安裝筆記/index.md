# ArgoCD 安裝筆記


Argo CD 安裝的紀錄

{{<admonition warning>}}
新版安裝資料待整理，本文版本為 ver 2.2.1
{{</admonition>}}

<!--more-->

# ArgoCD Install Memo

## Manual Install

{{<admonition info >}}
Kubernetes with Istio 的環境下建議參照 [kustomization install](#kustomization-Install-With-Istio) 的方式進行安裝
{{</admonition >}}

1. download install yaml (option)
    * 指定版本
        ```bash
         curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/v2.2.1/manifests/install.yaml -o install-2.2.1.yaml
        ```
    * 最新版本
        ```bash
         curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o install-latest.yaml
        ```
1. 建立 namespace
    * 純建立
        ```bash
        kubectl create namespace argocd
        ```
    * 含設定 istio 掛車
        ```bash
        kubectl create namespace argocd
        kubectl label namespace argocd istio-injection=enabled --overwrite
        ```
1. Install to K8s
    * 使用下載版 (指定版本)
        ```bash
        kubectl apply -n argocd -f install-2.2.1.yaml
        ```
    * 使用下載版 (最新版本)
        ```bash
        kubectl apply -n argocd -f install-latest.yaml
        ```
    * 使用線上指定版
        ```bash
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.2.1/manifests/install.yaml
        ```
    * 使用線上最新版
        ```bash
        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
        ```
1. Install Argo Cli (option)
    ```bash
    curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    chmod +x /usr/local/bin/argocd
    ```
1. Fix Https Problam

    [Reference](https://github.com/speedwing/eks-argocd-bootstrap/blob/master/argocd-bootstrap/argocd-istio-bootstrap/argocd-server.yaml)

    * 編輯發布設定
        ```bash
        kubectl edit deploy argocd-server -n argocd
        ```
    * 找到 spec.containers.command 區段並加入以下內容
        ```yaml
        spec:
          containers:
            - name: argocd-server
              command:
                - argocd-server
                # 加入以下五行，使用 http 連線
                - --staticassets
                - /shared/app
                - --repo-server
                - argocd-repo-server:8081
                - --insecure
        ```
    * 重新佈署
        ```bash
        kubectl rollout restart deploy -n argocd
        ```
## Helm Install

[Official Argo CD Helm Readme](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd)

重要：安裝時設定 server.extraArgs={--insecure} 來避開 TLS 憑證，如果要使用 TLS 憑證的話要多一些憑證設定，這邊先略過

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd
kubectl create namespace argocd
kubectl label namespace argocd istio-injection=enabled --overwrite
helm install argocd argo/argo-cd -n argocd --set server.extraArgs={--insecure}
```

## Ingress Install
### Use Specific ArgoCD Domain
1. nginx Ingress
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-server-ingress
      namespace: argocd
      annotations:
        # kubernetes.io/ingress.class: nginx
        # nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
        # nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    spec:
      ingressClassName: nginx
      rules:
      - host: argocd.<domain>
        http:
          paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
    ```

1. Istio Ingress

    [Reference](https://github.com/speedwing/eks-argocd-bootstrap/blob/master/applications/argocd-istio-app/templates/istio-resources.yaml)

    * Gateway yaml
        ```yaml
        apiVersion: networking.istio.io/v1beta1
        kind: Gateway
        metadata:
          name: argocd-gateway
          namespace: argocd
        spec:
          selector:
            istio: ingressgateway
          servers:
            - hosts:
                - argocd.<domain>
              port:
                name: http
                # port 要用 ingressgateway 中，port = 80 的那組設定的 TargetPort
                number: 8080
                protocol: HTTP
              # 如果要強轉 HTTPS
              # tls:
              #   httpsRedirect: true
            - hosts:
                # 這邊這樣設定只是想讓 istio 不會跳相同 host 的警告
                - argocd/argocd.<domain>
              port:
                name: https
                # port 要用 ingressgateway 中，port = 443 的那組設定的 TargetPort
                number: 8443
                protocol: HTTPS
              tls:
                mode: PASSTHROUGH
              # 另外的設定方法，待測試
              # tls:
              #   credentialName: argocd-server-tls # argocd server 會自動讀取這個名稱的 k8s secrets
              #   mode: SIMPLE
        ```
    * VirtualService yaml
        {{<admonition info >}}
        含有 https 導流的設定方式要研究一下
        {{</admonition >}}
        ```yaml
        apiVersion: networking.istio.io/v1beta1
        kind: VirtualService
        metadata:
          name: argocd-vs
          namespace: argocd
        spec:
          hosts:
          - argocd.<domain>
          gateways:
          - argocd-gateway
          http:
          - route:
            - destination:
                host: argocd-server.argocd.svc.cluster.local
                port:
                  number: 80
        ```
### Use Relative ArgoCD Path

{{<admonition warning >}}
暫時測不到 `<domain>/argocd/` 這種相對路徑的 URL 設定
{{</admonition >}}

1. nginx Ingress

    **待補**

1. Istio Ingress

    * Gateway yaml
        ```yaml
        apiVersion: networking.istio.io/v1beta1
        kind: Gateway
        metadata:
          name: argocd-gateway
          namespace: argocd
        spec:
          selector:
            istio: ingressgateway
          servers:
            - hosts:
                - <domain>
              port:
                name: http
                # port 要用 ingressgateway 中，port = 80 的那組設定的 TargetPort
                number: 8080
                protocol: HTTP
              # 如果要強轉 HTTPS
              # tls:
              #   httpsRedirect: true
            - hosts:
                # 這邊這樣設定只是想讓 istio 不會跳相同 host 的警告
                - argocd/<domain>
              port:
                name: https
                # port 要用 ingressgateway 中，port = 443 的那組設定的 TargetPort
                number: 443
                protocol: HTTPS
              tls:
                mode: PASSTHROUGH
              # 另外的設定方法，待測試
              # tls:
              #   credentialName: argocd-server-tls # argocd server 會自動讀取這個名稱的 k8s secrets
              #   mode: SIMPLE
        ```
    * VirtualService yaml
        {{<admonition info >}}
        含有 https 導流的設定方式要研究一下
        {{</admonition >}}
        ```yaml
        apiVersion: networking.istio.io/v1beta1
        kind: VirtualService
        metadata:
          name: argocd-vs
          namespace: argocd
        spec:
          hosts:
          - <domain>
          gateways:
          - argocd-gateway
          http:
          - route:
            - destination:
                host: argocd-server.argocd.svc.cluster.local
                port:
                  number: 80
        ```

## kustomization Install With Istio (Recommend when use istio)

此方法主要參考以下三個連結

* [說明文件](https://loadbalancing.se/2021/03/22/argocd-behind-istio-on-rancher/)
* [官方 issues](https://github.com/argoproj/argo-cd/issues/2784)
* [第三方的主要 GitRepo](https://github.com/epacke/argo-istio)

{{<admonition info >}}
這邊的 kustomization yaml 同時設定了 [argoproj-lab](https://github.com/argoproj-labs/argocd-extensions) 的擴充
{{</admonition >}}

1. 預先準備
    * 依據 [Manual Command Install](#Manual-Command-Install) 章節準備好官方 ArgoCD 安裝 Yaml 與 Kubernetes Namespace
    * 依據 [Ingress Install](#Ingress-Install) 章節準備好 Istio 使用的 VisualService 與 Gateway
2. 準備 kustomization yaml
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    namespace: argocd
    resources:
      - install-2.2.1.yaml
      - VirtualService.yaml
      - Gateway.yaml
    patchesStrategicMerge:
      - istio_patches.yaml
    components:
      # extensions controller component
      - https://github.com/argoproj-labs/argocd-extensions/manifests
    ```
3. 準備 istio patch yaml

    {{<admonition info >}}
    這邊準備的 yaml 除了預先設定排除 https 之外，就是要設定每個部屬出去的組件的版本號與應用程式名稱，以便 Istio 可以正確追蹤；在使用 [Manual Command Install](#Manual-Command-Install) + [Ingress Install](#Ingress-Install) 的方式安裝時，Istio 會發出因為沒有版本號與應用程式的 Label 而無法追蹤的錯誤。
    {{</admonition >}}

    {{<admonition warning >}}
    此處提供的 yaml 內容須依據實際安裝的版本去修改版本號，例如在本文件中使用的 Argo CD 版本為 v2.2.1 版，就要將相關版本設定為 v2.2.1。而 Redis 與其他套件的部分，建議參考官方安裝文件中使用的版號進行設定，盡可能使 label 中的版號與實際使用的套件版本相符。
    {{</admonition >}}

    <details>

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app.kubernetes.io/component: server
        app.kubernetes.io/name: argocd-server
        app.kubernetes.io/part-of: argocd
        app: argocd-server
        version: v2.2.1
      name: argocd-server
    spec:
      template:
        spec:
          containers:
          - name: argocd-server
            command:
            - argocd-server
            - --staticassets
            - /shared/app
            - --insecure
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app.kubernetes.io/component: repo-server
        app.kubernetes.io/name: argocd-repo-server
        app.kubernetes.io/part-of: argocd
        app: argocd-repo-server
        version: v2.2.1
      name: argocd-repo-server
    spec:
      template:
        metadata:
          labels:
            app: argocd-repo-server
            version: v2.2.1
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app.kubernetes.io/component: redis
        app.kubernetes.io/name: argocd-redis
        app.kubernetes.io/part-of: argocd
        app: argocd-redis
        version: v6.2.4
      name: argocd-redis
    spec:
      template:
        metadata:
          labels:
            app: argocd-redis
            version: v6.2.4
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app.kubernetes.io/component: dex-server
        app.kubernetes.io/name: argocd-dex-server
        app.kubernetes.io/part-of: argocd
        app: argocd-dex-server
        version: v2.30.0
      name: argocd-dex-server
    spec:
      template:
        metadata:
          labels:
            app: argocd-dex-server
            version: v2.30.0
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      labels:
        app.kubernetes.io/component: application-controller
        app.kubernetes.io/name: argocd-application-controller
        app.kubernetes.io/part-of: argocd
        app: argocd-application-controller
        version: v2.2.1
      name: argocd-application-controller
    spec:
      template:
        metadata:
          labels:
            app: argocd-application-controller
            version: v2.2.1
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: argocd-dex-server
    spec:
      # expose container ports to istio network
      ports:
      - name: http
        port: 5556
        protocol: TCP
        targetPort: 5556
      - name: http-grpc
        port: 5557
        protocol: TCP
        targetPort: 5557
      - name: http-metrics
        port: 5558
        protocol: TCP
        targetPort: 5558

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: argocd-metrics
    spec:
      # expose container ports to istio network
      ports:
      - name: http-metrics
        port: 8082
        protocol: TCP
        targetPort: 8082
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: argocd-repo-server
    spec:
      # expose container ports to istio network
      ports:
      - name: https-server
        port: 8081
        protocol: TCP
        targetPort: 8081
      - name: http-metrics
        port: 8084
        protocol: TCP
        targetPort: 8084
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: argocd-server-metrics
    spec:
      # expose container ports to istio network
      ports:
      - name: http-metrics
        port: 8083
        protocol: TCP
        targetPort: 8083
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: argocd-server
    spec:
      ports:
      - name: http-argocd-server
        port: 80
        protocol: TCP
        targetPort: 8080
      # delete https port
      - port: 443
        $patch: delete
    ```

    </details>

4. 安裝
    {{<admonition warning >}}
    建議前述準備的 yaml 檔案統一放在專門的資料夾
    {{</admonition >}}

    ```bash
    kubectl apply -k ./
    ```


# CLI Tool

[官方文件](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

{{<admonition warning >}}
CLI 不需要安裝在 ArgoCD 所在的叢集主機 (或 kubernetes master server)，但是有些 CLI 命令倚賴 kube config 來取得叢集資料，如果 CLI 裝在沒有 kube config 的環境時，有些命令會出錯
{{</admonition >}}

## Install - Linux

* Download Latest with curl (Linux)
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
* Download Concrete Version with curl (Linux)
```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.2.1/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

# Manage
## Remote Cluster

1. 參考 [Kubernetes Multi-Cluster](https://hackmd.io/@YuChia/Kubernetes-Multi-Cluster) 中的說明，調整 ArgoCD 所在的網路叢集的 Kubernetes config，加入外部叢集資訊
2. 確認外部叢集的名稱以供後續 ArgoCD 加入外部叢集時使用
    ```bash
    kubectl config get-contexts -o name
    ```
3. 使用 ArgoCD CLI 登入 ArgoCD
    ```bash
    argocd login <ARGOCD_SERVER>
    ```
4. 加入外部叢集
    ```bash
    argocd cluster add <remote-cluster name>
    ```
5. 現在可以在 UI 中看到外部叢集了
6. CLI 參考
    ```bash
    # List all known clusters in JSON format:
    argocd cluster list -o json

    # Add a target cluster configuration to ArgoCD. The context must exist in your kubectl config:
    argocd cluster add <cluster name>

    # Get specific details about a cluster in plain text (wide) format:
    argocd cluster get <cluster name> -o wide

    # Remove a target cluster context from ArgoCD
    argocd cluster rm <cluster name>
    ```

## User

1. Get Default Admin Password
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```

{{<admonition warning >}}
不完整
{{</admonition >}}

1. 使用官方提供的命令取得管理員密碼，登入系統後應修改密碼
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```

1. 編輯 ConfigMap
    ```bash
    kubectl edit cm argocd-cm -n argocd
    ```
2. 找到 data 區段，加入使用者
    ```yaml
    data:
      accounts.<user>: apiKey,login # 加入這個
      application.instanceLabelKey: argocd.argoproj.io/instance
      url: <argocd server url> # 這個可以順便改一下
    ```
3. 到 \<argocd server url>/settings/accounts 建立登入的 Token (密碼的更新方式待測試)

# Extension Install

## Core Install

[官方擴充套件](https://github.com/argoproj-labs/argocd-extensions) 建議使用 Kustomization 方式安裝，同時，安裝時會需要使用到 Git，請確認系統內已安裝 Git。

kustomization yaml (Base on [kustomization Install With Istio](#kustomization-Install-With-Istio))
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - install-2.2.1.yaml
  - VirtualService.yaml
  - Gateway.yaml
patchesStrategicMerge:
  - istio_patches.yaml
components:
  # extensions controller component
  - https://github.com/argoproj-labs/argocd-extensions/manifests
```

## Rollout-extension install

Argo Rollout dashboard into the Argo CD Web UI.

* download yaml
```bash
curl https://raw.githubusercontent.com/argoproj-labs/rollout-extension/v0.1.0/manifests/install.yaml -o argocd-rollout-extension.yaml
```
* apply
```bash
kubectl apply -n argocd -f argocd-rollout-extension.yaml
```

# Reference

[Argo Cd Official Startup](https://argo-cd.readthedocs.io/en/stable/getting_started/)
[Argo Cd Official TLS Configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/tls/)
[Argo Cd Official Startup-Github](https://github.com/argoproj/argo-cd/blob/master/docs/getting_started.md)
[Argo Cd Github Release Page](https://github.com/argoproj/argo-cd/releases)

