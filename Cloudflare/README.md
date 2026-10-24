# Cloudflare Tunnel 建立與啟動指南

使用 Docker 執行 Cloudflare Tunnel，不需要在路由器開放任何連接埠，就能把 NAS 內網的服務安全地公開到外網（內網穿透）。

## 目錄

- [前置需求](#前置需求)
- [步驟一：建立專案資料夾](#步驟一建立專案資料夾)
- [步驟二：取得 Tunnel Token](#步驟二取得-tunnel-token)
- [步驟三：建立 docker-compose.yaml](#步驟三建立-docker-composeyaml)
- [步驟四：啟動服務](#步驟四啟動服務)
- [常用維護指令](#常用維護指令)

## 前置需求

- NAS 已安裝 Docker 與 Docker Compose
- 擁有一個 Cloudflare 帳號，並已將自己的網域交給 Cloudflare 管理

## 步驟一：建立專案資料夾

```bash
mkdir -p Cloudflare && cd Cloudflare
```

> 本文範例路徑為 `/srv/mergerfs/Total_Storage/primary-drive/Cloudflare`，請依自己的環境調整。

## 步驟二：取得 Tunnel Token

1. 登入 Cloudflare，進入 **Zero Trust** → **Networks** → **Tunnels**。
2. 點選 **Create a tunnel**，類型選擇 **Cloudflared**，輸入 Tunnel 名稱。
3. 安裝方式選擇 **Docker**，複製畫面上指令中 `--token` 後面那一長串文字。
4. 在 **Public Hostname** 頁籤設定要對外的網域與內網服務位址（例如 `http://<你的NAS內網IP>:2283`）。

> **注意：** Token 等同密碼，任何拿到的人都能使用你的 Tunnel，請勿公開或上傳到 GitHub。

## 步驟三：建立 docker-compose.yaml

```bash
nano docker-compose.yaml
```

貼上以下內容（`Ctrl+O` 儲存，`Ctrl+X` 離開）：

```yaml
services:
  cloudflare-tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare-tunnel
    restart: always
    command: tunnel --no-autoupdate run --token <輸入自己的_Cloudflare_Tunnel_Token>
```

## 步驟四：啟動服務

在同一個資料夾執行：

```bash
docker compose up -d
```

已設定 `restart: always`，NAS 重新開機後會自動啟動。

## 常用維護指令

| 用途 | 指令 |
| :--- | :--- |
| 查看連線日誌（確認 Tunnel 是否 Ready） | `docker compose logs -f` |
| 檢查容器狀態 | `docker compose ps` |
| 更新到最新版 | `docker compose pull && docker compose up -d` |
| 停止服務 | `docker compose down` |
