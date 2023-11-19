# 2022-06-14 的 argocd 與 istio 版本更新筆記


稍微整理記錄一下這時間點有使用的服務版本與更新內容

* ArgoCD 2.4
* Istio 1.14

<!--more-->

## ArgoCD

ArgoCD 2.4 版已於 2022/06/10 發布
* [Github release page](https://github.com/argoproj/argo-cd/releases/tag/v2.4.0)
* [官方說明文件 - 2.3 升級 2.4 注意事項](https://argo-cd.readthedocs.io/en/latest/operator-manual/upgrading/2.3-2.4/)
* [官方 BLOG - 2.3 升級 2.4 的 Breaking Change](https://blog.argoproj.io/breaking-changes-in-argo-cd-2-4-29e3c2ac30c9)

需要注意的更新項目
1. Web UI 支援 Terminal，可以線上執行容器內命令
2. 因為加入了上述功能，權限控制的設定有些變化，若允許所有資源的 Create 行為時，預設允許使用 Web Terminal (建議重新檢視權限設定)
3. argocd-repo-server 的安全性增強，不再使用 default 這個 service Account 來做認證，可能導致一些套件受影響；由於我們的使用近乎全預設，暫時不確定直營這邊使用的 ArgoCD 如果要升級的話會不會受影響
4. Plugin 使用方式改變；同上，由於直營使用的方式沒有額外使用其他特別的操作，暫時不確定是否會受影響

我先前有製作 ArgoCD 2.3.4 with Istio 的 <https://github.com/YuChia-Wei/argoproj-deploy|kustomization 安裝文件>，近期會找時間測試更新成 2.4 版的

## istio

Istio 1.14 已於 2022/06/01 發布，之後也在 2022/06/09 時，與 1.12 / 1.13 版同步發布 patch，主要是做漏洞修補。

各版本細節可參照 [官方新聞頁面](https://istio.io/latest/news/)

以下為我稍作整理的 1.14 版本更新的簡介，若要查閱完整更新內容請參閱 [istio 1.14 變更紀錄](https://istio.io/latest/news/releases/1.14.x/announcing-1.14/change-notes/)
1. istio 1.14 支援的 kubernetes 版本為 1.21 ~ 1.24
2. 網路管理功能微調與新增功能；尚未研究相關功能我們用不用的到
3. 常態的錯誤修正與功能新增
4. Support for the SPIRE runtime (SPIFFE)
> [SPIRE(SPIFFE) 官網](https://spiffe.io/)
> [SPIRE (SPIFFE) Github](https://github.com/spiffe/spiffe)
> 好像很有用的東西，需要研究一下能做什麼
