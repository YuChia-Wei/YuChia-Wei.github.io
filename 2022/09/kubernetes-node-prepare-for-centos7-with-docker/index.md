# Kubernetes 環境準備 - centos7 & docker


定期更新版 kubernetes 安裝筆記

{{<admonition info>}}
|             |                            |
|-------------|----------------------------|
| K8s:        | v1.25.0                    |
| OS:         | CentOs 7 (kernel = 3.10.1) |
| cri:        | docker 20.10.18            |
| Update Time | 2022/09/12                 |
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

# Docker

## 安裝之前

{{<admonition warning>}}
在 kubernetes 1.24 版之後，由於被拔除了 docker 支援，所以請評估是否還有使用 docker 的需要，且就算是 docker，背後也是使用 containerd 來運行，所以如果不是要做為映像檔建置主機的話，其實可以考慮直接使用 containerd。
{{</admonition>}}

> [官方文件](https://docs.docker.com/engine/install/centos/)

## 核心套件安裝

- 更新 yum 並定義 docker 工具來源

    ```bash
    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

- 安裝 docker-ce 與依賴套件

    ```bash
    sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

    {{<admonition info>}}
    由於是要做為 kuberntes 的容器執行核心，其實可以考慮不安裝 docker-compose-plugin
    {{</admonition>}}

- 需要指定版本可使用以下方式安裝

  - 列出可用的版本

    ```bash
    sudo yum list docker-ce --showduplicates | sort -r
    ```

  - 安裝指定版本

    ```bash
    sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
    ```

## 調整設定檔

建立與編輯 `/etc/docker/daemon.json`

[官方對 Cgroup drivers 的說明](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers)

```bash
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m"
    }
}
EOF
```

## 啟動 docker

```bash
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker
```

# Cri-dockerd

## Reference

- [migrate-kubernetes-to-dockershim](https://devopstales.github.io/home/migrate-kubernetes-to-dockershim/)
- [how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim](https://www.mirantis.com/blog/how-to-install-cri-dockerd-and-migrate-nodes-from-dockershim)

## Install and configure

{{<admonition info>}}
前述的參考中，使用的下載命令都是 wget ，不過我的環境沒有 wget 可以用，也不想多安裝東西，就改用最小安裝的 centos 就有的 curl 了
{{</admonition>}}

```bash
# 先到 OS 預設的暫存資料夾
cd /tmp

# 確定目前 cri-dockerd 的版本並設定到環境參數中
echo $(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4)

# 2022-09-11 時的最新版本 = 0.2.5

# 2022-09-11 時的最新版本網址
# https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd-0.2.5.amd64.tgz
curl -L -o cri-dockerd-0.2.5.amd64.tgz https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd-0.2.5.amd64.tgz
tar xvf cri-dockerd-0.2.5.amd64.tgz

# Centos 可以考慮 rpm 安裝
# 安裝命令請自行測試
# centos 7
# curl -L -o cri-dockerd-0.2.5-3.el7.x86_64.rpm https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd-0.2.5-3.el7.x86_64.rpm
# centos 8
# curl -L -o cri-dockerd-0.2.5-3.el8.x86_64.rpm https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd-0.2.5-3.el8.x86_64.rpm

# 移動執行檔案
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/cri-dockerd

# 清掉解壓縮時多出來的資料夾
sudo rm -rf cri-dockerd

# 確認有沒有安裝成功
cri-dockerd --version
cri-dockerd 0.2.0 (HEAD)

curl -L -o cri-docker.service https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
curl -L -o cri-docker.socket https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```

## enable Cri-docker.service / socket

```bash
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
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

{{<admonition info>}}
我在公司環境的主機上並未設定 cri-socket 的參數，不確定是不是因為公司環境的 docker 版本是卡在 18.6.1 的關係
{{</admonition>}}

因為安裝 docker 的時候同時會裝 containerd，所以 kubeadm 會因為主機內有兩個容器執行服務而無法啟動，這時可以使用 `--cri-socket` 參數來指定要使用哪一個容器服務，也可以直接設定 kubeadm 的環境檔案來調整。

跟 kubeadm 相關的命令都需要加入此參數才能作用

- 預先拉取需要的 Image

    ```bash
    kubeadm config images pull --cri-socket unix:///var/run/cri-dockerd.sock
    ```

- 建立 kubernetes cluster

    ```bash
    kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock

    # 指定 control-plane-endpoint
    kubeadm init --control-plane-endpoint=<control-plane-endpoint-domain> \
                 --upload-certs \
                 --cri-socket unix:///var/run/cri-dockerd.sock
    ```

- reset kubernetes cluster

    ```bash
    kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock

    # 清除資料
    rm -rf /etc/cni/net.d

    # 重設 ip table
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    ```

- join kubernetes cluster

    ```bash
    kubeadm join <control-endpoint>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash> \
        --cri-socket unix:///var/run/cri-dockerd.sock
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

全預設 docker + weave-net cni 的 kubernetes cluster 網段

| Used For                          | Subnet        | Genmask     | Start      | End            |
| --------------------------------- | ------------- | ----------- | ---------- | -------------- |
| Kubernetes Pod Subnet (Weave-cni) | 10.32.0.0/12  | 255.240.0.0 | 10.32.0.1  | 10.47.255.254  |
| Kubernetes Service Subnet         | 10.105.0.0/12  | 255.240.0.0 | 10.105.0.1  | 10.111.255.254 |
| Docker (default bridge)           | 172.17.0.0/16 | 255.255.0.0 | 172.17.0.1 | 172.17.255.254 |
| Docker (custom bridge)            | 172.18.0.0/16 | 255.255.0.0 | 172.18.0.1 | 172.18.255.254 |


{{<admonition info>}}
在 docker 中另外建立的 bridge 網段預設是第二段依序 +1，極限數值範圍為 172.16.0.1 - 172.31.255.254

即

subnet pool = 172.17.0.0/12
subnet size = 16

[數值來源參考](https://github.com/docker/docker.github.io/issues/8663)
{{</admonition>}}

