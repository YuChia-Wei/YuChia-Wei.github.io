# Kubernetes 安裝筆記 part.1 : 環境準備


Kubernetes Master(Control-plane) / Worker 的 Node 環境準備

{{<admonition info >}}
|      |                                                              |
|------|--------------------------------------------------------------|
| OS:  | CentOs 7 (kernel = 3.10.1)                                   |
| K8s: | v1.24.1 (新安裝時預設使用當下的最新版，因此此版本資訊參考用) |
{{</admonition >}}

> 更新時間：
> * 2022/06/11 - use containerd / 不使用 docker
> * 2022/04/10 - kubelet kubeadm kubectl 等工具安裝錯誤的解決方案

<!--more-->

## Docker

{{<admonition warning >}}
在 kubernetes 1.24 版之後，由於被拔除了 docker 支援，所以這邊請評估是否還有使用 docker 的需要來選擇安裝，並請參考 [containerd](#containerd-安裝) 的設定章節
{{</admonition >}}

> [官方文件](https://docs.docker.com/engine/install/centos/)

* 清除舊安裝

    ```bash
    sudo yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine

    sudo yum remove docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

* 更新 yum 以安裝 docker

    ```bash
    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

* 安裝 docker-ce 與依賴套件

    ```bash
    sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

  * 指定版本安裝

    * 列出可用的版本

      ```bash
      sudo yum list docker-ce --showduplicates | sort -r
      ```

    * 安裝指定版本

      ```bash
      sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
      ```

* 啟動 docker 並設定預設啟動

    {{<admonition warning >}}
在 kubernetes 1.24 版之後，由於被拔除了 docker 支援，所以這邊可以不用啟用了，並請參考 [containerd](#containerd-安裝) 的設定章節
    {{</admonition >}}

  ```bash
  sudo systemctl start docker
  sudo systemctl enable docker
  ```

* Docker 磁碟管理模式確認

  * 確認目前 docker 用的磁碟管理模式

    [[2018 iThome 鐵人賽] Day 28](https://ithelp.ithome.com.tw/articles/10197423)

    ```bash
    docker info
    ```

    {{<admonition info >}}
正常來說預設安裝後就會使用 overlay2 的磁碟管理模式，若非此模式，可配合下一節的 daemon.json 設定檔來變更設定
    {{</admonition >}}

* Set daemon

  * 新安裝尚未有 daemon.json 檔案時可使用以下語法進行建立

    ```bash
    sudo mkdir /etc/docker
    cat <<EOF | sudo tee /etc/docker/daemon.json
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
          "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF
    ```

    {{<admonition info >}}
[此段語法來源](https://v1-22.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
如果有更換使用網段的需求可參考 [此篇](../../03/kubernetes-docker-的預設網段筆記)
    {{</admonition >}}

  * 編輯現有 daemon.json

    ```bash
    vim /etc/docker/daemon.json
    ```

    * 加入 cgroupdriver 設定

      可參考官方文件：

      * [configuring a cgroup driver](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configuring-a-cgroup-driver)
      * [kubernetes - docker](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)

      {{<admonition info >}}
由於新版的 kubernetes 文件對於 docker runtime 的[說明](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)中已拔除 daemon.json 設定檔範例。

於 2022 年 02 月 21 日 的測試，仍需要在 daemon.json 檔案中加入 cgroupdriver=systemd 的設定才可正常的初始化 kubernetes 叢集。

若要參考範例可選擇較舊的版本，如 1.22 版文件 ([連結](https://v1-22.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#docker))。
      {{</admonition >}}

      ```json
      "exec-opts": ["native.cgroupdriver=systemd"],
      ```

    * 加入磁碟管理模式的設定

      ```json
      "storage-driver": "overlay2"
      ```

      {{<admonition info >}}
由於現在 Linux OS 新安裝的 docker 都會預設都會使用 overlay2 的磁碟驅動，通常不用特別設定

有時可能也會發生調整為 overlay2 之後就無法啟動 docker 的狀況，所以若有 docker 無法啟動的情況時，可以省略 overlay2 的設定
      {{</admonition >}}

* restart docker

  ```bash
  sudo systemctl enable docker
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  ```

## containerd 安裝

{{<admonition info >}}
* Kubernetes 1.24 以後的版本建議使用此套件 (或其他預設支援的 ORI 套件)
* 如果有走前一章節的 docker 安裝的話，可以直接跳到 `設定 config`
{{</admonition >}}

* 更新 yum 以安裝 docker

    ```bash
    sudo yum install -y yum-utils
    sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

* 安裝 containerd.io

    ```bash
    sudo yum install containerd.io
    ```

* 設定 config

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

* 啟動 (重啟) containerd

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# 如果已經有設定啟用 containerd 的話就要用這個
sudo systemctl restart containerd
```
## Iptables 設定

{{<admonition info >}}
kubernetes 1.24 版本後變更
{{</admonition >}}

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

## 關閉 Swap

[參考文件](https://jasonlee.xyz/guan-bi-linux-swap-kong-jian/)

* 使用指令關閉

```bash
sudo swapoff -a
```

{{<admonition warning >}}
此命令僅是暫時性關閉，server 重開後仍會啟用 swap，因此需要搭配下一節的設定來完全關閉 swap
{{</admonition >}}

* 永久關閉 swap

    1. 註解掉以下命令開啟的檔案中，含有 swap 字樣的行次

        ```bash
        vim /etc/fstab
        ```

    2. 用 vim 開啟 /etc/sysctl.conf 檔案，並加入 `vm.swappiness=0`

        > 此項我在目前有運作的主機上並未進行此設定，但仍可正常運作，但是前述 /etc/fstab 的檔案是確定要修改的

## 安裝 kubernetes 所需 cli 工具

直接參考[官方資料](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

{{<admonition info >}}
* 以下命令會安裝 kubelet / kubeadm / kubectl 等 cli 工具，並且會調整 selinux 設定
* 此處紀錄的命令已修正 repo_gpgcheck 問題 (請見下面 Troubleshooting 章節)
{{</admonition>}}

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

{{<admonition info >}}
預設會安裝以下依賴工具
* conntrack-tools
* cri-tools
* kubernetes-cni
* libnetfilter_cthelper
* libnetfilter_cttimeout
* libnetfilter_queue
* socat
{{</admonition>}}

### Troubleshooting

如果在安裝 kubelet 等套件時出現以下訊息

```
https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64/repodata/repomd.xml: [Errno -1] repomd.xml signature could not be verified for kubernetes
Trying other mirror.
```

積極的解決方案 = unknow

消極的解決方案

修改 `/etc/yum.repos.d/kubernetes.repo` 裡面的 `repo_gpgcheck` 為 0

```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```

* ref1: https://github.com/kubernetes/kubernetes/issues/60134
* ref2: https://stackoverflow.com/questions/66853387/signature-could-not-be-verified-for-kubernetes-repo

