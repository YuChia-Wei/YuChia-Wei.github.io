# Openssl Tls Version Downgrade in Dockerfile


此邊筆記紀錄如何在 dockerfile 中設定 debian 11/12 與 alpine 中的 openSSL config 檔案

<!--more-->

## debain 11 (openSSL 1.0.x)

```dockerfile
RUN sed -i 's/DEFAULT@SECLEVEL=2/DEFAULT@SECLEVEL=1/g' /etc/ssl/openssl.cnf
RUN sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1/g' /etc/ssl/openssl.cnf
RUN sed -i 's/DEFAULT@SECLEVEL=2/DEFAULT@SECLEVEL=1/g' /usr/lib/ssl/openssl.cnf
RUN sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1/g' /usr/lib/ssl/openssl.cnf

#RUN sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1/g; s/DEFAULT@SECLEVEL=2/DEFAULT@SECLEVEL=1/g' /usr/lib/ssl/openssl.cnf
```

## debain 12 (openSSL 3.0.x)

```dockerfile
RUN sed -i 's/openssl_conf = openssl_init/#openssl_conf = openssl_init/' /etc/ssl/openssl.cnf
RUN sed -i '1i openssl_conf = default_conf' /etc/ssl/openssl.cnf &&
    echo "\n[ default_conf ]\nssl_conf = ssl_sect\n[ssl_sect]\nsystem_default = system_default_sect\n[system_default_sect]\nMinProtocol = TLSv1\nCipherString = DEFAULT:@SECLEVEL=1" >> /etc/ssl/openssl.cnf
```

## alpine (OpenSSL 3.0.x)

```dockerfile
RUN sed -i 's/openssl_conf = openssl_init/#openssl_conf = openssl_init/' /etc/ssl/openssl.cnf
RUN sed -i '1i openssl_conf = default_conf' /etc/ssl/openssl.cnf &&
    echo -e "\n[ default_conf ]\nssl_conf = ssl_sect\n[ssl_sect]\nsystem_default = system_default_sect\n[system_default_sect]\nMinProtocol = TLSv1\nCipherString = DEFAULT:@SECLEVEL=1" >> /etc/ssl/openssl.cnf
```

