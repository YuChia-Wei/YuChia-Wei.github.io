# Kube Config Setting


kube-config 設定參考

<!--more-->

{{<admonition warning >}}
這個做法比較 Hardcore，直接在 k8s 的管理主機上調整 kubectl 使用的 kube.config 內容，由於直接複製外部叢集的 admin.config，基於安全性問題，不建議用於正式環境上。
{{</admonition >}}

用 vim 開啟 kubernetes 用的設定檔

```bash
vim /etc/kubernetes/admin.conf
```

調整內容 (加入外部 k8s 叢集的相關資料)
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <first-cluster-cert-data>
    server: https://<first-cluster domain>:<first-cluster port>
  name: first-cluster
- cluster:
    certificate-authority-data: <second-cluster-cert-data>
    server: https://<second-cluster domain>:<second-cluster port>
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

