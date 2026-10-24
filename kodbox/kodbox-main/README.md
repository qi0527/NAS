# KodBox（可道雲）與 Nextcloud 私有雲安裝

用 Docker 架設私有雲端硬碟。**建議優先使用 KodBox**，實際使用下來問題比 Nextcloud 少、也比較好用；Nextcloud 的安裝方式放在後半段供參考。

> 本文範例路徑為 `/srv/mergerfs/Total_Storage/primary-drive/KodBox`，請依自己的環境調整。

## 目錄

- [KodBox 安裝](#kodbox-安裝)
  - [1. 進入資料夾](#1-進入資料夾)
  - [2. 建立 docker-compose.yml](#2-建立-docker-composeyml)
  - [3. 掛載其他硬碟（選用）](#3-掛載其他硬碟選用)
  - [4. 啟動](#4-啟動)
- [KodBox 更新](#kodbox-更新)
  - [方法一：使用指令](#方法一使用指令)
  - [方法二：使用 Portainer](#方法二使用-portainer)
- [KodBox 疑難排解](#kodbox-疑難排解)
  - [掛載資料夾權限問題](#掛載資料夾權限問題)
  - [換裝置前備份資料庫](#換裝置前備份資料庫)
  - [換裝置後 MariaDB 無法啟動](#換裝置後-mariadb-無法啟動)
- [Nextcloud 安裝](#nextcloud-安裝)
  - [1. 建立 Stack](#1-建立-stack)
  - [2. 調整上傳檔案大小限制](#2-調整上傳檔案大小限制)
  - [3. 設定信任網域與 HTTPS](#3-設定信任網域與-https)

## KodBox 安裝

### 1. 進入資料夾

```bash
mkdir -p /srv/mergerfs/Total_Storage/primary-drive/KodBox
cd /srv/mergerfs/Total_Storage/primary-drive/KodBox
```

### 2. 建立 docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
services:
  db:
    image: mariadb:10.11
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - "./mnt/user/appdata/kodbox/db:/var/lib/mysql"
    environment:
      - "TZ=Asia/Taipei"
      - "MYSQL_ROOT_PASSWORD=<輸入自己的資料庫root密碼>"
      - "MYSQL_DATABASE=kodbox"
      - "MYSQL_USER=kodbox"
      - "MYSQL_PASSWORD=<輸入自己的資料庫密碼>"
    restart: always

  app:
    image: kodcloud/kodbox:1.63
    ports:
      - "8668:80"
    links:
      - db
      - redis
    volumes:
      - "./site:/var/www/html"
      - "./user:/data"
    restart: always

  redis:
    image: redis:alpine
    environment:
      - "TZ=Asia/Taipei"
    restart: always
```

> MariaDB 請固定使用 `mariadb:10.11`，不要用 `latest`，避免換機或更新時版本不相容。

### 3. 掛載其他硬碟（選用）

如果要讓 KodBox 讀取其他硬碟，在 `app` 的 `volumes` 加上一行：

```yaml
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/2T:/mnt/2T"
```

### 4. 啟動

```bash
docker compose up -d
```

啟動後開啟 `http://<你的NAS內網IP>:8668`，依照畫面完成初始設定。資料庫主機填 `db`，帳號密碼填 `docker-compose.yml` 中設定的值。

## KodBox 更新

### 方法一：使用指令

```bash
cd /srv/mergerfs/Total_Storage/primary-drive/KodBox
docker compose pull && docker compose up -d
```

更新完成後通常會自動啟動。如果沒有，再執行 `sudo docker compose up -d`。

### 方法二：使用 Portainer

1. 點選 **Images**。
2. 在 **Pull image** 輸入 `kodcloud/kodbox:latest`，點選 **Pull the image**。
3. 到 **Containers** 選取 KodBox → **Duplicate/Edit** → **Deploy the container**，即會使用剛下載的新映像重新啟動。

## KodBox 疑難排解

### 掛載資料夾權限問題

上傳檔案失敗或無法寫入時，通常是資料夾權限不符。先查詢容器內 nginx 使用者的 UID：

```bash
docker exec -it kodbox-app-1 id nginx
```

輸出範例：

```text
uid=101(nginx) gid=101(nginx) groups=101(nginx),82(www-data),101(nginx)
```

將資料夾擁有者改成相同的 UID / GID，然後重新啟動：

```bash
sudo chown -R 101:101 /srv/mergerfs/Total_Storage/primary-drive/KodBox/user
docker compose down && docker compose up -d
```

### 換裝置前備份資料庫

換裝置前，先把資料庫匯出成 SQL，再用 rsync 搬移整個資料夾。

1. 進入 KodBox 根目錄：

   ```bash
   cd /srv/mergerfs/Total_Storage/primary-drive/KodBox
   ```

2. 設定密碼環境變數：

   ```bash
   export MYSQL_ROOT_PASSWORD="<輸入自己的資料庫root密碼>"
   ```

3. 以 root 身份匯出 kodbox 資料庫：

   ```bash
   docker exec -t kodbox-db-1 mariadb-dump -u root -p"$MYSQL_ROOT_PASSWORD" kodbox > /srv/mergerfs/Total_Storage/primary-drive/KodBox/kodbox_db_backup.sql
   ```

4. 清除密碼環境變數：

   ```bash
   unset MYSQL_ROOT_PASSWORD
   ```

### 換裝置後 MariaDB 無法啟動

換裝置後 MariaDB 不斷重啟，常見錯誤如下：

| 錯誤訊息 | 原因 |
| :--- | :--- |
| `[ERROR] Can't init tc log` / `Aborting` | `tc.log` 損壞 |
| `undo tablespace ... exists. Creating system tablespace ... is not supported.` | InnoDB 系統檔案不一致 |

根本原因幾乎都是 **MariaDB 版本不同**，將 image 固定為 `mariadb:10.11` 後，依下列步驟重建資料庫並匯入備份。

> **警告：** 以下步驟會刪除資料庫系統檔，請務必先準備好 `kodbox_db_backup.sql`。

```bash
# 1. 停止並移除資料庫容器
cd /srv/mergerfs/Total_Storage/primary-drive/KodBox
docker stop kodbox-db-1
docker rm kodbox-db-1

# 2. 刪除損壞的系統檔
cd /srv/mergerfs/Total_Storage/primary-drive/KodBox/mnt/user/appdata/kodbox/db
sudo rm -f ibdata1 ib_logfile* undo* tc.log aria* ib_buffer_pool
sudo rm -rf mysql/ performance_schema/ sys/ mariadb_upgrade_info/

# 3. 重新啟動
cd /srv/mergerfs/Total_Storage/primary-drive/KodBox
docker compose up -d

# 4. 匯入備份
export MYSQL_ROOT_PASSWORD="<輸入自己的資料庫root密碼>"
docker exec -i kodbox-db-1 mariadb -u root -p"$MYSQL_ROOT_PASSWORD" kodbox < kodbox_db_backup.sql
unset MYSQL_ROOT_PASSWORD
```

## Nextcloud 安裝

需要先安裝 Docker 與 Portainer。參考影片：[Nextcloud 安裝教學](https://www.youtube.com/watch?v=p0I8pikm2P4&t=1162s)

### 1. 建立 Stack

在 Portainer → **Stacks** → **Add stack** 貼上以下內容：

```yaml
services:
  app:
    image: nextcloud
    depends_on:
      - db
    links:
      - db
    environment:
      - MYSQL_PASSWORD=<輸入自己的資料庫密碼>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    ports:
      - "8888:80"
    restart: always
    volumes:
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/Nextcloud/path/to/nextcloud:/var/www/html"
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/Nextcloud/path/to/apps:/var/www/html/custom_apps"
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/Nextcloud/path/to/config:/var/www/html/config"
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/Nextcloud/path/to/data:/var/www/html/data"

  db:
    image: "mariadb:10.5"
    command: "--transaction-isolation=READ-COMMITTED --binlog-format=ROW"
    environment:
      - MYSQL_ROOT_PASSWORD=<輸入自己的資料庫root密碼>
      - MYSQL_PASSWORD=<輸入自己的資料庫密碼>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    restart: always
    volumes:
      - "/srv/dev-disk-by-uuid-<你的硬碟UUID>/Nextcloud/path/to/db:/var/lib/mysql"
```

### 2. 調整上傳檔案大小限制

在 Portainer 開啟 Nextcloud 容器的終端機（Console），執行：

```bash
apt update
apt install nano
nano .htaccess
```

在 `.htaccess` 最上方加入以下內容，`Ctrl+O` 儲存、`Ctrl+X` 離開：

```apache
php_value upload_max_filesize 16G
php_value post_max_size 16G
php_value max_input_time 3600
php_value max_execution_time 3600
php_value memory_limit 2048M
```

### 3. 設定信任網域與 HTTPS

編輯 `config.php`（在容器內為 `config/config.php`，或在 NAS 上編輯對應的 config 資料夾）：

```bash
nano config/config.php
```

加入允許連線的網域：

```php
'trusted_domains' =>
  array (
    0 => '<你的NAS內網IP>:8888',
    1 => '<輸入自己的網域，例如 nextcloud.example.com>',
  ),
```

並在 `'installed' => true,` 後面加入：

```php
'overwriteprotocol' => 'https',
'default_phone_region' => 'TW',
'enable_previews' => true,
'auth.bruteforce.protection.enabled' => false,
```
