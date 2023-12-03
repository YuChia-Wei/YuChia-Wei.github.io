# Istio 附帶的 Plugin 安裝


Istio 官方安裝套件附帶的套件安裝方式，包含各服務的 Istio Ingress Gateway 設定檔。

<!--more-->

# 安裝 Istio 官方工具中附帶的服務

執行以下命令時，需確認目前 bash 所在位置為 istioctl 工具的資料夾中。

**建議安裝 prometheus 與 kiali**

* 安裝全部

```bash
kubectl apply -f samples/addons
```

* 個別安裝

  * prometheus

    ```bash
    kubectl apply -f samples/addons/prometheus.yaml
    ```

  * Kiali

    ```bash
    kubectl apply -f samples/addons/kiali.yaml
    ```

  * Grafana

    ```bash
    kubectl apply -f samples/addons/grafana.yaml
    ```

  * Jaeger

    ```bash
    kubectl apply -f samples/addons/jaeger.yaml
    ```

  * Zipkin

    **與 jaeger 為同質服務，則一安裝即可**

    ```bash
    kubectl apply -f samples/addons/extras/zipkin.yaml
    ```

* Prometheus Operator

    ```bash
    kubectl apply -f samples/addons/extras/prometheus-operator.yaml
    ```

# 各個套件的 gateway 設定

> 參考 : https://istio.io/latest/docs/tasks/observability/gateways/

* prometheus (普羅米修斯)

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
    - "prometheus.${INGRESS_DOMAIN}"
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

* Kiali

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kiali-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-kiali
      protocol: HTTP
    hosts:
    - "kiali.${INGRESS_DOMAIN}"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-vs
  namespace: istio-system
spec:
  hosts:
  - "kiali.${INGRESS_DOMAIN}"
  gateways:
  - kiali-gateway
  http:
  - route:
    - destination:
        host: kiali
        port:
          number: 20001
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: kiali
  namespace: istio-system
spec:
  host: kiali
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF
```

* Grafana

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grafana-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-grafana
      protocol: HTTP
    hosts:
    - "grafana.${INGRESS_DOMAIN}"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-vs
  namespace: istio-system
spec:
  hosts:
  - "grafana.${INGRESS_DOMAIN}"
  gateways:
  - grafana-gateway
  http:
  - route:
    - destination:
        host: grafana
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: grafana
  namespace: istio-system
spec:
  host: grafana
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

* 追蹤 (Jaeger/Zipkin)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: tracing-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http-tracing
      protocol: HTTP
    hosts:
    - "tracing.${INGRESS_DOMAIN}"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tracing-vs
  namespace: istio-system
spec:
  hosts:
  - "tracing.${INGRESS_DOMAIN}"
  gateways:
  - tracing-gateway
  http:
  - route:
    - destination:
        host: tracing
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: tracing
  namespace: istio-system
spec:
  host: tracing
  trafficPolicy:
    tls:
      mode: DISABLE
---
EOF

```

# 解除安裝

```bash
kubectl -n istio-system delete gateway grafana-gateway kiali-gateway prometheus-gateway tracing-gateway
kubectl -n istio-system delete virtualservice grafana-vs kiali-vs prometheus-vs tracing-vs
kubectl -n istio-system delete destinationrule grafana kiali prometheus tracing
```

