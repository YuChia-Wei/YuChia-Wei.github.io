# Kubernetes 1.24 版本環境設定重點


由於 kubernetes 在 1.24 版之後已不預設支援 Docker ( [kubernetes 1.24 release page](https://kubernetes.io/blog/2022/05/03/kubernetes-1-24-release-announcement/#major-themes) )，所以照原本的安裝方式的話，在建立叢集時會出現錯誤。

但是因為原本的作法在安裝 Docker 時已經有安裝其他 kubernetes 有支援的容器套件 containerd，所以這篇主要紀錄如何去設定 containerd 來作為 Docker 的替代品，並紀錄一些跟原先的安裝參數不一樣的地方。

相關內容已更新至 [Kubernetes 安裝筆記 part.1 : 環境準備](./Part1-NodeEnv.md)~

{{<admonition info >}}
|||
|--|--|
|OS:| CentOs 7 (kernel = 3.10.1)|
|K8s:| v1.24.1 (新安裝時預設使用當下的最新版，因此此版本資訊參考用)|
{{</admonition >}}

<!--more-->

## iptable 橋接設定變化

來到 1.24 之後，第一個不一樣的環境設定舊識 iptable 的設定參數，原本舊版是

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

現在新版要改成這樣

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## containerd

在原本的環境準備方式中，可以發現安裝 docker 的時候其實有另外裝一個叫做 containerd.io 的東西，其實這是核心的容器執行元件，詳細技術細節我這邊就不說了，小弟我才疏學淺，直接看技術文件一定比我這篇筆記來的詳細。

而 1.24 版本的 kubernetes 雖然拔掉了 docker 原生支援，但是其實 kubernetes 也可以直接使用 containerd 來操作容器的 (原本的操安裝方式比較像是藉由 docker 去操作 containerd)。所以，我們現在在已經有安裝相關套件的狀況下，只要找到方式讓 kubernetes 改用 containerd 來運作就行了~

所以，接下來我們就是要開始設定 containerd 的相關參數 (可參考[官方文件](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd))

### 變更 config

path: `/etc/containerd/config.toml`

因為使用 yum 安裝 containerd 時的預設設定檔長這樣 (已忽略註解項目)

```toml
disabled_plugins = ["cri"]
```

我們需要把 `disabled_plugins` 裡面的 cri 移除，並加入 systemd 的相關控制，所以設定檔會變成這樣

```toml
disabled_plugins = []

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

{{<admonition warning >}}
這個設定檔建議使用以下命令用 containerd 內部的預設設定檔覆蓋後再去調整 `SystemdCgroup=true`
因為我一開始使用上面的設定的時候有發生 CoreDNS 起不來的問題，之後就好了，也許可以評估一下要不要換用預設設定檔去改。

```bash
# 輸出 containerd 的預設設定檔出來
containerd config default | tee /etc/containerd/config.toml

# 編輯該設定檔
vi /etc/containerd/config.toml

# 預設設定的的 SystemdCgroup 設定為 false，改成 true 就可以了
```

{{</admonition >}}

然後啟動 (重啟) containerd

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# 如果已經有設定啟用 containerd 的話就要用這個
sudo systemctl restart containerd
```

### crictl 設定調整

這個設定檔一開始不存在，需要自己建立並寫入以下內容

```bash
vi /etc/crictl.yaml
```

yaml 內容如下
```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```

## Single Node Setting

Kubernetes 1.24 之後的 single node 設定也有調整

舊的命令只要這樣下就可以了
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

新的要這樣下
```bash
# kubernetes 1.24 要改用這個
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

## 待測試、解決問題

1. 同時執行 docker & containerd (如果還想要使用 docker 的話，我不確定是否可以並行，理論上是可以，因為 docker 也是使用 containerd 來處理容器的)

## 參考資料

* 完整的 kubernetes 1.24 安裝方式 - 中文 [此網頁](https://www.etaon.link/2022/06/6/Install-Kubernetes-1.24.html)
> 我有參考這邊來確認自己的安裝方式有缺什麼東西

* 另外一篇安裝 kubernetes 1.24 & 使用 CentOs 的 [ Blog ](https://omegaatt.com/blogs/develop/2022/centos-7-kubernetes-install.html)

* 別人整理的 kubeadm 1.24 版使用 docker 的安裝方式在 [此網頁](https://www.teanote.pub/archives/503)
> 有些設定跟我不一樣，而且這篇使用的 OS 是 CentOs 8，我是用 7
> 另外，由於我改使用 containerd 來做為 kubernetes 的容器執行工具，所以這篇使用 docker 的作法也僅作為參考

