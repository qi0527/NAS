# 自架 NAS 筆記

從零開始用 OpenMediaVault 建立家用 NAS 的完整筆記，包含硬碟合併與備份、Docker 服務架設、外網連線與反向代理。

## 目錄

- [架構概覽](#架構概覽)
- [文件列表](#文件列表)
- [建議閱讀順序](#建議閱讀順序)
- [使用前注意](#使用前注意)
- [授權](#授權)

## 架構概覽

```text
外網使用者
   │
   ▼
Cloudflare Tunnel / DuckDNS   ← 外網連線、動態 DNS
   │
   ▼
Nginx Proxy Manager           ← 反向代理、SSL 憑證
   │
   ▼
Docker 服務：Immich、KodBox / Nextcloud
   │
   ▼
OpenMediaVault + MergerFS + SnapRAID + UPS
```

## 文件列表

| 主題 | 說明 |
| :--- | :--- |
| [OpenMediaVault](openmediavault/openmediavault-nas/README.md) | NAS 系統安裝、Docker、Portainer、K3s 串接 |
| [MergerFS + SnapRAID](openmediavault/openmediavault-nas/Mergerfs-SnapRAID/README.md) | 合併多顆硬碟並建立校驗備份 |
| [UPS](openmediavault/openmediavault-nas/UPS/README.md) | 不斷電系統設定，停電自動關機 |
| [DuckDNS](dns-duckdns/dns-duckdns-main/README.md) | 免費動態 DNS |
| [Cloudflare Tunnel](Cloudflare/README.md) | 免開連接埠的內網穿透 |
| [Nginx Proxy Manager](nginx/nginx-main/README.md) | 反向代理與 SSL 憑證 |
| [Immich](immich/README.md) | 自架相簿（Google 相簿替代方案） |
| [KodBox / Nextcloud](kodbox/kodbox-main/README.md) | 私有雲端硬碟 |

## 建議閱讀順序

1. [OpenMediaVault](openmediavault/openmediavault-nas/README.md)：安裝系統、Docker 與 Portainer
2. [MergerFS + SnapRAID](openmediavault/openmediavault-nas/Mergerfs-SnapRAID/README.md)：建立儲存池
3. [UPS](openmediavault/openmediavault-nas/UPS/README.md)：設定停電保護
4. [DuckDNS](dns-duckdns/dns-duckdns-main/README.md) 或 [Cloudflare Tunnel](Cloudflare/README.md)：設定外網連線
5. [Nginx Proxy Manager](nginx/nginx-main/README.md)：設定網域與 SSL
6. [Immich](immich/README.md)、[KodBox](kodbox/kodbox-main/README.md)：架設應用服務

## 使用前注意

- 文件中所有 `<輸入自己的...>` 都是佔位符，請換成自己的密碼、Token、網域或 IP。
- 範例路徑 `/srv/mergerfs/Total_Storage/primary-drive/` 請依自己的環境調整。
- `.env`、資料庫備份（`.sql`）以及任何含有密碼或 Token 的檔案，**請勿上傳到 GitHub**。

## 授權

本專案採用 [MIT License](LICENSE)。
