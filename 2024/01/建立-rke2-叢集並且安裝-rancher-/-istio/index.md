# 建立 RKE2 叢集並且安裝 Rancher / istio


我初期在公司建立地端 Kubernetes 叢集以進行 POC 時，都是使用 kubeadm 這套官方工具進行的 (我早期文章就是基於此撰寫的)。

但是當需要將 Kubernetes 拓展到其他單位時，那相對複雜的安裝與管理方式卻是第一個難以躍過的門檻。

因此，後續改用 RKE2 建立，並且導入 Rancher 進行 Kubernetes 叢集管理。從 2022 年末使用至今，由於我們公司的運用情境相對簡單，因此沒出現太多難以處理的問題。

這篇紀錄我的 RKE2 + Rancher 的安裝命令，主要內容都是依據 iThome Kubernetes Summit 2022 時我參與的 SUSE 工作坊講師提供的資訊進行調整，並適度的加上一些說明。

<!--more-->

# RKE2 Install

## 運作環境

{{<admonition info >}}
|         |                 |                         |
|---------|-----------------|-------------------------|
| OS      | Ubuntu 20.04    |                         |
| RKE2    | v1.24.11+rke2r1 | 應配合 Rancher 選擇版本 |
| Rancher | v2.7.9          | 應配合 RKE2 選擇版本    |
{{</admonition >}}

## First Node

### download rke2 install file

```bash
#預設不給 type 就是 server 了，所以不用特地給 type
#curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -

curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
```

### Create RKE2 Config

```bash
sudo mkdir -p /etc/rancher/rke2/
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
# 主機名稱
node-name:
  - "rke2-node-01"

# k8s 各主機溝通用的憑證簽章
tls-san:
- rke2-node-01          # node host
- rke2-node-01.local.vm # node host domain
- rke2-cluster.local.vm # k8s cluster api server domain
- 1.1.1.1               # k8s cluster api server domain 的 vip，看參考資料多半都有設定

#如果要做純 k8s master 的話可已加入以下設定，RKE2 預設是混合 NODE
#node-taint:
#  - "CriticalAddinsonly=true:NoExecute"

#以下參數所有 master node 都要相同

# 由於我的環境主要使用 istio，因此關掉 nginx 的 ingress
disable: rke2-ingress-nginx

cni:
- cilium
EOF
```

### Run Install

```bash
sudo INSTALL_RKE2_CHANNEL=v1.24.11+rke2r1 ./install.sh

export PATH=$PATH:/opt/rke2/bin
sudo systemctl enable rke2-server

#這邊執行後會要等一下
sudo systemctl start rke2-server
```

### Set Config and Kubectl

```bash
mkdir .kube
sudo cp /etc/rancher/rke2/rke2.yaml .kube/config

#如果不是用 root 登入的話，需要另外給予權限
#sudo chown rancher .kube/config

sudo cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
```

### Get rke2 token

{{<admonition info >}}
這邊是用來給其他 Master Node 用的，如果只打算建立一台 Master Node，則可以忽略此步驟
{{</admonition >}}

```bash
sudo cat /var/lib/rancher/rke2/server/token
```

## Other Master Node

{{<admonition info >}}
基本流程大致一樣，只有 config 會有些許差異，因此此節僅保留差異處
{{</admonition >}}

### Create rke2 Config

{{<admonition info >}}
config 最大的差異是在第二台以後的 master 都需要加上 token / server 等欄位
{{</admonition >}}

```bash
sudo mkdir -p /etc/rancher/rke2/
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml

token: --

#這邊 server 有人用第一台主機的 domain，有人用叢集 domain，只要能連上主要的 master 主機並使用 9345 port 即可
#port 的部分 RKE2 預設使用 9345，但是仍有開通 6443 來供一般使用
server: https://rke2-node-01.local.vm:9345

# 主機名稱
node-name:
  - "rke2-node-02"

# k8s 各主機溝通用的憑證簽章
tls-san:
- rke2-node-02          # node host
- rke2-node-02.local.vm # node host domain
- rke2-cluster.local.vm # k8s cluster api server domain
- 1.1.1.1               # k8s cluster api server domain 的 vip，看參考資料多半都有設定

#如果要做純 k8s master 的話可已加入以下設定，RKE2 預設是混合 NODE
#node-taint:
#  - "CriticalAddinsonly=true:NoExecute"

#以下參數所有 master node 都要相同

# 由於我的環境主要使用 istio，因此關掉 nginx 的 ingress
disable: rke2-ingress-nginx

cni:
- cilium
EOF
```

## uninstall

- [官方解安裝文件](https://docs.rke2.io/install/linux_uninstall/)

ubuntu
直接執行 `/usr/local/bin/rke2-uninstall.sh` 就好，輕鬆愜意，會把所有自訂設定等資料全部清空，所以設定檔記得留備份

centos
直接執行 `/usr/bin/rke2-uninstall.sh` 就好，輕鬆愜意，會把所有自訂設定等資料全部清空，所以設定檔記得留備份

# Istio / Rancher 等其他服務安裝

{{<admonition info >}}
由於後續的操作都會需要連上 kubernetes，因此請記得運用 [Set Config and Kubectl](#set-config-and-kubectl) 章節中的命令將工具與設定檔抓出來
{{</admonition >}}

## Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Helm Repo

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

## istio

{{<admonition info >}}
這邊只安裝 istio 的基礎工具，ingress 以及其他配合 rancher 的設定會跟 rancher 的安裝一起進行
{{</admonition>}}

```bash
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system --wait
helm install istiod istio/istiod -n istio-system --wait
```

## cert-manager

{{<admonition info >}}
- [rancher latest version doc](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/upgrade-cert-manager)
  > rancher 官方有說明目前最新版本對於 cert-manager 的版本測試狀況，可依據自己的需求進行相關版本調整

- 如果網路環境有硬體可以處理外部憑證，可以忽略此套件的安裝，並且在安裝 rancher 時使用外部憑證 (並搭配無憑證設定的 istio ingress)
{{</admonition>}}

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.7.1 --wait
```

## rancher

{{<admonition info >}}
這邊配合前面安裝的 RKE2 版本而使用 2.7.9 的 Rancher
{{</admonition>}}

### basic install

```bash
kubectl create namespace cattle-system
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=my-rancher.local.vm --version 2.7.9

# use external tls
# helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=my-rancher.local.vm --version 2.7.9 --set external-tls
```

### set istio

```bash
helm install istio-ingressgateway istio/gateway -n cattle-system --wait


sudo mkdir -p ~/istio-setting
cat <<EOF | sudo tee ~/istio-setting/gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: rancher-gateway
  namespace: cattle-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - hosts:
        - my-rancher.local.vm
      port:
        name: http
        number: 80
        protocol: HTTP
    # 如果是使用外部憑證的話，這邊設定可以省去或是要另外調整
    - port:
        number: 443
        name: https-443
        protocol: HTTPS
      hosts:
        - my-rancher.local.vm
      tls:
        # mode: PASSTHROUGH
        mode: SIMPLE
        credentialName: cattle-system/tls-rancher-ingress
EOF

cat <<EOF | sudo tee ~/istio-setting/virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: rancher-vs
  namespace: cattle-system
spec:
  gateways:
    - rancher-gateway
  hosts:
    - my-rancher.local.vm
  http:
    - headers:
        request:
          set:
            x-forwarded-proto: https
      route:
        - destination:
            host: rancher.cattle-system.svc.cluster.local
            port:
              number: 80

  # 如果是使用外部憑證的話，這邊設定可以省去或是要另外調整
  tls:
    - match:
      - port: 443
        sniHosts:
        - my-rancher.local.vm
      route:
      - destination:
          host: rancher.cattle-system.svc.cluster.local
          port:
            number: 443
EOF

kubectl apply -f ./

```

