# 在 linux 安裝 ssl 中間人憑證


如果網路環境會被替換憑證，且暫時無法取得憑證時，可以利用本篇筆記所記錄的方式處理憑證

- [參考資料](https://stackoverflow.com/a/76244812)

<!--more-->

1. Get SSL Certificate
    ```bash
    openssl s_client -connect {domain}:443 -showcerts
    ```
1. 找出輸出文字中的所有憑證區塊，並存到 /usr/local/share/ca-certificates/*.crt
1. 執行憑證更新
    ```bash
    sudo update-ca-certificates
    ```
1. 重啟需要使用憑證的服務 (例如 docker)

