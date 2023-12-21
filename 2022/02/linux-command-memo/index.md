# Linux Command Memo


ssh 遠端到 linux server 時常用的指令筆記

<!--more-->

## 命令紀錄

* 清空 CLI
    ```
    ctrl + L
    ```
* 查看路由表
    ```bash
    netstat -rn
    ```
* 查看網路使用狀態 (Port 使用狀態)
    ```bash
    netstat -lptu
    netstat -tulpn
    netstat -tln
    ```
* 刪除路由表
    ```bash
    route del -net 10.32.0.0/12
    ```
* 清除 IpTables 資料
    資料來源：http://s2.naes.tn.edu.tw/~kv/iptables.htm
    ```bash
    ###-----------------------------------------------------###
    # 清除先前的設定
    ###-----------------------------------------------------###
    # 清除預設表 filter 中，所有規則鏈中的規則
    iptables -F
    # 清除預設表 filter 中，使用者自訂鏈中的規則
    iptables -X

    # 清除mangle表中，所有規則鏈中的規則
    iptables -F -t mangle
    # 清除mangle表中，使用者自訂鏈中的規則
    iptables -t mangle -X

    # 清除nat表中，所有規則鏈中的規則
    iptables -F -t nat
    # 清除nat表中，使用者自訂鏈中的規則
    iptables -t nat -X
    ```

    資料來源：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
    ```bash
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    ```
* 重啟應用程式:
    ```bash
    systemctl restart <application Name>
    ```
* 確認應用程式執行狀態
    ```bash
    systemctl status <application Name>
    ```
* 查看應用程式紀錄 (log)
    ```bash
    journalctl -xeu <application Name>
    ```
* 設定環境參數
    {{<admonition info >}}可以設定到 `~/.bash_profile` 檔案內，來讓每次登入都自動套用相關環境參數
    {{</admonition >}}

    ```bash
    # 建立環境變數
    export <key>=<value>

    # 使用環境變數 = ${<key>}
    ```

    使用範例

    ```bash
    export test=123
    echo ${test}
    # bash 中會顯示 123
    ```

## 倚賴套件安裝

* 網路工具 (netstat 需要的套件)

```bash
# yum install net-tools     [On CentOS/RHEL]
# apt install net-tools     [On Debian/Ubuntu]
# zypper install net-tools  [On OpenSuse]
# pacman -S net-tools      [On Arch Linux]
```

