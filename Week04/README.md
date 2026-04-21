# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統層級設定檔目錄，存放各服務的設定檔 | 存放 Docker daemon 設定檔（如 `daemon.json`），控制 daemon 行為 |
| /var/lib/docker/ | 應用程式的持久化狀態資料目錄 | Docker 的資料根目錄，存放所有映像層、容器資料、volumes、網路設定等 |
| /usr/bin/docker | 使用者可執行的二進位程式目錄 | Docker CLI 執行檔，使用者下的所有 `docker` 指令都是呼叫這支程式 |
| /run/docker.sock | 執行期間的暫存資料與 socket 檔案目錄 | Docker daemon 的 Unix socket，CLI 與 daemon 之間的通訊管道 |

### 瀏覽 FHS 與 Docker 資料目錄

![步驟 1-3 瀏覽 FHS 與 Docker 資料目錄](%E6%AD%A5%E9%A9%9F%201-3%EF%BC%9A%E7%80%8F%E8%A6%BD%20FHS%20%E8%88%87%20Docker%20%E8%B3%87%E6%96%99%E7%9B%AE%E9%8C%84.png)

### 檢視各路徑權限

![檢視其他 Docker 檔案的權限](%E6%AA%A2%E8%A6%96%E5%85%B6%E4%BB%96%20Docker%20%E6%AA%94%E6%A1%88%E7%9A%84%E6%AC%8A%E9%99%90.png)

---

## Docker 系統資訊

- Storage Driver：`overlay2`
- Backing Filesystem：`extfs`
- Docker Root Dir：`/var/lib/docker`
- 作業系統：Ubuntu 25.10，架構：aarch64，CPUs：2，記憶體：3.304 GiB
- 拉取映像前 `/var/lib/docker/` 大小：**243M**
- 拉取映像後 `/var/lib/docker/` 大小：**429M**（拉取 nginx:latest 後增加約 186M）

### docker info 觀察系統資訊

![用 docker info 觀察系統資訊](%E7%94%A8%20docker%20info%20%E8%A7%80%E5%AF%9F%E7%B3%BB%E7%B5%B1%E8%B3%87%E8%A8%8A.png)

![用 docker info 觀察系統資訊-2](%E7%94%A8%20docker%20info%20%E8%A7%80%E5%AF%9F%E7%B3%BB%E7%B5%B1%E8%B3%87%E8%A8%8A-2.png)

### 拉取映像前後對比磁碟使用量

![拉取映像前後對比磁碟使用量](%E6%8B%89%E5%8F%96%E6%98%A0%E5%83%8F%E5%89%8D%E5%BE%8C%E5%B0%8D%E6%AF%94%E7%A3%81%E7%A2%9F%E4%BD%BF%E7%94%A8%E9%87%8F.png)

---

## 權限結構

### Docker Socket 權限解讀

```
srw-rw---- 1 root docker 0 Apr 21 13:34 /var/run/docker.sock
```

| 欄位 | 值 | 說明 |
|---|---|---|
| 檔案類型 | `s` | Unix domain socket，不是一般檔案 |
| owner 權限 | `rw-` | root 可讀寫 |
| group 權限 | `rw-` | docker 群組成員可讀寫 |
| others 權限 | `---` | 其他使用者完全無法存取 |
| owner | root | 由 Docker daemon 以 root 身份建立 |
| group | docker | 屬於 docker 群組，群組成員免 sudo 即可操作 |

![解讀 Docker Socket 權限](%E8%A7%A3%E8%AE%80%20Docker%20Socket%20%E6%AC%8A%E9%99%90.png)

### 使用者群組

加入 docker group 前，`id` 輸出：

```
uid=1000(debby) gid=1000(debby) groups=1000(debby),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),116(lpadmin)
```

尚未包含 docker 群組，因此 `docker ps` 會出現 permission denied。

![檢查當前使用者的群組](%E6%AA%A2%E6%9F%A5%E7%95%B6%E5%89%8D%E4%BD%BF%E7%94%A8%E8%80%85%E7%9A%84%E7%BE%A4%E7%B5%84.png)

### 加入 docker group

![加入 docker group](%E5%8A%A0%E5%85%A5%20docker%20group%20.png)

![加入 docker group 並驗證](%E5%8A%A0%E5%85%A5%20docker%20group%20%E4%B8%A6%E9%A9%97%E8%AD%89.png)

### 未加入 docker group 時的錯誤

![未加入 docker group 時觀察錯誤](%E6%9C%AA%E5%8A%A0%E5%85%A5%20docker%20group%20%E6%99%82%E8%A7%80%E5%AF%9F%E9%8C%AF%E8%AA%A4.png)

### 安全意涵

docker group ≈ root 的原因：Docker socket（`/var/run/docker.sock`）的 group 是 docker，group 權限是 `rw-`，代表 docker group 成員可以直接對 socket 讀寫，等同於可以叫 Docker daemon（以 root 身份執行）做任何事。

**安全示範**：docker group 的使用者可以執行 `docker run --rm -v /etc/shadow:/host-shadow:ro alpine cat /host-shadow`，直接在容器內讀取 host 的 `/etc/shadow`（密碼 hash 檔），這是一個 root 才能存取的檔案。這代表任何在 docker group 的使用者，都可以用 volume mount 的方式繞過 Linux 檔案權限，讀取甚至修改任何 host 檔案，實質上等同於 root 權限。

![安全示範 docker group 用戶的權力](%E5%AE%89%E5%85%A8%E7%A4%BA%E7%AF%84%E2%80%94%E2%80%94docker%20group%20%E7%94%A8%E6%88%B6%E7%9A%84%E6%AC%8A%E5%8A%9B.png)

---

## 程序與服務管理

### systemctl status docker

```
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
   Active: active (running) since Tue 2026-04-21 13:34:30 CST; 1h 38min ago
   Main PID: 1623 (dockerd)
   Tasks: 11
   Memory: 175M
   TriggeredBy: ● docker.socket
```

![用 systemctl 完整解讀 Docker 服務狀態](%E7%94%A8%20systemctl%20%E5%AE%8C%E6%95%B4%E8%A7%A3%E8%AE%80%20Docker%20%E6%9C%8D%E5%8B%99%E7%8B%80%E6%85%8B.png)

![用 systemctl 管理 Docker 生命週期](%E7%94%A8%20systemctl%20%E7%AE%A1%E7%90%86%20Docker%20%E7%94%9F%E5%91%BD%E9%80%B1%E6%9C%9F.png)

### 確認 Docker 開機自動啟動 & 查看 daemon 日誌

![確認 Docker 開機自動啟動 查看 Docker daemon 日誌](%E7%A2%BA%E8%AA%8D%20Docker%20%E9%96%8B%E6%A9%9F%E8%87%AA%E5%8B%95%E5%95%9F%E5%8B%95%3A%E6%9F%A5%E7%9C%8B%20Docker%20daemon%20%E6%97%A5%E8%AA%8C.png)

### journalctl 日誌分析

從 `journalctl -u docker --since "1 hour ago"` 可觀察到：
- `13:34:30`：Docker daemon 啟動，開始監聽 fd:// 及 containerd socket
- `15:08:25`：有容器建立事件（拉取映像、啟動容器）
- `15:10:35`：容器停止/清理事件
- daemon 由 `docker.socket` 觸發啟動（socket activation 機制）

### CLI vs Daemon 差異

**Docker CLI**（`/usr/bin/docker`）是一個獨立的執行檔，它只是一個「命令工具」，負責把使用者的指令翻譯成 API 請求，然後透過 `/var/run/docker.sock` 傳給 Docker daemon。

**Docker Daemon**（`dockerd`）才是真正執行容器的服務，負責管理映像、容器、網路、volumes 等所有資源，以 root 身份在背景持續運行。

因此，`docker --version` 成功只是代表 CLI 執行檔存在且可執行，跟 daemon 是否在跑完全無關。只要 daemon 停止，`docker ps`、`docker run` 等所有需要與 daemon 溝通的指令都會失敗，出現 `Cannot connect to the Docker daemon` 的錯誤。

### 觀察 Docker daemon process

![觀察 Docker daemon process](%E8%A7%80%E5%AF%9F%20Docker%20daemon%20process.png)

### 跑容器並觀察 process

![跑容器並觀察 process](%E8%B7%91%E5%AE%B9%E5%99%A8%E4%B8%A6%E8%A7%80%E5%AF%9F%20process.png)

### 用 top 查看系統資源使用

![用 top 查看系統資源使用](%E7%94%A8%20top%20%E6%9F%A5%E7%9C%8B%E7%B3%BB%E7%B5%B1%E8%B3%87%E6%BA%90%E4%BD%BF%E7%94%A8.png)

### 驗證 hello-world 不需 sudo

![驗證 hello-world 不需 sudo](%E9%A9%97%E8%AD%89%20hello-world%20%E4%B8%8D%E9%9C%80%20sudo.png)

---

## 環境變數

- `$PATH`：`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin`
- `which docker`：`/usr/bin/docker`
- 容器內外環境變數差異觀察：
  - Host：`USER=debby`、`HOME=/home/debby`、`SHELL=/bin/bash`，PATH 包含 snap/bin 等 Ubuntu 特有路徑
  - Container（alpine）：無 `USER`、`SHELL` 變數，`HOME=/root`（容器預設為 root），`HOSTNAME` 為容器 ID（`42cc66804c98`），PATH 大幅精簡，沒有 snap、games 等路徑
  - **關鍵差異**：容器有自己完全隔離的環境變數，不會繼承 Host 的使用者身份與路徑設定

### 查看身份變數

![查看身份變數](%E6%9F%A5%E7%9C%8B%E8%BA%AB%E4%BB%BD%E8%AE%8A%E6%95%B8.png)

### 定位 Docker CLI 路徑 & 檢視 $PATH 搜尋順序

![定位 Docker CLI 路徑 檢視 $PATH 搜尋順序](%E5%AE%9A%E4%BD%8D%20Docker%20CLI%20%E8%B7%AF%E5%BE%91%3A%E6%AA%A2%E8%A6%96%20%24PATH%20%E6%90%9C%E5%B0%8B%E9%A0%86%E5%BA%8F.png)

### 定位 Docker CLI 執行檔

![定位 Docker CLI 執行檔](%E5%AE%9A%E4%BD%8D%20Docker%20CLI%20%E5%9F%B7%E8%A1%8C%E6%AA%94.png)

### 對比容器內外的環境變數

![對比容器內外的環境變數](%E5%B0%8D%E6%AF%94%E5%AE%B9%E5%99%A8%E5%85%A7%E5%A4%96%E7%9A%84%E7%92%B0%E5%A2%83%E8%AE%8A%E6%95%B8.png)

---

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active (running) | inactive (dead)，停止於 15:23:01 | active (running)，重啟於 15:24:30 |
| docker --version | 正常（Docker version 29.3.0） | 正常（CLI 獨立執行檔，不受 daemon 影響） | 正常 |
| docker ps | 正常（空列表） | `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | 正常（空列表） |
| ps aux \| grep dockerd | 有 process（PID 1623） | 無 dockerd process | 有 process（新 PID） |

### 記錄故障前基線

![記錄故障前基線](%E8%A8%98%E9%8C%84%E6%95%85%E9%9A%9C%E5%89%8D%E5%9F%BA%E7%B7%9A.png)

### 故障注入與觀測

![故障注入 觀測故障](%E6%95%85%E9%9A%9C%E6%B3%A8%E5%85%A5%3A%E8%A7%80%E6%B8%AC%E6%95%85%E9%9A%9C.png)

### 回復 Docker Daemon 驗證

![回復 Docker daemon 驗證](%E5%9B%9E%E5%BE%A9%20Docker%20daemon%3A%E9%A9%97%E8%AD%89.png)

---

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | `srw-rw----`（0660，root:docker） | `srw-------`（0600，只有 root 可存取） | `srw-rw----`（0660，root:docker） |
| docker ps（不加 sudo） | 正常 | `permission denied while trying to connect to the Docker API` | 正常 |
| sudo docker ps | 正常 | 正常（root 仍是 owner，有讀寫權限） | 正常 |
| systemctl status docker | active | active（daemon 不受 socket 權限影響） | active |

### 破壞 Socket 權限故障注入

![破壞 Docker Socket 權限-故障注入](%E7%A0%B4%E5%A3%9E%20Docker%20Socket%20%E6%AC%8A%E9%99%90-%E6%95%85%E9%9A%9C%E6%B3%A8%E5%85%A5.png)

### 回復 Socket 權限驗證

![回復 socket 權限 驗證](%E5%9B%9E%E5%BE%A9%20socket%20%E6%AC%8A%E9%99%90%3A%E9%A9%97%E8%AD%89.png)

---

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | Docker daemon（dockerd）沒有在執行，socket 不存在或無法連線 | 檢查 `systemctl status docker`；用 `ps aux \| grep dockerd` 確認 process 是否存在 |
| permission denied…docker.sock | Daemon 有在跑，但當前使用者對 socket 沒有讀寫權限 | 檢查 `ls -la /var/run/docker.sock` 確認權限；檢查 `groups` 確認是否在 docker group；改用 `sudo docker` 驗證 |

**差異說明**：`Cannot connect` 代表根本連不上 socket（daemon 沒跑），屬於服務層問題，要先修服務；`permission denied` 代表 socket 存在、daemon 有跑，但使用者沒有存取權限，屬於檔案權限問題，要修權限或群組。

---

## 排錯紀錄

- **症狀**：執行 `docker ps` 出現 `permission denied while trying to connect to the Docker API at unix:///var/run/docker.sock`，但 `sudo docker ps` 正常。
- **診斷**：先確認 daemon 狀態（`systemctl status docker` 顯示 active）；再確認 socket 權限（`ls -la /var/run/docker.sock` 顯示 `srw-------`，group 的 rw 被拿掉）；再確認使用者群組（`groups` 顯示包含 docker），判斷是 socket 權限被 `chmod 600` 破壞，而非 daemon 問題。
- **修正**：執行 `sudo chmod 660 /var/run/docker.sock` 恢復 group 讀寫權限；執行 `sudo chown root:docker /var/run/docker.sock` 確認 group owner 正確。
- **驗證**：`ls -la /var/run/docker.sock` 確認恢復為 `srw-rw----`；執行 `docker ps`（不加 sudo）成功回傳空列表，確認修正有效。

---

## 設計決策

**為什麼教學環境用 `usermod -aG docker $USER` 加 group 而不是每次都 sudo？這個選擇的風險是什麼？**

在教學環境中，用 `usermod -aG docker $USER` 讓使用者加入 docker group，目的是讓操作更流暢方便，不用每次都輸入密碼，降低練習的摩擦力。

但這個選擇有明確的安全風險：如同本週安全示範所展示，docker group 的成員等同於 root，可以透過 volume mount 存取 host 上的任何檔案（包括 `/etc/shadow`、`/etc/passwd` 等敏感系統檔案），甚至可以掛載整個 `/`，完全掌控 host 系統。這個權限完全繞過了 Linux 的一般使用者權限模型。

在生產環境中，應該避免將一般使用者加入 docker group，改用 `sudo` 控制存取，或使用 Rootless Docker（讓 daemon 以非 root 身份執行）來降低風險。教學環境之所以接受這個取捨，是因為在受控的 VM 環境中，安全風險可控，方便性的優先級較高。
