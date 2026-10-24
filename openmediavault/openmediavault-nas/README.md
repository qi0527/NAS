# OpenMediaVault 自架 NAS

以 OpenMediaVault（OMV）作為 NAS 作業系統，安裝 Docker 與 Portainer 管理容器，並用 Portainer 串接 K3s 叢集查看服務日誌。

## 目錄

- [學習重點](#學習重點)
- [安裝 OMV](#安裝-omv)
- [安裝 OMV 外掛與 Docker](#安裝-omv-外掛與-docker)
- [安裝 Portainer](#安裝-portainer)
  - [安裝](#安裝)
  - [初始設定](#初始設定)
  - [更新](#更新)
- [Portainer 串接 K3s 叢集](#portainer-串接-k3s-叢集)
  - [1. 在 K3s 伺服器安裝 Agent](#1-在-k3s-伺服器安裝-agent)
  - [2. 將服務改為 NodePort](#2-將服務改為-nodeport)
  - [3. 查詢機器的實體 IPv4 位址](#3-查詢機器的實體-ipv4-位址)
  - [4. 在 Portainer 新增環境](#4-在-portainer-新增環境)
  - [5. 查看服務日誌](#5-查看服務日誌)
- [疑難排解](#疑難排解)
  - [Docker 異常時重新啟動](#docker-異常時重新啟動)
  - [清理未使用的 Docker 資源](#清理未使用的-docker-資源)
  - [Portainer 無法連線本機環境（Docker 29 以上）](#portainer-無法連線本機環境docker-29-以上)
  - [查詢 OMV 無故關機原因](#查詢-omv-無故關機原因)

## 學習重點

1. OMV 基礎安裝
2. OMV 外掛（omv-extras）
3. 安裝 Docker 與 Portainer
4. Portainer 串接並管理 K3s 叢集（查看 Pod 日誌）

延伸文件：

- [MergerFS + SnapRAID 合併硬碟與備份](Mergerfs-SnapRAID/README.md)
- [UPS 不斷電系統設定](UPS/README.md)

## 安裝 OMV

1. 到 [OpenMediaVault 官網](https://www.openmediavault.org/download.html) 下載 ISO 並安裝。
2. 安裝過程中設定 root 時可直接留空按 Enter，之後輸入一般使用者的帳號與密碼。
3. 安裝完成後，用瀏覽器開啟 `http://<你的NAS內網IP>`。
4. 預設帳密為 `admin` / `openmediavault`，**登入後請立即修改密碼**。

## 安裝 OMV 外掛與 Docker

1. 在電腦上安裝 SSH 軟體連線到 OMV，例如 [FinalShell](https://hostbuf.net/)。
2. 參考影片：[OMV 外掛安裝教學](https://www.youtube.com/watch?v=UawYG_itpM4)
3. 一鍵安裝 omv-extras：

   ```bash
   wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | sudo bash
   ```

4. 在 OMV 網頁的外掛區安裝 `openmediavault-compose`（一定要先安裝）。
5. 安裝 curl：

   ```bash
   sudo apt-get install curl
   ```

6. 安裝 Docker：

   ```bash
   curl -o docker.sh -L https://get.docker.com
   sudo sh docker.sh
   ```

## 安裝 Portainer

### 安裝

必須先安裝好 Docker。

Linux：

```bash
sudo docker run -d --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -e TZ=Asia/Taipei \
  --restart=always \
  portainer/portainer-ce:latest
```

Windows：

```bash
docker run -d --name portainer -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data --restart=always portainer/portainer-ce:latest
```

### 初始設定

1. 開啟 `http://<你的NAS內網IP>:9000` 建立管理員帳號（新版密碼至少 12 個字元）。
2. 如果需要查看初始化相關訊息：

   ```bash
   sudo docker logs portainer 2>&1 | grep -i token
   ```

3. 進入 **Environments** → `local`，將 **Public IP** 設為自己 NAS 的內網 IP（`<你的NAS內網IP>`）。

### 更新

```bash
# 1. 停止並移除舊容器（資料存在 portainer_data，不會遺失）
docker stop portainer
docker rm portainer

# 2. 下載最新映像
docker pull portainer/portainer-ce

# 3. 重新啟動
sudo docker run -d --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -e TZ=Asia/Taipei \
  --restart=always \
  portainer/portainer-ce:latest
```

## Portainer 串接 K3s 叢集

Portainer 已安裝在 Docker 上，想要直接讀取並管理 K3s 時使用。

### 1. 在 K3s 伺服器安裝 Agent

```bash
kubectl apply -f https://downloads.portainer.io/ce2-39/portainer-agent-k8s-lb.yaml
```

確認狀態（STATUS 需為 `Running`）：

```bash
kubectl get pods -n portainer
```

### 2. 將服務改為 NodePort

避免出現連線被拒絕的問題：

```bash
kubectl patch svc portainer-agent -n portainer -p '{"spec":{"type":"NodePort","ports":[{"port":9001,"targetPort":9001,"nodePort":30778}]}}'
```

### 3. 查詢機器的實體 IPv4 位址

```bash
ip route get 1.1.1.1 | awk '{print $7}'
```

例如查到 `192.168.50.15`。

### 4. 在 Portainer 新增環境

1. 左側點選 **Environments** → 右上角 **Add environment**。
2. 選擇 **Kubernetes** → **Start wizard**。
3. 展開 **More options**，選擇 **Agent** 模式（不要用預設的 Edge Agent）。
4. **Name**：輸入自訂名稱，例如 `k3s-cluster`。
5. **Environment URL**：填入「實體 IP:30778」，例如 `192.168.50.15:30778`，**不要填 localhost**。
6. 點選 **Connect**。

### 5. 查看服務日誌

1. 進入剛綁定的 K3s 環境。
2. 點選左側 **Applications** 或 **Pods**。
3. 點進要查看的服務，在詳細頁面點選 **Logs**。

## 疑難排解

### Docker 異常時重新啟動

```bash
sudo systemctl restart docker
```

### 清理未使用的 Docker 資源

會刪除所有未被使用的映像、容器與網路，**執行前請確認沒有需要保留的項目**：

```bash
docker system prune -a
```

### Portainer 無法連線本機環境（Docker 29 以上）

**問題：** Docker Engine 升級到 v29.0.0 以上後，Portainer 出現以下錯誤：

```text
Failed loading environment
The environment named local is unreachable.
```

**原因：** Docker v29 將最低支援的 API 版本提高到 1.44，而目前的 Portainer 可能仍使用較舊的 API 版本，因此被拒絕連線（參考 GitHub Discussion #12926）。

**解法：** 透過 systemd 覆寫設定，讓 Docker 接受較舊的 API 版本（只調整 API 相容性，不需要降級 Docker）：

```bash
# 1. 建立設定檔目錄
sudo mkdir -p /etc/systemd/system/docker.service.d

# 2. 設定 DOCKER_MIN_API_VERSION 為 1.41（若仍有問題，可改為 1.24）
echo -e "[Service]\nEnvironment=DOCKER_MIN_API_VERSION=1.41" | sudo tee /etc/systemd/system/docker.service.d/override.conf

# 3. 重新載入設定
sudo systemctl daemon-reload

# 4. 重新啟動 Docker
sudo systemctl restart docker
```

### 查詢 OMV 無故關機原因

查看上一次開機的系統日誌：

```bash
sudo journalctl -b -1 -e
```
