# ArgoCD install use kustomizaion


使用 kustomization 安裝 ArgoCD，並且利用 patch 來加入 istio 所需要的 Label。

此篇文所使用的安裝設定檔連結：[Github](https://github.com/YuChia-Wei/argoproj-deploy)

<!--more-->

# ArgoCD Memo

設定檔中也包含了 `--insecure` 執行參數，以便使用 http 連線
此定義檔主要增加 Kubernetes 的 Label 設定 (app, version)，讓 Istio 可正常存取
為了方便以後的更新，有特別將各種不同類型的 patch 設定檔切割出來

* 安裝檔案說明
    |檔案|說明|
    |--|--|
    |[Kustomiztion.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/kustomization.yaml)|kustomiztion 定義檔|
    |[install-2.3.4.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/install-2.3.4.yaml)|argocd 官方 2.3.4 版本安裝檔|
    |[VirtualService.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/VirtualService.yaml)|Istio 所需的 VirtualService 服務安裝檔|
    |[Gateway.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/Gateway.yaml)|Istio 所需的 Gateway 服務安裝檔 **(不是 IngressGateway)**|
    |[istio_patches_deployment.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/istio_patches_deployment.yaml)| argocd 官方版本安裝檔的覆蓋參數，增加 Istio 所需的 Label 定義 |
    |[istio_patches_service.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/istio_patches_service.yaml)| argocd 官方版本安裝檔的覆蓋參數，增加 Istio 所需的 Label 定義 |
    |[istio_patches_statefulset.yaml](https://github.com/YuChia-Wei/argoproj-deploy/tree/main/ArgoCD/istio_patches_statefulset.yaml)| argocd 官方版本安裝檔的覆蓋參數，增加 Istio 所需的 Label 定義 |

### 安裝檔準備

1. download install yaml (option)
    * 指定版本
        ```bash=
         curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/v2.3.4/manifests/install.yaml -o install-2.3.4.yaml
        ```
    * 最新版本
        ```bash=
         curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o install-latest.yaml
        ```
1. 建立 namespace
    * 純建立
        ```bash=
        kubectl create namespace argocd
        ```
    * 含設定 istio 掛車
        ```bash=
        kubectl create namespace argocd
        kubectl label namespace argocd istio-injection=enabled --overwrite

        # 如果安裝 istio 的時候是特別設定版本的話 (通常出現在有使用金絲雀部署來更新 istio 的狀況)
        # 如果已經有設定掛車的話，可以用這方式刪掉舊的同時設定新的
        #kubectl label namespace argocd istio-injection- istio.io/rev=1-13-3
        kubectl label namespace argocd istio.io/rev=1-13-3
        ```

### When Upgrade

* 升級後不會更新設定檔，因此原始的密碼與設定都相同
* 可以先用以下命令檢查差異 (cmd 位置要先到 argocd kustomiztion 的位置)
    ```bash=
    # 查看差異
    kubectl diff -k ./

    # 將差異資料輸出到特定檔案
    kubectl diff -k ./ > different-data.txt

    # 測試執行並輸出結果
    kubectl apply -k ./ --dry-run=client > dry-run.txt

    ```

### Install

```
kubectl apply -k ./
```
