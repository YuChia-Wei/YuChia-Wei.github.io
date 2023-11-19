# Kubectl command note


Kubernetes CLI - Kubectl 的命令表

<!--more-->

kubectl 是 k8s 核心的工具，每個 node 都會需要裝一個

這邊整理出常用命令，詳細命令請參考官方網站

{{<admonition info >}}
這邊列出一些常用的完整命令或是一些範例，若有其他需求可以配合常用參數表進行組合
有些命令上有應用情境，在這邊就不會附上範例 (例如 kubectl apply / kubectl delete)
{{</admonition >}}

1. 取得現在使用的設定檔

    ```
    kubectl config current-context
    ```
1. 常用參數表

    | 參數                      | 說明                                                                                                      |
    | ------------------------- |:--------------------------------------------------------------------------------------------------------- |
    | -n `<namespace>`          | 定義現在的命令是對應**特定** namespace，取 nodes 資料時無效果                                             |
    | --all-namespaces          | 定義現在的命令是對應**所有** namespace，取 nodes 資料時無效果，在有指定特定資源時會報錯                   |
    | -A                        | 定義現在的命令是對應**所有** namespace，取 nodes 資料時無效果，在有指定特定資源時會報錯                   |
    | -o wide                   | 呈現更多資料，`kubectl describe *` 命令無法使用                                                           |
    | po `<pods name>`          | 代表 pods，沒有設定指定 pods name 的話代表所有 pods                                                       |
    | pos `<pods name>`         | 同 po `<pods name>`                                                                                       |
    | pods `<pods name>`        | 同 po `<pods name>`                                                                                       |
    | no `<Nodes name>`         | 代表 nodes，沒有設定指定 nodes name 的話代表所有 nodes 或是執行命令的本機 (看執行 get 還是 describe 命令) |
    | node `<Nodes name>`       | 同 no `<Nodes name>`                                                                                      |
    | nodes `<Nodes name>`      | 同 no `<Nodes name>`                                                                                      |
    | svc `<service name>`      | 代表 service，沒有設定指定 service name 的話代表所有 service                                              |
    | service `<service name>`  | 同 svc `<service name>`                                                                                   |
    | services `<service name>` | 同 svc `<service name>`                                                                                   |
    | all                       | 取得可以取得的所有資源 ( pods / nodes / service / ...)                                                    |
    | ns `namespace`            | 配合 get / describe 命令，可以指定要取得哪個 namespace 的資料，沒有定義名字的話代表全部，與 -n 效果不同   |

1. kubectl describe *

    **取得指定項目的詳細內容**
    **可以用來看檔案 (或網路資源) 內容**

    {{<admonition warning >}}
    `-o wide` 參數無效
    不建議使用 `all` & `-A` 參數
    {{</admonition >}}

    * 指定 pods 詳細內容
        ```bash
        kubectl describe pods <pods name> -n <namespace>
        ```
    * Node 詳細內容
      **因為 node 沒有 namespace 問題，所以 namespace 相關參數在看 node 時無效 (但不會報錯)**
      **沒有定義 node 名稱時，預設執行命令的那台機器**
        ```bash
        kubectl describe no <pods name>
        ```
    * 所有可取得的資料的詳細內容 (預設命名空間)
        ```bash
        kubectl describe all
        ```
    * 所有可取得的資料的詳細內容 (所有命名空間)
        ```bash
        kubectl describe all -A
        ```

1. kubectl get *

    **取得指定項目的資訊清單，這個命令的資料主要表現形式為表格**


    * 取得特定 namespace 的所有資訊
        ```bash
        kubectl get all -n <name space>
        ```
    * 取得所有 K8s 資料的表格
        ```bash
        kubectl get all -A
        ```
    * 取得 kube-system 這個 namespace 中的所有 pods 並列出更詳細的內容
        ```bash
        kubectl get po -n kube-system -o wide
        kubectl get pods -n kube-system -o wide
        ```

    * 列出所有 pods (不限命名空間)
        ```bash
        kubectl get pods --all-namespaces
        kubectl get pods -A
        ```
    * 取得所有服務
        ```bash
        kubectl get svc -A
        kubectl get services -A
        ```
    * 取得部屬清單
        ```bash
        kubectl get deployment -A
        ```
1. kubectl apply -f `<file or url>`
    **套用部屬設定並進行部屬**
1. kubectl delete -f `<file or url>`
    **刪除部屬設定與已部屬資源**
1. kubectl port-forward `<pods name>` `<host port>:<pods port>` -n `<namespace>`
    **定義 pode 對外的 port**
    {{<admonition warning >}}
    不建議使用
    {{</admonition >}}
1. kubectl rollout restart deploy -n `<NAMESPACE>`
    **重啟已部屬資源**
    {{<admonition info >}}
    所有命名空間的參數 `-A` 在這個命令無效
    {{</admonition >}}
    REF1: https://qvault.io/open-source/how-to-restart-all-pods-in-a-kubernetes-namespace/
    REF2: https://discuss.kubernetes.io/t/restart-all-deployment-in-a-namespace/8165/7
    * 重啟指定命名空間內的資源
        ```bash
        kubectl rollout restart deploy -n <NAMESPACE>
        ```
    * 指定部屬資料進行重啟
        {{<admonition info >}}
        部屬設定的名稱可配合 kubectl get deploymant 命令取得
        {{</admonition >}}
        ```bash
        kubectl rollout restart deployments/<name>
        ```

