# Immich 舊版安裝筆記（V3 以前）

> **此為舊版筆記，僅供參考。** 新安裝請改看 [Immich V3 / VectorChord 版](../README.md)。

## 目錄

- [安裝](#安裝)
- [更新](#更新)
- [換裝置前備份資料庫](#換裝置前備份資料庫)

## 安裝

1. 建立資料夾並下載官方設定檔：

   ```bash
   mkdir ./immich-app
   cd ./immich-app
   sudo wget https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
   sudo wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
   ```

2. 編輯 `.env`：

   ```bash
   sudo nano .env
   ```

   ```env
   # 所有環境變數說明：https://immich.app/docs/install/environment-variables

   # 照片儲存位置（改成自己的資料夾）
   UPLOAD_LOCATION=./library

   # Immich 版本，可固定為特定版本，例如 "v1.71.0"
   IMMICH_VERSION=release

   # PostgreSQL 密碼，請改成自己的隨機密碼
   DB_PASSWORD=<輸入自己的資料庫密碼>

   ###################################################################################
   # 以下通常不需要修改
   DB_HOSTNAME=immich_postgres
   DB_USERNAME=postgres
   DB_DATABASE_NAME=immich
   REDIS_HOSTNAME=immich_redis

   VITE_SERVER_ENDPOINT=https://<輸入自己的網域，例如 immich.example.com>
   ```

3. 啟動：

   ```bash
   sudo docker compose up -d
   ```

## 更新

```bash
docker compose pull && docker compose up -d
```

## 換裝置前備份資料庫

換裝置前，先把資料庫匯出成 SQL，再用 rsync 搬移整個資料夾。

1. 進入 Immich 根目錄：

   ```bash
   cd /srv/mergerfs/Total_Storage/primary-drive/immich
   ```

2. 設定密碼環境變數，讓 `pg_dump` 可以通過認證：

   ```bash
   export PGPASSWORD="<輸入自己的資料庫密碼>"   # 與 .env 的 DB_PASSWORD 相同
   ```

3. 匯出資料庫：

   ```bash
   docker exec -t immich_postgres pg_dump -U postgres -d immich > /srv/mergerfs/Total_Storage/primary-drive/immich/immich_db_backup.sql
   ```

4. 備份完成後清除密碼環境變數：

   ```bash
   unset PGPASSWORD
   ```
