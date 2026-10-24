# DuckDNS 動態 DNS 設定

家用網路的對外 IP 通常會變動，使用 DuckDNS 可以把會變動的 IP 自動綁定到固定的網域名稱，讓你隨時都能從外網連回 NAS。

## 目錄

- [什麼是 DNS](#什麼是-dns)
  - [DNS 查詢流程](#dns-查詢流程)
- [為什麼使用 DuckDNS](#為什麼使用-duckdns)
- [事前準備](#事前準備)
- [方法一：使用 Docker Compose](#方法一使用-docker-compose)
- [方法二：使用 Portainer Stacks](#方法二使用-portainer-stacks)
- [確認是否成功](#確認是否成功)

## 什麼是 DNS

DNS（Domain Name System）負責把網域名稱（例如 `example.com`）轉換成 IP 位址（例如 `192.0.2.1`），讓人不需要記住 IP 也能連到網路服務。

### DNS 查詢流程

1. **本地快取**：電腦先檢查自己是否有快取紀錄。
2. **本地 DNS 伺服器**：查不到時，向 ISP 提供的 DNS 伺服器詢問。
3. **遞迴查詢**：DNS 伺服器依序向根伺服器、TLD 伺服器、權威 DNS 伺服器查詢。
4. **回傳結果**：取得 IP 後回傳給使用者的電腦。

## 為什麼使用 DuckDNS

DuckDNS 是免費的動態 DNS（DDNS）服務，適合把動態 IP 的設備綁定到固定網域。

- **免費**：個人與小型網路環境可免費使用。
- **支援動態 IP**：IP 改變時會自動更新，不需手動修改。
- **設定簡單**：支援網頁、API、路由器或伺服器腳本等多種更新方式。
- **用途廣泛**：適合 NAS、自架伺服器、遠端監控等需要外部連線的服務。
- **隱私友善**：只需要最少的個人資料。

## 事前準備

1. 到 [DuckDNS 官網](https://www.duckdns.org) 登入，建立自己的子網域。
2. 記下頁面上方顯示的 **token**（請勿公開）。
3. NAS 已安裝 Docker（建議搭配 Portainer）。
4. 查詢 PUID / PGID：在 NAS 上執行以下指令，輸出中的 `uid` 與 `gid` 就是要填的數字。

```bash
id <你的使用者名稱>
```

## 方法一：使用 Docker Compose

建立資料夾與設定檔：

```bash
mkdir duckdns
cd duckdns
nano docker-compose.yml
```

貼上以下內容：

```yaml
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    network_mode: host
    environment:
      - PUID=1000                              # 替換為你的 UID
      - PGID=100                               # 替換為你的 GID
      - TZ=Asia/Taipei
      - SUBDOMAINS=<輸入自己的子網域>           # 例如 myhome（不含 .duckdns.org）
      - TOKEN=<輸入自己的_DuckDNS_Token>        # duckdns.org 首頁上方的 token
      - UPDATE_IP=ipv4
      - LOG_FILE=true
    volumes:
      - ./config:/config
    restart: unless-stopped
```

啟動容器：

```bash
docker compose up -d
```

## 方法二：使用 Portainer Stacks

1. 進入 Portainer，選擇本機環境 → **Stacks** → **Add stack**。
2. 輸入名稱（例如 `duckdns`），將上方[方法一](#方法一使用-docker-compose)的 YAML 內容貼到編輯器中。
3. 點選 **Deploy the stack**。

## 確認是否成功

```bash
docker logs duckdns
```

日誌中出現更新成功的訊息，並且在 DuckDNS 網站上看到的 IP 與你家的對外 IP 一致，就代表設定完成。
