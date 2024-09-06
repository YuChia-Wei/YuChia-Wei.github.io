# Kubernetes 環境準備 - centos7 & containerd


定期更新版 kubernetes 安裝筆記

{{<admonition info>}}
|             |                            |
|-------------|----------------------------|
| K8s:        | v1.25.0                    |
| OS:         | CentOs 7 (kernel = 3.10.1) |
| cri:        | containerd 1.6.8-3.1.el7   |
| Update Time | 2022/09/14                 |
{{</admonition>}}

<!--more-->



# 主機校時

## 安裝所需套件

```bash
sudo yum install ntp
```

## 啟用自動校正

- 編輯設定檔

  ```bash
  sudo vi /etc/ntp.conf
  ```

  把 server 區塊的設定調整一下，改成中華電信的校時服務 (會比使用 linux 的校時伺服器快一點，因為比較近)

  [校時伺服器參考](https://www.stdtime.gov.tw/chinese/bulletin/NTP%20promo.txt)
  [這邊有另外一組可以嘗試](https://www.pool.ntp.org/zone/tw)
  [其他參考](https://vector.cool/ntp-server-taiwan/)

  ```conf
  # org linux time server
  # server 0.centos.pool.ntp.org iburst
  # server 1.centos.pool.ntp.org iburst
  # server 2.centos.pool.ntp.org iburst
  # server 3.centos.pool.ntp.org iburst

  # taiwan time server
  server tock.stdtime.gov.tw
  server watch.stdtime.gov.tw
  server time.stdtime.gov.tw
  server clock.stdtime.gov.tw
  server tick.stdtime.gov.tw
  ```

- 將校時功能加入系統服務

  ```bash
  sudo systemctl start ntpd
  sudo systemctl enable ntpd
  ```

- 確認啟用狀態

  ```bash
  systemctl status ntpd
  ```

## 參考

- [參考 1](https://www.ltsplus.com/linux/rhel-centos-7-setup-ntp-sync-time)
- [參考 2](https://blog.gtwang.org/linux/linux-ntp-installation-and-configuration-tutorial/)

# containerd

## 核心套件安裝

- 更新 yum 並定義 docker 工具來源

    {{<admonition info>}}
因為 containerd 的最新版也在 docker 的儲存庫中，所以直接使用 docker 的來源
    {{</admonition>}}

    ```bash
    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

- 安裝 containerd

    ```bash
    sudo yum install containerd.io
    ```

- 需要指定版本可使用以下方式安裝

  - 列出可用的版本

    ```bash
    sudo yum list containerd.io --showduplicates | sort -r
    ```

  - 安裝指定版本

    ```bash
    sudo yum install containerd.io-<VERSION_STRING>
    ```

## 調整設定檔

path: `/etc/containerd/config.toml`

使用 yum 安裝 containerd 時的預設設定檔長這樣

```toml
disabled_plugins = ["cri"]
```

使用 containerd 內部的預設設定檔覆蓋後再去調整 `SystemdCgroup=true`

```bash
# 輸出 containerd 的預設設定檔出來
sudo containerd config default | tee /etc/containerd/config.toml

# 編輯該設定檔
sudo vi /etc/containerd/config.toml

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
sudo vi /etc/crictl.yaml
```

yaml 內容如下
```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```

## 其他輔助性質的工具安裝

### containerd 的 CNI

{{<admonition info>}}
這個不裝不會影響 kubernetes 運作，但是會造成 containerd 內部出現錯誤時，無法在沒有 kubernetes 的狀態下進行容器操作。
{{</admonition>}}

> cni 在文章撰寫當下 (2022-09-19 最新版本為 1.1.1) [cni github](https://github.com/containernetworking/plugins/releases)

```bash
sudo mkdir -p /opt/cni/bin/
sudo wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### nerdctl

nerdctl [Github](https://github.com/containerd/nerdctl) 是輔助性質的命令列工具，建議安裝

撰文當下最新版本連結如下
> https://github.com/containerd/nerdctl/releases/download/v0.23.0/nerdctl-0.23.0-linux-amd64.tar.gz

```bash
wget https://github.com/containerd/nerdctl/releases/download/v0.23.0/nerdctl-0.23.0-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local/bin nerdctl-0.23.0-linux-amd64.tar.gz
```

{{<admonition info>}}
nerdctl 的發布清單裡面有一個是 nerdctl-full-{version}-linux-amd64.tar.gz，這個包含了 containerd / runc / CNI 等必要倚賴，如果使用手動安裝的話，也許可以考慮用這包一次全裝。
{{</admonition>}}

## 啟動 containerd

{{<admonition warning>}}
啟動 containerd 的段落會放在這邊是因為，前面的所有安裝變更都會需要重啟 containerd !!
{{</admonition>}}

```bash
sudo systemctl daemon-reload

# 套件安裝會預設啟用
sudo systemctl enable --now containerd

# 如果已經有設定啟用 containerd 的話就要用這個
sudo systemctl restart containerd
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
     # sysctl settings are defined through files in
     # /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
     #
     # Vendors settings live in /usr/lib/sysctl.d/.
     # To override a whole file, create a new file with the same in
     # /etc/sysctl.d/ and put new settings there. To override
     # only specific settings, add a file with a lexically later
     # name in /etc/sysctl.d/ and put new settings there.
     #
     # For more information, see sysctl.conf(5) and sysctl.d(5).
     vm.swappiness=0
     ```

# Kubernetes CLI & SELinux Set

ref: [官方文件](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

{{<admonition info>}}
先前舊的命令會有 repo_gpgcheck 的問題，在撰文當下 (2022/09/12) 已被修正

若仍有相關問題的話，請參考官方說明

```
If the baseurl fails because your Red Hat-based distribution cannot interpret basearch,
replace \$basearch with your computer's architecture.

Type uname -m to see that value. For example,
the baseurl URL for x86_64 could be: https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64.
```

{{</admonition>}}

安裝完畢後可看到安裝了已下套件

```bash
Installed:
  kubeadm.x86_64 0:1.25.0-0
  kubectl.x86_64 0:1.25.0-0
  kubelet.x86_64 0:1.25.0-0

Dependency Installed:
  conntrack-tools.x86_64 0:1.4.4-7.el7
  cri-tools.x86_64 0:1.24.2-0
  kubernetes-cni.x86_64 0:0.8.7-0
  libnetfilter_cthelper.x86_64 0:1.0.0-11.el7
  libnetfilter_cttimeout.x86_64 0:1.0.0-7.el7
  libnetfilter_queue.x86_64 0:1.0.2-2.el7_2
  socat.x86_64 0:1.7.3.2-2.el7

Complete!
```

# kubernetes cluster init / join command (use kubeadm)

- 預先拉取需要的 Image

    ```bash
    kubeadm config images pull
    ```

- 建立 kubernetes cluster

    ```bash
    kubeadm init

    # 指定 control-plane-endpoint
    kubeadm init --control-plane-endpoint=<control-plane-endpoint-domain> \
                 --upload-certs
    ```

- reset kubernetes cluster

    ```bash
    kubeadm reset

    # 清除資料
    rm -rf /etc/cni/net.d

    # 重設 ip table
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    ```

- join kubernetes cluster

    ```bash
    kubeadm join <control-endpoint>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
    ```

# Other Setting

## Master node

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

## Weave net CNI

如果是正在建立叢集，就會需要另外安裝 CNI ( Container Network Interface )，這邊採用 Weave net CNI

- 直接安裝

    [weave net doc - install](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#-installation)
    ```bash
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    ```

- 如果要特別調整網段

    ```bash
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=172.30.0.0/16"
    ```

- 清除安裝

    ```bash
    kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    ```

# 網段資料參考

全預設 weave-net cni 的 kubernetes cluster 網段

| Used For                          | Subnet        | Genmask     | Start      | End            |
| --------------------------------- | ------------- | ----------- | ---------- | -------------- |
| Kubernetes Pod Subnet (Weave-cni) | 10.32.0.0/12  | 255.240.0.0 | 10.32.0.1  | 10.47.255.254  |
| Kubernetes Service Subnet         | 10.105.0.0/12  | 255.240.0.0 | 10.105.0.1  | 10.111.255.254 |

