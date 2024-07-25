# 自架NAS DNS處理需要 DuckDNS
1. 網址: (https://www.duckdns.org)
2. 先輸入自己的網域
3. 需要有portainer與docker
4. 進portain => docker => Stacks 加入以下
```
---
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    network_mode: host #optional
    environment:
      - PUID=1000 #optional
      - PGID=100  #optional
      - TZ=Etc/UTC #optional
      - SUBDOMAINS=totosss0527  輸入自己的
      - TOKEN= 輸入自己的
      - UPDATE_IP=ipv4 #optional
      - LOG_FILE=true #optional
    volumes:
      - /DNS/duckdns/config:/config #optional
    restart: unless-stopped
```