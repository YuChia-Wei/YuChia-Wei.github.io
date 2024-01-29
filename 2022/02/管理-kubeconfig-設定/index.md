# 管理 KubeConfig 設定


kube-config 設定參考

<!--more-->

{{<admonition warning >}}
這個做法會有安全性問題，理想上還是另外使用管理工具 (如 rancher) 來處理使用者資訊以及一般使用者使用的 kubeconfig
{{</admonition >}}

## 取得 kubeadm 所建立的叢集的管理設定檔

```bash
cat /etc/kubernetes/admin.conf
```

## 取得 suse rke2 所產生的管理設定檔

```bash
cat /etc/rancher/rke2/rke2.yaml
```

## 設定檔案結構

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <first-cluster-cert-data>
    server: https://<first-cluster domain>:6443
  name: first-cluster
- cluster:
    certificate-authority-data: <second-cluster-cert-data>
    server: https://<second-cluster domain>:6443
  name: second-cluster
contexts:
- context:
    cluster: first-cluster
    user: first-cluster-admin
  name: first-cluster-admin@first-cluster
- context:
    cluster: second-cluster
    user: second-cluster-admin
  name: second-cluster-admin@second-cluster
current-context: first-cluster-admin@first-cluster
kind: Config
preferences: {}
users:
- name: first-cluster-admin
  user:
    client-certificate-data: <first-cluster-admin-cert-data>
    client-key-data: <first-cluster-admin-cert-key>
- name: second-cluster-admin
  user:
    client-certificate-data: <second-cluster-admin-cert-data>
    client-key-data: <second-cluster-admin-cert-key>
```

