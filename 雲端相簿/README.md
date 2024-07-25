# 自架NAS Immich  相簿
1. mkdir ./immich-app
2. cd ./immich-app
3. sudo wget https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
4. sudo wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
5. sudo nano .env   #進去裡面修改資料
```
# You can find documentation for all the supported env variables at https://immich.app/docs/install/environment-variables
# The location where your uploaded files are stored
UPLOAD_LOCATION=./library   #自己的檔案夾
# The Immich version to use. You can pin this to a specific version like "v1.71.0"
IMMICH_VERSION=release
# Connection secret for postgres. You should change it to a random password
DB_PASSWORD=postgres
# The values below this line do not need to be changed
###################################################################################
DB_HOSTNAME=immich_postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
REDIS_HOSTNAME=immich_redis

VITE_SERVER_ENDPOINT=https://immich.totosss0527.com

```
6. 更新 docker compose pull && docker compose up -d
7. 啟動 sudo docker compose up -d