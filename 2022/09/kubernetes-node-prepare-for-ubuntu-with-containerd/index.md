# Kubernetes 環境準備 - ubuntu & containerd


kubernetes 安裝筆記

{{<admonition info>}}
|             |                         |
|-------------|-------------------------|
| K8s:        | v1.25.1                 |
| OS:         | ubuntu server 20.04 LTS |
| cri:        | containerd 1.6.8        |
| Update Time | 2022/09/17              |
{{</admonition>}}

<!--more-->



# Ubuntu 根目錄磁區

由於先前主要習慣是使用 CentOS7，在磁區格式上慣用的是 xfs 格式，所以我有特別調整根目錄磁區的格式化類型為 xfs，而非 ubuntu 預設的 ext4

我這邊另外收集了一些網路上針對 xfs / ext4 兩者的比較資訊，供需要的人參考評估

- [StackExchange 上的一個針對 ubuntu 選擇 ext4 作為預設格式的原因回應](https://askubuntu.com/a/1384946)
- [ubuntu tw 上的一篇各格式比較表(2009 年的文章了)](https://www.ubuntu-tw.org/modules/newbb/viewtopic.php?viewmode=compact&type=&topic_id=13024&forum=11)
- [XFS-vs-EXT4 的一篇簡中資源](http://xiaqunfeng.cc/2017/07/06/XFS-vs-EXT4/)

# 主機校時

## 安裝所需套件

- 安裝檢查
  {{<admonition info>}}
  核心的同步服務是 ntp，所以裝 ntp 其實就夠了
  {{</admonition>}}

  ```bash
  # dpkg -l 可以列出已安裝套件
  #dpkg -l | grep ntpdate
  dpkg -l | grep ntp
  ```

- 安裝語法
  ```bash
  #sudo apt-get install ntpdate
  sudo apt install ntp
  ```

  {{<admonition info>}}
  在 ubuntu 中，用 apt / apt-get 安裝套件時會預設啟用 (至少我在撰寫本文時，都不用特別另外執行 `systemctl enable` 命令，安裝的服務就會自行啟動了)
  {{</admonition>}}

## 啟用自動校正

- 編輯設定檔

  ```bash
  sudo vi /etc/ntp.conf
  ```

  把 server 區塊的設定調整一下，改成中華電信的校時服務

  [校時伺服器參考](https://www.stdtime.gov.tw/chinese/bulletin/NTP%20promo.txt)
  [這邊有另外一組可以嘗試](https://www.pool.ntp.org/zone/tw)
  [其他參考](https://vector.cool/ntp-server-taiwan/)

  ```conf
  # org linux time server
  # pool 0.ubuntu.pool.ntp.org iburst
  # pool 1.ubuntu.pool.ntp.org iburst
  # pool 2.ubuntu.pool.ntp.org iburst
  # pool 3.ubuntu.pool.ntp.org iburst

  # taiwan time server
  pool tock.stdtime.gov.tw iburst
  pool watch.stdtime.gov.tw iburst
  pool time.stdtime.gov.tw iburst
  pool clock.stdtime.gov.tw iburst
  pool tick.stdtime.gov.tw iburst

  # Use Ubuntu's ntp server as a fallback.
  # pool ntp.ubuntu.com
  ```

  改完需要重啟
  ```bash
  sudo vi /etc/ntp.conf
  ```

  {{<admonition info>}}
  這邊可以自己評估要不要調整，因為在改設定檔前使用 `ntpq -p` 命令去查他使用了那些校時伺服器時，有看到使用台灣的位置，也許這個調整行為是多的？
  {{</admonition>}}

- 確認服務狀態

  ntp 在安裝後預設就會被啟用，因此可以用以下網路工具確認是否已開始監聽 UDP Port 123

  ```bash
  netstat -tlunp

  # 如果沒有安裝網路工具，可以用以下命令安裝
  sudo apt install net-tools
  ```

- 其他參考命令

  ```bash
  # 查看目前使用的校時伺服器狀態
  sudo ntpq -p
  # 啟動
  sudo /etc/init.d/ntp start
  # 重開
  sudo /etc/init.d/ntp restart
  # 停止
  sudo /etc/init.d/ntp stop
  ```

## 參考

- [參考 1](https://nknuahuang.wordpress.com/tutorials/%E6%9C%89%E9%97%9Cubuntu%E8%88%87wordpress%E6%95%99%E5%AD%B8/ubuntu%E6%8E%92%E7%A8%8B%E6%A0%A1%E6%99%82%E8%A8%AD%E5%AE%9A/)
- [參考 2](https://blog.gtwang.org/linux/linux-ntp-installation-and-configuration-tutorial/)
- [參考 3](http://simple-is-beauty.blogspot.com/2017/04/ubuntu-ntp.html)

# containerd

[這邊筆記一個中國的 bili bili 連結](https://www.bilibili.com/read/cv16292700/)

## 核心套件安裝

* [安裝參考](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/install-containerd-on-ubuntu-22-04.html)

### 使用 apt-get 套件管理工具安裝

[安裝參考 in docker doc](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

{{<admonition info>}}
因為 containerd 的最新版也在 docker 的儲存庫中，所以直接使用 docker 的來源。
使用套件安裝會自動啟用並且一起安裝需要的套件 `runc` 。
{{</admonition>}}

- 更新 apt-get 的套件庫
  - 安裝需要的額外套件
    ```bash
    sudo apt-get update
    sudo apt-get install \
            ca-certificates \
            curl \
            gnupg \
            lsb-release
    ```

  - 設定安裝套件需要的憑證資料
    ```bash
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

- 安裝 containerd
  ```bash
  sudo apt-get update
  sudo apt-get install containerd.io
  ```

- 需要指定版本可使用以下方式安裝

  - 列出可用的版本

    ```bash
    sudo apt-cache madison containerd.io
    ```

  - 安裝指定版本

    ```bash
    sudo apt-get install containerd.io=<VERSION_STRING>
    ```

### 使用手動安裝

{{<admonition info>}}
這邊手動安裝僅作為整理記錄，我並未使用該方法安裝。
手動安裝的好處是不用多設定、安裝一些額外的東西，使用哪種安裝方式就因人而異了。
{{</admonition>}}

- 下載並設定 containerd

  * containerd 於 2022-09-16 時的最新版下載連結
    > https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz

  * 下載並解壓縮
    ```bash
    wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz

    sudo tar Czxvf /usr/local containerd-1.6.8-linux-amd64.tar.gz
    ```
  * 下載 systemd 使用的 service 執行檔
    ```bash
    wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

    sudo mv containerd.service /usr/lib/systemd/system/
    ```

- 下載並設定 runc

  * runc 於 2022-09-16 時的最新版下載連結
    > https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

  * 下載並安裝
    ```bash
    wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

    sudo install -m 755 runc.amd64 /usr/local/sbin/runc
    ```

## 調整設定檔

path: `/etc/containerd/config.toml`

因為使用  安裝 containerd 時的預設設定檔長這樣

```toml
disabled_plugins = ["cri"]
```

使用 containerd 內部的預設設定檔覆蓋後再去調整 `SystemdCgroup=true`

```bash
# 如果使用手動安裝的話，會沒有這個設定檔與資料夾，需要先手動建立資料夾
sudo mkdir -p /etc/containerd/

# 輸出 containerd 的預設設定檔出來
containerd config default | tee /etc/containerd/config.toml

# 編輯該設定檔
vi /etc/containerd/config.toml

# 預設設定的的 SystemdCgroup 設定為 false，改成 true 就可以了

# 或者也可以直接用 sed 處理
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

設定 systemdCgroup = true ([官方文件參考](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd))

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

## 建立 crictl 設定調整

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

## 啟動 containerd

```bash
sudo systemctl daemon-reload
# 由於使用 apt-get 套件安裝的話會預設啟動，所以這邊建議使用重啟的命令
sudo systemctl restart containerd

sudo systemctl enable --now containerd
```

# Network Setting

ref: [官方文件](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites)

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

# Disable Swap

[參考文件](https://jasonlee.xyz/guan-bi-linux-swap-kong-jian/)

- 暫時關閉

  ```bash
  sudo swapoff -a
  ```

  {{<admonition warning>}}
  此命令僅是暫時性關閉，server 重開後仍會啟用 swap，因此需要搭配下一節的設定來完全關閉 swap
  {{</admonition>}}

- 永久關閉 swap

  1. 註解掉以下命令開啟的檔案中，含有 swap 字樣的行次

     ```bash
     sudo vim /etc/fstab
     ```

  2. 用 vim 開啟 `/etc/sysctl.conf` 檔案，並加入 `vm.swappiness=0`

     > 有時候沒有設定此行也可以正常運行，但如果可以的話，關掉也許比較好

     新主機設定後此檔案應該會長這樣

     ```conf
     # ...其他原本就存在的註解資料
     vm.swappiness=0
     ```

# Kubernetes CLI & SELinux Set

ref: [官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

```bash
sudo apt-get update

# 這邊跟前面用套件安裝 containerd 的部分差了 apt-transport-https 這個套件，如果是用套件安裝 containerd 的話，可以考慮調整這條安裝命令
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

#指定版本
#sudo apt-get install -y kubelet=1.24.6-00 kubeadm=1.24.6-00 kubect

#降版與指定版本
#更改版本記得 apt-mark unhold kubelet kubeadm kubectl
#sudo apt-get install -y --allow-downgrades kubelet=1.24.6-00 kubeadm=1.24.6-00 kubect

# 避免 apt-get 隨便更新相關套件
sudo apt-mark hold kubelet kubeadm kubectl
```

安裝完畢後可看到安裝了已下套件

```bash
Setting up conntrack (1:1.4.5-2) ...
Setting up kubectl (1.25.1-00) ...
Setting up ebtables (2.0.11-3build1) ...
Setting up socat (1.7.3.3-2) ...
Setting up cri-tools (1.25.0-00) ...
Setting up kubernetes-cni (1.1.1-00) ...
Setting up kubelet (1.25.1-00) ...
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /lib/systemd/system/kubelet.service.
Setting up kubeadm (1.25.1-00) ...
Processing triggers for man-db (2.9.1-1) ...
```

# kubernetes cluster init / join command (use kubeadm)

- 預先拉取需要的 Image

  ```bash
  sudo kubeadm config images pull
  ```

- 建立 kubernetes cluster

  ```bash
  sudo kubeadm init

  # 指定 control-plane-endpoint
  sudo kubeadm init --control-plane-endpoint=<control-plane-endpoint-domain> \
                    --upload-certs
  #sudo kubeadm init --control-plane-endpoint=k8s-ubuntu-containerd.mshome.net \
  #                  --upload-certs

  # 指定版本
  # 由於指定版本會需要對應版本的 kubeadm / kubelet 等工具，記得安裝對應版本的工具
  sudo kubeadm init --control-plane-endpoint=<control-plane-endpoint-domain> \
                    --upload-certs
                    --kubernetes-version=v1.24.6
  ```

- reset kubernetes cluster

  ```bash
  kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets
  #kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets

  sudo kubeadm reset

  # 清除資料
  sudo rm -rf /etc/cni/net.d

  # 重設 ip table
  sudo iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
  ```

- join kubernetes cluster

  ```bash
  sudo kubeadm join <control-endpoint>:6443 --token <token> \
          --discovery-token-ca-cert-hash sha256:<hash>
  ```

# Other Setting

## Master node

### kubeconfig

#### bash profile

Master Node 在 kubeadm join / init 結束之後，可以考慮將 kubeconfig 檔案設定在 bash_profile 裡面

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf

vi .bash_profile
```

.bash_profile 的設定
```config
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/bin

export PATH

export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### 一般使用者 (非 root 帳號)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### single node

如果需要在 master 上部署服務的話，需要使用以下命令

```bash
#kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-
```

## Weave net CNI

如果是正在建立叢集，就會需要另外安裝 CNI ( Container Network Interface )，這邊採用 Weave net CNI

- 直接安裝

  [weave net doc - install](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation)
  ```bash
  #kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
  # 1.24.6
  #kubectl apply -f https://cloud.weave.works/k8s/net?k8s-version=Q2xpZW50IFZlcnNpb246IHZlcnNpb24uSW5mb3tNYWpvcjoiMSIsIE1pbm9yOiIyNSIsIEdpdFZlcnNpb246InYxLjI1LjIiLCBHaXRDb21taXQ6IjU4MzU1NDRjYTU2OGI3NTdhOGVjYWU1YzE1M2YzMTdlNTczNjcwMGUiLCBHaXRUcmVlU3RhdGU6ImNsZWFuIiwgQnVpbGREYXRlOiIyMDIyLTA5LTIxVDE0OjMzOjQ5WiIsIEdvVmVyc2lvbjoiZ28xLjE5LjEiLCBDb21waWxlcjoiZ2MiLCBQbGF0Zm9ybToibGludXgvYW1kNjQifQpLdXN0b21pemUgVmVyc2lvbjogdjQuNS43ClNlcnZlciBWZXJzaW9uOiB2ZXJzaW9uLkluZm97TWFqb3I6IjEiLCBNaW5vcjoiMjQiLCBHaXRWZXJzaW9uOiJ2MS4yNC42IiwgR2l0Q29tbWl0OiJiMzliZjE0OGNkNjU0NTk5YTUyZTg2NzQ4NWMwMmM0ZjlkMjhiMzEyIiwgR2l0VHJlZVN0YXRlOiJjbGVhbiIsIEJ1aWxkRGF0ZToiMjAyMi0wOS0yMVQxMzoxMjowNFoiLCBHb1ZlcnNpb246ImdvMS4xOC42IiwgQ29tcGlsZXI6ImdjIiwgUGxhdGZvcm06ImxpbnV4L2FtZDY0In0K
  ```

- 如果要特別調整網段

  {{<admonition info>}}
由於發現官方的安裝語法於 2022-10-11 時發現有變動，以下命令待測試
  {{</admonition>}}

  ```bash
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=172.30.0.0/16"
  ```

- 清除安裝

  {{<admonition info>}}
由於發現官方的安裝語法於 2022-10-11 時發現有變動，以下命令待測試
  {{</admonition>}}

  ```bash
  kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  ```

# 網段資料參考

全預設 weave-net cni 的 kubernetes cluster 網段

| Used For                          | Subnet       | Genmask     | Start     | End            |
|-----------------------------------|--------------|-------------|-----------|----------------|
| Kubernetes Pod Subnet (Weave-cni) | 10.32.0.0/12 | 255.240.0.0 | 10.32.0.1 | 10.47.255.254  |
| Kubernetes Service Subnet         | 10.96.0.0/12 | 255.240.0.0 | 10.96.0.1 | 10.111.255.254 |

