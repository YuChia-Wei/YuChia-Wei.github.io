# Kubernetes Metrics Install


要讓 Kubernetes 中的 HPA(horizontal-pod-autoscaler) 起作用，就必須安裝負載監測服務，這邊整理了 Metrics-Server 與 Prometheus-Adapter 的安裝方式。

<!--more-->

# Prometheus-Adapter

1. Prometheus Install

    1. Use Istioctl addon
        在搭配 istio 進行建置的 Kubernetes 的環境下，由於 istio 官方的安裝工具中有提供配合 istio 的 Prometheus 安裝設定，方便起見，可直接使用 istio 提供的安裝設定進行安裝。

        若沒有使用 istio，則建議統一使用 Helm 的方式安裝 Prometheus / Prometheus-Adapter。

        > 以下命令的紀錄已預設您目前的 bash 位置位於 istio 官方套件的資料夾中；即 istioctl 的下載資料夾

        ```bash
        kubectl apply -f samples/addons/prometheus.yaml
        ```

    2. Use Helm

        * 新增 prometheus 的 helm charts repo

            ```bash
            helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
            helm repo add stable https://kubernetes-charts.storage.googleapis.com/
            helm repo update
            ```
        * Install

            以下命令會在 monitoring 這個 namespace 中安裝 prometheus

            ```bash
            helm -n monitoring install prometheus prometheus-community/prometheus
            ```

    3. Prometheus 服務網頁的 istio gateway (選擇性)

        [istio 官方參考資料](https://istio.io/latest/docs/tasks/observability/gateways/)

        {{<admonition info >}} 這邊是建立進入 Prometheus 網頁的 istio 相關服務，非必要
        {{</admonition >}}

        ```bash
        cat <<EOF | kubectl apply -f -
        apiVersion: networking.istio.io/v1alpha3
        kind: Gateway
        metadata:
        name: prometheus-gateway
        namespace: istio-system
        spec:
        selector:
            istio: ingressgateway
        servers:
        - port:
            number: 80
            name: http-prom
            protocol: HTTP
            hosts:
            - "<your domain>"
        ---
        apiVersion: networking.istio.io/v1alpha3
        kind: VirtualService
        metadata:
        name: prometheus-vs
        namespace: istio-system
        spec:
        hosts:
        - "prometheus.${INGRESS_DOMAIN}"
        gateways:
        - prometheus-gateway
        http:
        - route:
            - destination:
                host: prometheus
                port:
                number: 9090
        ---
        apiVersion: networking.istio.io/v1alpha3
        kind: DestinationRule
        metadata:
        name: prometheus
        namespace: istio-system
        spec:
        host: prometheus
        trafficPolicy:
            tls:
            mode: DISABLE
        ---
        EOF
        ```

2. Prometheus-Adapter Install

    1. 準備安裝要用的 Helm Values yaml (named prometheus-adapter-values.yaml)
        * [Yaml 原始參考資料](https://github.com/devopsbox-io/example-istio-hpa/blob/main/prometheus-adapter-values.yaml)
        * [rules 區塊設定內容參考資料](https://www.nutanix.dev/2021/11/17/karbon-and-metrics-api-a-practical-guide/)

        ```yaml
        prometheus:
          # 這邊設定 kubernetes 中 prometheus 服務的位置
          # 由於我使用的是 istio 內附的安裝檔，所以連結為 istio 的連結
          # url 格式為 http://{kube-service-name}.{namespace}.svc.cluster.{cluster-name}
          url: http://prometheus.istio-system.svc.cluster.local
          port: 9090
        rules:
          default: true
          resource:
            cpu:
              containerQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, container!=""}[5m])) by (<<.GroupBy>>)
              nodeQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, id='/'}[5m])) by (<<.GroupBy>>)
              resources:
                overrides:
                  node:
                    resource: node
                  namespace:
                    resource: namespace
                  pod:
                    resource: pod
              containerLabel: container
            memory:
              containerQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>, container!=""}) by (<<.GroupBy>>)
              nodeQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>,id='/'}) by (<<.GroupBy>>)
              resources:
                overrides:
                  node:
                    resource: node
                  namespace:
                    resource: namespace
                  pod:
                    resource: pod
              containerLabel: container
            window: 5m
          custom:
          - seriesQuery: '{__name__=~"istio_requests_total"}'
            seriesFilters: []
            resources:
              overrides:
                kubernetes_namespace:
                  resource: namespace
                kubernetes_pod_name:
                  resource: pod
                destination_service_name:
                  resource: service
            name:
              matches: "^(.*)_total"
              as: "${1}_per_min"
            metricsQuery: sum(increase(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
        ```

    2. 使用 helm cli 安裝

        {{<admonition warning >}}
        我的測試環境是搭配 ArgoCD 設定 Prometheus 的 Helm Repo，並在設定 Application 時進行以下變更
        1. 指定使用預設 values
        2. prometheus url 是使用 ArgoCD WebUI 去獨立調整，並沒有設定在自訂 Values yaml 中
        3. 自訂 Values yaml 的部分僅有 rules: 以下內容

        由於不是使用 CLI 作業，我不保證我這邊提供的命令是可用的
        {{</admonition >}}

        ```bash
        helm -n istio-system install prometheus-adapter prometheus-community/prometheus-adapter -f prometheus-adapter-values.yaml
        ```

    3. 等待部屬完畢之後 HPA 中的 unknow 應該就都會變成實際的資源使用量了

    4. helm 安裝時使用的自訂 Yaml 檔案中的 rule 說明

        * 版本 1

            [參考網址]( https://github.com/devopsbox-io/example-istio-hpa )

            ```yaml
            rules:
              custom:
              - seriesQuery: 'istio_requests_total{kubernetes_namespace!="",kubernetes_pod_name!=""}'
                resources:
                  overrides:
                    kubernetes_namespace: {resource: "namespace"}
                    kubernetes_pod_name: {resource: "pod"}
                name:
                  matches: "^(.*)_total"
                  as: "${1}_per_second"
                metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
            ```

        * 版本 2

            [參考網址](https://xisheng.vip/blog/kubernetes/task/HPA.html#%E4%BA%8C%EF%BC%89customer-metric-hpa)

            ```yaml
            rules:
              custom:
              - seriesQuery: '{__name__=~"istio_requests_total"}'
                seriesFilters: []
                resources:
                  overrides:
                    kubernetes_namespace:
                      resource: namespace
                    kubernetes_pod_name:
                      resource: pod
                    destination_service_name:
                      resource: service
                name:
                  matches: "^(.*)_total"
                  as: "${1}_per_min"
                metricsQuery: sum(increase(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
            ```

        * 版本 3 (建議)

            {{<admonition info >}}
            建議使用此版本，這個版本除了建立自訂定義的 metrics 資源外，也還原了原本 metrics-server 才會有的 cpu / memory 等資源
            使用此版本之後，預設的 istio-ingressgateway / istiod 等 istio 原生 HPA 才會正常；若不使用此版本的定義檔，就必須回頭調整 istio 的設定，將 HPA 的資源改讀自定義資源
            {{</admonition >}}

            [參考網址](https://www.nutanix.dev/2021/11/17/karbon-and-metrics-api-a-practical-guide/)

            ```yaml
            rules:
              default: true
              resource:
                cpu:
                  containerQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, container!=""}[5m])) by (<<.GroupBy>>)
                  nodeQuery: sum(rate(container_cpu_usage_seconds_total{<<.LabelMatchers>>, id='/'}[5m])) by (<<.GroupBy>>)
                  resources:
                    overrides:
                      node:
                        resource: node
                      namespace:
                        resource: namespace
                      pod:
                        resource: pod
                  containerLabel: container
                memory:
                  containerQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>, container!=""}) by (<<.GroupBy>>)
                  nodeQuery: sum(container_memory_working_set_bytes{<<.LabelMatchers>>,id='/'}) by (<<.GroupBy>>)
                  resources:
                    overrides:
                      node:
                        resource: node
                      namespace:
                        resource: namespace
                      pod:
                        resource: pod
                  containerLabel: container
                window: 5m
              custom:
              - seriesQuery: '{__name__=~"istio_requests_total"}'
                seriesFilters: []
                resources:
                  overrides:
                    kubernetes_namespace:
                      resource: namespace
                    kubernetes_pod_name:
                      resource: pod
                    destination_service_name:
                      resource: service
                name:
                  matches: "^(.*)_total"
                  as: "${1}_per_min"
                metricsQuery: sum(increase(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
            ```


3. 安裝後測試

    安裝好 Prometheus-Adapter 之後，可以使用以下指令取得自訂資源值
    如果有套用前面提供的 istio 相關資源檔設定，就可以在輸出中找到 istio 相關的資源屬性，至於名稱就是依據用哪一個設定擋而有所不同 ( `*_per_second` or `*_per_min` )

    ```bash
    kubectl get --raw /apis/metrics.k8s.io/v1beta1
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
    ```

    接下來再去 HPA 設定中調整參數，targetAverageValue 的數值則是依據需要進行調整
    ```yaml
    metrics:
    - type: Pods
      pods:
        metricName: istio_requests_per_min
        targetAverageValue: '100'
    ```

    用硬體資源判斷
    ```yaml
    metrics:
    - type: Pods
      pods:
        metricName: cpu_usage
        targetAverageValue: '100m'
    ```

    其他的設定參考
    * https://blog.csdn.net/u010918487/article/details/119052933
    * https://docs.microsoft.com/en-us/answers/questions/457334/horizontal-pod-autoscaler-is-unable-to-fetch-metri.html
    * https://medium.com/cloudzone/autoscaling-kubernetes-workloads-with-istio-metrics-92f86baabba9
    * https://dustinspecker.com/posts/scaling-kubernetes-pods-prometheus-metrics/
    * https://imroc.cc/k8s/best-practice/custom-metrics-hpa/

    Rules 的定義資料
    * https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config.md

    [HPA 設定檔版本差異參考](https://stackoverflow.com/questions/54375276/scale-deployment-based-on-custom-metric?rq=1)

## Use Metrics-Server

[Metrics-server 官方 GitHub](https://github.com/kubernetes-sigs/metrics-server)

* 前置作業 - TLS Issue
    [metrics-server use https](https://kubernetes.io/zh/docs/setup/production-environment/tools/_print/#%E6%97%A0%E6%B3%95%E5%9C%A8-kubeadm-%E9%9B%86%E7%BE%A4%E4%B8%AD%E5%AE%89%E5%85%A8%E5%9C%B0%E4%BD%BF%E7%94%A8-metrics-server)
    [k8s 啟用證書管理](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs)
    [metrics-server github faq](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md#how-to-run-metrics-server-securely)

    1. Edit Kubelet Config
        * 使用命令確認 kubelet-config 的名稱
            ```
            kubectl get cm -n kube-system
            ```

            執行後應該可以看到 `kubelet-config-<version>` 的設定名稱
        * 編輯 kubelet-config
            ```
            kubectl edit cm kubelet-config-<version> -n kube-system
            ```
        * 在 data 區段最下方加入這行 (或確認這行有沒有存在)
            ```
            serverTLSBootstrap: true
            ```
        * 執行 :wq 存檔
    2. 調整每個 Node 中的 kubelet 設定檔
        * 編輯設定檔
            ```
            vim /var/lib/kubelet/config.yaml
            ```
        * 在 data 區段最下方加入這行 (或確認這行有沒有存在)
            ```
            serverTLSBootstrap: true
            ```
        * 執行 :wq 存檔
    3. Restart kubelet (每個 NODE (Master/sub) 都要執行)
        ```
        systemctl restart kubelet
        ```
    4. 使用命令確認憑證狀態
        ```
        kubectl get csr
        ```
    5. 批准前一步取得的憑證清單中，尚未認證的項目
        ```
        kubectl certificate approve <CSR-名稱>
        ```

    補充-列出現有憑證
    ```
    kubeadm certs check-expiration
    ```
    補充-更新憑證
    ```
    kubeadm certs renew
    ```
    補充-自動更新 kubelet 憑證的工具
    [kubelet rubber stamp](https://github.com/kontena/kubelet-rubber-stamp)

    {{<admonition info >}}
    kubeadm 安裝時預設有設定自動更新憑證，但不確定是否可自動更新 kubelet 的憑證，待確認
    {{</admonition >}}

* Use Helm Install

    1. Add Metrics-Server Helm Repo
        ```bash
        helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
        ```
    2. Install
        **若未給 namespace 參數，會被安裝在 default 之中，建議安裝到 Kube-system**
        ```bash
        helm upgrade --install metrics-server metrics-server/metrics-server -n kube-system
        ```
    3. Edit metrics-server deploy setting
        * command
            ```
            kubectl edit deploy -n kube-system metrics-server
            ```
        * 調整內容
            [metrics-server depoly 設定參考](https://stackoverflow.com/a/57199534)
            ```yaml
            spec:
              containers:
              - args:
                - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
                #下面這行如果沒有處理前置步驟那邊的證書設定的話會需要
                #- --kubelet-insecure-tls
                - --metric-resolution=30s
                image: k8s.gcr.io/metrics-server-amd64:v0.3.3
                imagePullPolicy: Always
                name: metrics-server
                resources: {}
                terminationMessagePath: /dev/termination-log
                terminationMessagePolicy: File
                volumeMounts:
                - mountPath: /tmp
                  name: tmp-dir
              dnsPolicy: ClusterFirst
              #下面這行很重要
              hostNetwork: true
            ```
* Proxy 問題

    如果更新完前述內容遲遲無法看到 HPA 顯示資源使用狀況的話，需檢視 Proxy 設定

    建議 NO_PROXY 要設定這些

    * Linux 設定 NO_PROXY
        ```bash
        export NO_PROXY=istio-sidecar-injector.istio-system.svc,.svc,172.30.0.0/16,localhost,10.0.0.0/8
        ```
    * kube.apiserver 設定 NO_PROXY
        ```bash
        vim /etc/kubernetes/manifests/kube-apiserver.yaml
        ```
        編輯 env: 區段
        ```yaml
        env:
            - name: HTTPS_PROXY
              value: <Proxy Server>
            - name: NO_PROXY
              value: istio-sidecar-injector.istio-system.svc,.svc,localhost,10.0.0.0/8
        ```


## REF
* [github devopsbox-io/example-istio-hpa](https://github.com/devopsbox-io/example-istio-hpa)
* [k8s安装metrics-server](https://www.cnblogs.com/lfl17718347843/p/14283796.html)
* [deckhouse.io/prometheus-metrics-adapter](https://deckhouse.io/en/documentation/v1/modules/301-prometheus-metrics-adapter/usage.html)

