# KodBox 可道雲安裝 

## 盡量先用Kodbox  問題不會比Nextcloud多 比較好用
### 一定要有portainer&docker
### 進portain => docker => Stacks 加入以下
```
version: '3.5'
services:
  db:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/kodbox/mnt/user/appdata/kodbox/db:/var/lib/mysql"
    environment:
      - "TZ=Asia/Taipei"
      - "MYSQL_ROOT_PASSWORD= 輸入自己的密碼"
      - "MYSQL_DATABASE=kodbox"
      - "MYSQL_USER=kodbox"
      - "MYSQL_PASSWORD=輸入自己的密碼"
    restart: always

  app:
    image: kodcloud/kodbox
    ports:
      - "8668:80"
    links:
      - db
      - redis
    volumes:
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/kodbox/site:/var/www/html"
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/kodbox/mnt/user:/data"
    restart: always

  redis:
    image: redis:alpine
    environment:
      - "TZ=Asia/Taipei"
    restart: always

```
## 程式完成打包成Docker鏡像:
1. 一定要先下載docker
2. 先在nas docker安裝python 3.9.9 slim
3. 創建tmp 資料夾 mkdir /home/qi/tmp  
4. cd /home/qi/tmp  
5. 先在tmp資料夾內上傳(app.py,templates,data,static)所有資料夾
6. nano Dockerfile
7. 在dockerfile內上傳 (Ctrl+o 儲存ctrl+x退出)
```
ROM python:3.9.9

RUN pip3 install numpy
RUN pip3 install Flask
RUN pip3 install mediapipe
RUN pip3 install Flask-Bcrypt
RUN pip3 install pymongo
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    libsm6 \
    libxext6 \
    libxrender-dev
RUN pip3 install opencv-python-headless
RUN mkdir -p /identify/
COPY ./app.py /identify
COPY ./templates /identify/templates
COPY ./static /identify/static
COPY ./data /identify/data
WORKDIR /identify
CMD [ "python", "/identify/app.py"]

```

## NextCloud 雲端安裝
### 一定要有portainer&docker
### 觀看 (https://www.youtube.com/watch?v=p0I8pikm2P4&t=1162s)
```
--- 
version: "2"
services: 
  app: 
    depends_on: 
      - db
    environment: 
      - MYSQL_PASSWORD=輸入自己密碼
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    image: nextcloud
    links: 
      - db
    ports: 
      - "8888:80"
    restart: always
    volumes: 
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/nextcloud:/var/www/html"
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/apps:/var/www/html/custom_apps"
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/config:/var/www/html/config"
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/data:/var/www/html/data"
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/theme:/var/www/html/themes/<YOUR_CUSTOM_THEME>"
  db: 
    command: "--transaction-isolation=READ-COMMITTED --binlog-format=ROW"
    environment: 
      - MYSQL_ROOT_PASSWORD=輸入自己密碼
      - MYSQL_PASSWORD=輸入自己密碼
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    image: "mariadb:10.5"
    restart: always
    volumes: 
      - "/srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/db:/var/lib/mysql"
```
### 在portainer的nextclud的終端機
1. ``` ls -a ```
2. ``` apt update ```
3. ```apt install nano ```
4. ```nano .htaccess``` 的最上層只接加上(以下)
```
php_value upload_max_filesize 16G
php_value post_max_size 16G
php_value max_input_time 3600
php_value max_execution_time 3600
php_value memory_limit 2048M
```
5. ctrl +o儲存ctrl +x 退出
6. nano config/config.php 或終端 cd /srv/dev-disk-by-uuid-0d415a2a-8d65-4885-9a4b-5097896f095d/Nextcloud/path/to/config    後sudo nano config.php
7. 加入
```
'trusted_domains' => 
  array (
    0 => '192.168.1.65:8080',
    1 => 'cloudnextcloudhomeabc.totosss0527.com',
  ),
```
8. 然後在'installed' => true,後面加入
```
'overwriteprotocol' => 'https',
'default_phone_region' => 'TW',
'enable_previews' => true,
'auth.bruteforce.protection.enabled' => false,
```






