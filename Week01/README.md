# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- **Host OS**: macOS (M4 Mac)
- **VM 名稱**: vct-w01-412630708
- **Ubuntu 版本**: Ubuntu 25.10 (Oracular Oriole)
- **Docker 版本**: Docker version 29.3.0
- **Docker Compose 版本**: Docker Compose version v5.1.0

## VM 資源配置驗證
| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | \`lscpu | grep "^CPU(s)"\` | CPU(s): 2 |
| 記憶體 | 4 GB | \`free -h | grep Mem\` | Mem: 3.3Gi |
| 磁碟 | 20 GB | \`df -h / | grep "/$"\` | /dev/nvme0n1p2 19G |
| Hypervisor | VMware | \`lscpu | grep Hypervisor\` | Hypervisor vendor: VMware |

## 四層驗收證據
- [x] ① Repository: \`cat /etc/apt/sources.list.d/docker.list\` 配置成功
- [x] ② Engine: \`dpkg -l | grep docker-ce\` (5:29.3.0) 已安裝
- [x] ③ Daemon: \`sudo systemctl status docker\` 顯示 Active (running)
- [x] ④ 端到端: \`sudo docker run hello-world\` 成功輸出 Hello from Docker!
- [x] Compose: \`v5.1.0\` 驗證可執行

![四層驗收](./復原後完整驗證.png)

## 容器操作紀錄
- [x] **nginx**: \`sudo docker run -d -p 8080:80 nginx\` + \`curl localhost:8080\` 成功。
- [x] **alpine**: 進入 \`/sh\` 環境，確認 OS 版本為 Linux (Alpine)。
- [x] **映像列表**: \`sudo docker images\` 包含 alpine, nginx, hello-world。

![驗證nginx功能完整](./驗證nginx功能完整.png)

## Snapshot 清單
| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | 21:28 | 原始乾淨基線，僅安裝 Docker 引擎 | hostnamectl、docker --version |
| docker-ready | 21:32 | 開發就緒狀態，包含預拉取的映像檔 | sudo docker images、hello-world |

### Snapshot 結構與磁碟觀察
![差異磁碟機制](./差異磁碟機制.png)
![快照復原](./快照復原.png)

## 故障演練三階段對照
| 項目 | 故障前 (Baseline) | 故障中 (Broken) | 回復後 (Restored) |
|---|---|---|---|
| **docker.list 存在** | 是 | **否 (.broken)** | **是** |
| **apt-cache policy** | 有版本 | **顯示 none** | **有版本** |
| **hello-world 成功** | 是 | N/A | **是** |
| **nginx curl 成功** | 是 | N/A | **是** |

### 故障注入與回復證據紀錄
- **故障觀測**：顯示軟體源失效，無法獲取更新
![故障證據](./故障證據.png)
- **演練過程**：再次注入並執行回復流程驗證
![再次注入故障](./再次注入故障.png)

## 手動修復 vs Snapshot 回復比較
| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 約 1 分鐘 (需確診並輸入 mv) | 約 30 秒 (自動還原) |
| 適用情境 | 已知特定設定檔錯誤時 | 系統發生不明原因大規模損壞時 |
| 風險 | 可能產生人為二次輸入錯誤 | 強制回到穩定歷史狀態，風險極低 |

## Snapshot 保留策略
- **新增條件**：安裝新服務前、大改軟體源設定前、或完成階段性穩定測試後。
- **保留上限**：最多 3 個活躍 snapshot，防止 delta vmdk 過大影響效能。
- **刪除條件**：新節點確認穩定後，刪除最舊的備份節點。

## 最小可重現命令鏈
\`\`\`bash
# 故障注入：移走 Repository 設定
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken && sudo apt update
# 驗證故障：Candidate 變為 none
apt-cache policy docker-ce | head -5
# 回復驗證：
ls /etc/apt/sources.list.d/docker.list && sudo docker run --rm hello-world
\`\`\`

## 排錯紀錄 (Troubleshooting)
- **症狀**：`curl http://localhost:8080` 出現 `Connection reset by peer`。
- **診斷**：使用 `docker ps` 確認容器已啟動，判斷為服務初始化延遲。
- **修正**：在指令鏈中加入 `sleep 5`。
- **驗證**：等待 5 秒後成功取得 Nginx 歡迎頁面。

## 設計決策
在 M4 Mac 環境下，選擇 **VMware Fusion (Type 2 Hypervisor)** 搭配 Ubuntu 25.10 的方案。主因是需要利用其 **Snapshot (差異磁碟)** 機制，讓我們在進行系統級故障演練時，擁有「後悔藥」功能，而不必重新安裝整個作業系統。
