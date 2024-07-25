# 自架NAS  NGINX 反向代理
## 必需要有docker,portainer
1. mkdir nginx
2. cd nginx
3. nano config.json
```
{ 
  "database": { 
     "engine": "mysql", 
     "host": "db", 
     "name": "npm", 
     "user": "npm", 
     "password": "npm", 
     "port": 3306
   }
 }

```
4. nano docker-compose.yml
```
--- 
version: '3' 
services: 
  app: 
    image: 'jc21/nginx-proxy-manager:latest' 
    ports: 
      - '80:80' #HTTP Traffic 
      - '81:81' #Dashboard Port
      - '443:443' #HTTPS Traffic 
    volumes: 
      - ./config.json:/app/config/production.json 
      - ./data:/data 
      - ./letsencrypt:/etc/letsencrypt
  db: 
    image: 'jc21/mariadb-aria:10.4.15-innodb'
    environment: 
      MYSQL_ROOT_PASSWORD: 'npm' 
      MYSQL_DATABASE: 'npm' 
      MYSQL_USER: 'npm' 
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./data/mysql:/var/lib/mysql

```
5. 初始帳號:admin@example.com
6. 先用好CloudFlare與 Duckdns DNS 
7. 申請SSL 需要 CloudFlare API 


