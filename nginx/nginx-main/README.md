# Nginx Proxy Manager 反向代理

使用 Nginx Proxy Manager（NPM）把不同的網域導向 NAS 上不同的服務（例如 Immich、KodBox），並透過圖形介面申請免費的 SSL 憑證。

## 目錄

- [前置需求](#前置需求)
- [安裝](#安裝)
  - [1. 建立資料夾](#1-建立資料夾)
  - [2. 建立 config.json](#2-建立-configjson)
  - [3. 建立 docker-compose.yml](#3-建立-docker-composeyml)
  - [4. 啟動](#4-啟動)
- [登入管理後台](#登入管理後台)
- [申請 SSL 憑證](#申請-ssl-憑證)
- [調整上傳大小與逾時](#調整上傳大小與逾時)

## 前置需求

- NAS 已安裝 Docker 與 Portainer
- 已完成 [Cloudflare](../../Cloudflare/README.md) 與 [DuckDNS](../../dns-duckdns/dns-duckdns-main/README.md) 的 DNS 設定

## 安裝

### 1. 建立資料夾

```bash
mkdir nginx
cd nginx
```

### 2. 建立 config.json

```bash
nano config.json
```

```json
{
  "database": {
    "engine": "mysql",
    "host": "db",
    "name": "npm",
    "user": "npm",
    "password": "<輸入自己的資料庫密碼>",
    "port": 3306
  }
}
```

### 3. 建立 docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    ports:
      - '80:80'    # HTTP
      - '81:81'    # 管理後台
      - '443:443'  # HTTPS
    volumes:
      - ./config.json:/app/config/production.json
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt

  db:
    image: 'jc21/mariadb-aria:10.4.15-innodb'
    environment:
      MYSQL_ROOT_PASSWORD: '<輸入自己的資料庫root密碼>'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: '<輸入自己的資料庫密碼>'   # 要跟 config.json 的 password 一樣
    volumes:
      - ./data/mysql:/var/lib/mysql
```

### 4. 啟動

```bash
docker compose up -d
```

## 登入管理後台

開啟 `http://<你的NAS內網IP>:81`，使用預設帳號登入：

| 欄位 | 預設值 |
| :--- | :--- |
| 帳號 | `admin@example.com` |
| 密碼 | `changeme` |

> 第一次登入後請立即修改帳號與密碼。

## 申請 SSL 憑證

使用 Cloudflare DNS 驗證申請 Let's Encrypt 憑證時，需要 Cloudflare API Token：

1. 登入 Cloudflare → **My Profile** → **API Tokens** → **Create Token**。
2. 選擇 **Edit zone DNS** 範本（權限 `Zone:DNS:Edit`），指定自己的網域。
3. 在 NPM 的 **SSL Certificates** → **Add SSL Certificate** → **Let's Encrypt**，勾選 **Use a DNS Challenge**，DNS Provider 選擇 Cloudflare，並填入：

```ini
dns_cloudflare_api_token = <輸入自己的_Cloudflare_API_Token>
```

> **注意：** API Token 可以修改你的 DNS 紀錄，請勿公開。

## 調整上傳大小與逾時

上傳大檔案失敗或逾時，需要調整 Nginx 設定。進入容器修改：

```bash
docker exec -it nginx-app-1 /bin/sh
```

如果主機本身另外安裝了 Nginx，則編輯 `/etc/nginx/nginx.conf`：

```bash
sudo nano /etc/nginx/nginx.conf
```

在 `http { }` 區塊內加入：

```nginx
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
client_max_body_size 50M;
client_header_timeout 3600s;
client_body_timeout 3600s;
fastcgi_connect_timeout 3600s;
fastcgi_send_timeout 3600s;
fastcgi_read_timeout 3600s;
```

<details>
<summary>點此展開完整 nginx.conf 範例</summary>

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
        client_max_body_size 50M;
        client_header_timeout 3600s;
        client_body_timeout 3600s;
        fastcgi_connect_timeout 3600s;
        fastcgi_send_timeout 3600s;
        fastcgi_read_timeout 3600s;
}
```

</details>
