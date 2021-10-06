# local-photon

## Prerequisites
### Hardware
2CPU / 2GB Mem / 40GB Disk
### Software
Docker Engine: v17.06.0-ce or later
Docker Compose: v1.18.0 or later


## My Environment
Photon OS 3.0 (64-bit)
2 vCPU / 2 GB Memory / 200 GB Disk 


### Usage
CPU: 263 MHz
MEMORY: 368 MB
STORAGE: 104.08 GB



## OS preparation

1. 使用 **root** 登入，若安裝時沒有設定密碼則初始密碼為 `changeme`

2. 為了讓 root user 可以 SSH，將 **PermitRootLogin** 設定為 **yes**
```
$ sed -I 's/^PermitRootLogin no/PermitRootLogin yes/' /etc/ssh/sshd_config
$ systemctl restart sshd
```

3. 加入我的 DNS suffix，將 `sapphire.local` 改成你自己的
```
$ echo "Domain=sapphire.local" >> /etc/systemd/network/99-static-en.network
$ systemctl restart systemd-networkd
```

4. Photon OS 預設有裝 docker，但需要 **enabled**
```
$ systemctl enable docker
$ systemctl start docker
$ sudo docker version
```

5. 安裝 [docker-compose](https://docs.docker.com/compose/install/#install-compose)
```
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/harbor
$ sudo docker-compose --version

```

6. 安裝憑證(CA)

    6.1 生成 CA 私鑰
    ```
    $ openssl genrsa -out ca.key 4096
    ```

    6.2  生成 CA
    ```
    openssl req -x509 -new -nodes -sha512 -days 3650 \
     -subj "/C=TW/ST=Taiwan/L=Taipei/O=example/OU=Personal/CN=harbor.sapphire.local" \
     -key ca.key \
     -out ca.crt

    ```

    6.3  生成 Server 私鑰
    ```
    openssl genrsa -out harbor.sapphire.local.key 4096
    ```

    6.4 生成 Server CSR
    ```
    openssl req -sha512 -new \
        -subj "/C=TW/ST=Taiwan/L=Taipei/O=example/OU=Personal/CN=harbor.sapphire.local" \
        -key harbor.sapphire.local.key \
        -out harbor.sapphire.local.csr
    ```

    6.5 生成 x509 格式標準檔案
    ```
    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = harbor.sapphire.local
    DNS.2 = harbor
    IP.1 = 10.213.180.70
    EOF
    ```

    6.6 利用此 v3.ext 檔案生成 harbor 憑證
    ```
    openssl x509 -req -sha512 -days 3650 \
        -extfile v3.ext \
        -CA ca.crt -CAkey ca.key -CAcreateserial \
        -in harbor.sapphire.local.csr \
        -out harbor.sapphire.local.crt
    ```

    6.7 創建資料夾憑證
    ```
    mkdir -p /data/cert
    mkdir -p /etc/docker/certs.d/harbor.sapphire.local
    cp ~/harbor.sapphire.local.crt /data/cert/
    cp ~/harbor.sapphire.local.key /data/cert/
    ```
    6.8 將憑證轉成 Docker daemon 吃的格式並丟進去 Photon OS 憑證路徑
    ```
    openssl x509 -inform PEM -in ~/harbor.sapphire.local.crt -out /etc/docker/certs.d/harbor.sapphire.local/harbor.sapphire.local.cert

    ```
    6.9 丟入 Photon OS 憑證路徑，並重啟 docker
    ```
    cp ~/harbor.sapphire.local.key /etc/docker/certs.d/harbor.sapphire.local/
    cp ~/ca.crt /etc/docker/certs.d/harbor.sapphire.local/
    systemctl restart docker
    ```
7. 環境準備完成。

## Install harbor

1. 到 [Harbor Release](https://github.com/goharbor/harbor/releases) 選版本下載，本文使用線上版本 **v2.3**
```
$ curl -L https://github.com/goharbor/harbor/releases/download/v2.3.1/harbor-online-installer-v2.3.1.tgz -o ~/harbor.tgz

$ tar -zxvf ~/harbor.tgz
harbor/prepare
harbor/LICENSE
harbor/install.sh
harbor/common.sh
harbor/harbor.yml.tmpl
```

2. 複製一份 tamplate，修改 harbor 配置：hostname/certificate/private_key/password
```
$ cp harbor/harbor.yml.tmpl harbor/harbor.yml

$ sed -i 's/hostname: reg.mydomain.com/hostname: harbor.sapphire.local/' ~/harbor/harbor.yml
$ sed -i 's/your\/certificate\/path/data\/cert\/harbor.sapphire.local.crt/' ~/harbor/harbor.yml
$ sed -i 's/your\/private\/key\/path/data\/cert\/harbor.sapphire.local.key/' ~/harbor/harbor.yml
```
可加入以下指令修改預設密碼：
```
sed -i 's/Harbor12345/<您的密碼>/' ~/harbor/harbor.yml
```

3. 執行安裝，可在此步驟同時安裝其他外掛如 ` --with-trivy`
```
$ sudo harbor/install.sh
```

4. 完成，輸入 `https://harbor.sapphire.local` 進入 harbor UI


![](https://i.imgur.com/V6HlSfc.png)
