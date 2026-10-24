# Immich 自架相簿（V3 / VectorChord 版）

Immich 是可自架的 Google 相簿替代方案，支援手機自動備份、人臉辨識、地圖與搜尋。本文以 **v3.0.3** 為例，說明全新安裝、日常更新、備份，以及從舊版升級到 V3 的方法。

> 本文範例路徑為 `/srv/mergerfs/Total_Storage/primary-drive/immich`，請依自己的環境調整。

## 目錄

- [重要原則](#重要原則)
- [全新安裝](#全新安裝)
  - [1. 建立資料夾](#1-建立資料夾)
  - [2. 下載官方設定檔](#2-下載官方設定檔)
  - [3. 編輯 .env](#3-編輯-env)
  - [4. docker-compose.yml 範例](#4-docker-composeyml-範例)
  - [5. 啟動 Immich](#5-啟動-immich)
- [日常更新流程](#日常更新流程)
  - [1. 確認目前版本](#1-確認目前版本)
  - [2. 設定要更新的版本](#2-設定要更新的版本)
  - [3. 更新前備份](#3-更新前備份)
  - [4. 修改 .env 版本](#4-修改-env-版本)
  - [5. 更新容器](#5-更新容器)
  - [6. 確認更新成功](#6-確認更新成功)
  - [7. 備份更新後的狀態](#7-備份更新後的狀態)
  - [8. 功能測試](#8-功能測試)
  - [9. 刪除舊版備份](#9-刪除舊版備份)
  - [一鍵更新指令](#一鍵更新指令)
- [完整備份整個 Immich 資料夾](#完整備份整個-immich-資料夾)
- [從舊版升級到 V3](#從舊版升級到-v3)
  - [1. 先固定舊版版本號](#1-先固定舊版版本號)
  - [2. 將資料庫 image 改成 VectorChord](#2-將資料庫-image-改成-vectorchord)
  - [3. 重建容器](#3-重建容器)
  - [4. 檢查 extension](#4-檢查-extension)
  - [5. 升級到 V3](#5-升級到-v3)
- [必要檔案與備份保留建議](#必要檔案與備份保留建議)

## 重要原則

1. **V3 之後請使用 VectorChord 資料庫**，不要再使用舊的 `pgvecto-rs`。

   | 版本 | 資料庫 image |
   | :--- | :--- |
   | V3（建議） | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` |
   | V3 以前（舊） | `docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0` |

2. **固定版本號，不要使用 `release`**。使用 `release` 可能會自動跳到新的大版本，造成升級失敗。

   ```env
   # 不建議
   IMMICH_VERSION=release

   # 建議
   IMMICH_VERSION=v3.0.3
   ```

3. **更新前一定先備份**，流程為：改 `.env` → 備份目前版本 → 更新 → 成功後備份新版 → 穩定後刪除舊備份。

## 全新安裝

### 1. 建立資料夾

```bash
mkdir -p /srv/mergerfs/Total_Storage/primary-drive/immich
cd /srv/mergerfs/Total_Storage/primary-drive/immich
```

### 2. 下載官方設定檔

```bash
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

下載後請檢查 `docker-compose.yml`，確認資料庫使用的是 VectorChord 版本。

### 3. 編輯 .env

```bash
nano .env
```

建議內容：

```env
# 照片與影片儲存位置
UPLOAD_LOCATION=./library

# PostgreSQL 資料庫儲存位置
DB_DATA_LOCATION=./postgres

# 時區
TZ=Asia/Taipei

# Immich 版本（請固定版本號，不要使用 release）
IMMICH_VERSION=v3.0.3

# PostgreSQL 密碼
# 請改成自己的隨機密碼（建議只用英文與數字 A-Za-z0-9）
# 資料庫建立後，不要隨便更改這個密碼
DB_PASSWORD=<輸入自己的資料庫密碼>

###################################################################################
# 以下通常不需要修改
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
DB_HOSTNAME=immich_postgres
REDIS_HOSTNAME=immich_redis
```

### 4. docker-compose.yml 範例

```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/redis:6.2-alpine@sha256:2ba50e1ac3a0ea17b736ce9db2b0a9f6f8b85d4c27d5f5accc6a416d8f42c6d5
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # 如果資料庫放在 HDD / mergerfs 硬碟池，可以取消下面這行註解
      # DB_STORAGE_TYPE: 'HDD'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always

volumes:
  model-cache:
```

### 5. 啟動 Immich

```bash
cd /srv/mergerfs/Total_Storage/primary-drive/immich
docker compose pull
docker compose up -d
```

查看狀態，四個容器都應該是 `healthy`：

```bash
docker compose ps
```

```text
immich_server             Up / healthy
immich_machine_learning   Up / healthy
immich_postgres           Up / healthy
immich_redis              Up / healthy
```

查看日誌：

```bash
docker logs -f immich_server
```

啟動完成後，用瀏覽器開啟 `http://<你的NAS內網IP>:2283` 建立管理員帳號。

## 日常更新流程

適用於 V3 內的小版本更新（例如 `v3.0.1 → v3.0.3`）。資料庫已經是 VectorChord 的話，不需要重新做資料庫遷移。

```text
確認目前版本
   ↓
備份目前設定檔與資料庫
   ↓
修改 .env 版本
   ↓
docker compose pull
   ↓
docker compose up -d --force-recreate
   ↓
確認 healthy
   ↓
備份更新成功後的版本
   ↓
穩定一週後刪除舊備份
```

### 1. 確認目前版本

```bash
cd /srv/mergerfs/Total_Storage/primary-drive/immich
grep IMMICH_VERSION .env
```

### 2. 設定要更新的版本

以更新到 `v3.0.3` 為例：

```bash
OLD_VERSION=$(grep '^IMMICH_VERSION=' .env | cut -d= -f2)
NEW_VERSION="v3.0.3"
NOW=$(date +%F_%H%M)

echo "目前版本：$OLD_VERSION"
echo "準備更新到：$NEW_VERSION"
```

### 3. 更新前備份

備份設定檔：

```bash
cp docker-compose.yml docker-compose.${OLD_VERSION}.before-${NEW_VERSION}.${NOW}.yml
cp .env .env.${OLD_VERSION}.before-${NEW_VERSION}.${NOW}
```

備份資料庫：

```bash
source .env
docker exec -t immich_postgres pg_dumpall -c -U "$DB_USERNAME" > immich_db_backup_${OLD_VERSION}_before_${NEW_VERSION}_${NOW}.sql
```

會產生類似以下檔案：

```text
docker-compose.v3.0.1.before-v3.0.3.2026-07-17_1318.yml
.env.v3.0.1.before-v3.0.3.2026-07-17_1318
immich_db_backup_v3.0.1_before_v3.0.3_2026-07-17_1318.sql
```

### 4. 修改 .env 版本

```bash
sed -i "s/^IMMICH_VERSION=.*/IMMICH_VERSION=${NEW_VERSION}/" .env
grep IMMICH_VERSION .env
```

應該看到 `IMMICH_VERSION=v3.0.3`。

### 5. 更新容器

```bash
docker compose pull
docker compose up -d --force-recreate
```

### 6. 確認更新成功

```bash
docker compose ps
docker logs --tail=100 immich_server
```

確認 image 版本：

```bash
docker inspect immich_server --format '{{.Config.Image}}'
docker inspect immich_machine_learning --format '{{.Config.Image}}'
docker inspect immich_postgres --format '{{.Config.Image}}'
```

正常應類似：

```text
ghcr.io/immich-app/immich-server:v3.0.3
ghcr.io/immich-app/immich-machine-learning:v3.0.3
ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
```

### 7. 備份更新後的狀態

```bash
NOW=$(date +%F_%H%M)

cp docker-compose.yml docker-compose.${NEW_VERSION}.working.${NOW}.yml
cp .env .env.${NEW_VERSION}.working.${NOW}

source .env
docker exec -t immich_postgres pg_dumpall -c -U "$DB_USERNAME" > immich_db_backup_${NEW_VERSION}_${NOW}.sql
```

### 8. 功能測試

更新後建議逐一確認：

- 手機自動備份
- 照片瀏覽、影片播放
- 相簿、搜尋
- 人臉辨識、地圖
- 分享連結
- 登入與登出

### 9. 刪除舊版備份

新版本穩定運作一週後，再刪除舊版本的備份。先確認檔案：

```bash
ls -la
```

刪除舊版備份（以下檔名為範例，請換成實際檔名）：

```bash
rm -i \
  docker-compose.v3.0.1.before-v3.0.3.2026-07-17_1318.yml \
  .env.v3.0.1.before-v3.0.3.2026-07-17_1318 \
  immich_db_backup_v3.0.1_before_v3.0.3_2026-07-17_1318.sql
```

`rm -i` 會逐一詢問 `remove regular file 'xxx'?`，確認是舊備份後輸入 `y`。

### 一鍵更新指令

只需要修改 `NEW_VERSION` 這一行，即可完成上述第 1 到第 7 步：

```bash
cd /srv/mergerfs/Total_Storage/primary-drive/immich

OLD_VERSION=$(grep '^IMMICH_VERSION=' .env | cut -d= -f2)
NEW_VERSION="v3.0.3"
NOW=$(date +%F_%H%M)

echo "目前版本：$OLD_VERSION"
echo "準備更新到：$NEW_VERSION"

# 更新前備份
cp docker-compose.yml docker-compose.${OLD_VERSION}.before-${NEW_VERSION}.${NOW}.yml
cp .env .env.${OLD_VERSION}.before-${NEW_VERSION}.${NOW}
source .env
docker exec -t immich_postgres pg_dumpall -c -U "$DB_USERNAME" > immich_db_backup_${OLD_VERSION}_before_${NEW_VERSION}_${NOW}.sql

# 修改版本並更新
sed -i "s/^IMMICH_VERSION=.*/IMMICH_VERSION=${NEW_VERSION}/" .env
grep IMMICH_VERSION .env
docker compose pull
docker compose up -d --force-recreate

# 檢查狀態
docker compose ps
docker logs --tail=100 immich_server

# 更新後備份
NOW_AFTER=$(date +%F_%H%M)
cp docker-compose.yml docker-compose.${NEW_VERSION}.working.${NOW_AFTER}.yml
cp .env .env.${NEW_VERSION}.working.${NOW_AFTER}
source .env
docker exec -t immich_postgres pg_dumpall -c -U "$DB_USERNAME" > immich_db_backup_${NEW_VERSION}_${NOW_AFTER}.sql

echo "更新完成，請確認 Immich 網頁、手機備份、照片瀏覽、搜尋、人臉辨識是否正常。"
```

## 完整備份整個 Immich 資料夾

更新大版本前，除了 SQL 備份，也建議把整個 Immich 資料夾壓縮備份。

1. 進入目錄並先做 SQL 備份：

   ```bash
   cd /srv/mergerfs/Total_Storage/primary-drive/immich
   source .env
   docker exec -t immich_postgres pg_dumpall -c -U "$DB_USERNAME" > immich_db_backup_full_$(date +%F_%H%M).sql
   ```

2. 停止 Immich：

   ```bash
   docker compose down
   ```

3. 回到上一層壓縮整個資料夾：

   ```bash
   cd /srv/mergerfs/Total_Storage/primary-drive
   BACKUP_NAME="immich_backup_$(date +%F_%H%M).tar.gz"
   sudo tar --acls --xattrs -czpf "$BACKUP_NAME" immich
   sudo chown "$USER:$USER" "$BACKUP_NAME"
   ls -lh "$BACKUP_NAME"
   ```

   壓縮檔會包含 `.env`、`docker-compose.yml`、`library/`、`postgres/` 與 SQL 備份檔。

4. 重新啟動 Immich：

   ```bash
   cd /srv/mergerfs/Total_Storage/primary-drive/immich
   docker compose up -d
   ```

## 從舊版升級到 V3

如果原本是 V3 以前的 Immich，資料庫可能還是舊的 `pgvecto-rs`。**不能直接把 Immich 升到 V3**，否則會出現以下錯誤：

```text
No vector extension found
microservices worker exited with code 1
Killing api process
```

正確的升級順序：

```text
舊版 Immich + pgvecto-rs
   ↓
先把資料庫換成 VectorChord（Immich 版本先不升）
   ↓
確認 vchord / vector extension 正常
   ↓
再把 Immich 升到 V3
```

### 1. 先固定舊版版本號

以 `v2.7.5` 為例，在 `.env` 中設定：

```env
IMMICH_VERSION=v2.7.5
```

### 2. 將資料庫 image 改成 VectorChord

```yaml
database:
  container_name: immich_postgres
  image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
  environment:
    POSTGRES_PASSWORD: ${DB_PASSWORD}
    POSTGRES_USER: ${DB_USERNAME}
    POSTGRES_DB: ${DB_DATABASE_NAME}
    POSTGRES_INITDB_ARGS: '--data-checksums'
  volumes:
    - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
  shm_size: 128mb
  restart: always
```

同時刪除舊版資料庫設定中的 `command`（含 `shared_preload_libraries=vectors.so` 那段）以及舊的資料庫 `healthcheck`：

```yaml
command:
  [
    'postgres',
    '-c',
    'shared_preload_libraries=vectors.so',
    ...
  ]
```

### 3. 重建容器

```bash
docker compose down
docker compose pull
docker compose up -d --force-recreate
```

### 4. 檢查 extension

```bash
docker exec -it immich_postgres psql -U postgres -d immich -c "SELECT extname, extversion FROM pg_extension ORDER BY extname;"
```

應該看到 `vchord` 與 `vector`，例如：

```text
vchord | 0.4.3
vector | 0.8.1
```

### 5. 升級到 V3

```bash
sed -i 's/^IMMICH_VERSION=.*/IMMICH_VERSION=v3.0.3/' .env
grep IMMICH_VERSION .env

docker compose pull
docker compose up -d --force-recreate
docker compose ps
```

## 必要檔案與備份保留建議

Immich 正常運作只需要以下檔案：

```text
.env
docker-compose.yml
library/
postgres/
```

建議至少保留一份「目前成功版本」的設定檔與 SQL 備份：

```text
docker-compose.yml
.env
docker-compose.v3.0.3.working.<時間>.yml
.env.v3.0.3.working.<時間>
immich_db_backup_v3.0.3_<時間>.sql
library/
postgres/
```

舊版本的備份等新版本穩定一週後再刪除。

> **注意：** `.env` 與 `.sql` 備份檔內含資料庫密碼與資料，請勿上傳到 GitHub。
