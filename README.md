# 自架NAS OpenMediaVault

## 主要學習:
1. 基礎安裝
2. omv 掛件
3. 安裝Portainer與Docker


## OMV基礎安裝:
1. OpenMediaVault官網安裝 (https://www.openmediavault.org/download.html)
2. 灌系統時root名字只接空白按enter就好密碼也是輸入一般使用者名字和密碼


## OMV 掛件:
1. 觀看 (https://www.youtube.com/watch?v=UawYG_itpM4)
2. ```sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/  install | sudo bash```
3. 再來插件區下載openmediavault-compose掛件一定要先下載

## 安裝Portainer與Docker:
1. 一定要先下載docker
2. ```  docker run -d -p 8000:8000 -p 9443:9443 -p 9000:9000 --name portainer --restart=always -v ```
3. ``` /var/run/docker.sock:/var/run/docker.sock -v ```
4. ``` portainer_data:/data portainer/portainer-ce:latest```
5. 安裝好後進入網頁更改 Environment => Name= local => Public IP= 自己nas的ip(192.168.0.152)