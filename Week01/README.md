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
| CPU | 2 vCPU | `lscpu | grep "^CPU(s)"` | CPU(s): 2 |
| 記憶體 | 4 GB | `free -h | grep Mem` | Mem: 3.3Gi |
| 磁碟 | 20 GB | `df -h / | grep "/$"` | /dev/nvme0n1p2 19G |
| Hypervisor | VMware | `lscpu | grep Hypervisor` | Hypervisor vendor: VMware |

## 四層驗收證據
- [x] ① Repository: \`cat /etc/apt/sources.list.d/docker.list\` 配置成功
- [x] ② Engine: \`dpkg -l | grep docker-ce\` (5:29.3.0) 已安裝
- [x] ③ Daemon: \`sudo systemctl status docker\` 顯示 Active (running)
- [x] ④ 端到端: \`sudo docker run hello-world\` 成功輸出 Hello from Docker!
- [x] Compose: \`v5.1.0\` 驗證可執行

![四層驗收](./復原後完整驗證.png)

## 容器操作紀錄
- [x] **nginx**: \`sudo docker run -d -p 8080:80 nginx\` + \`curl localhost:8080\` 成功回傳 HTML。
- [x] **alpine**: 進入 \`/sh\` 環境並確認 OS 版本。
- [x] **映像列表**: \`sudo docker images\` 包含 alpine, nginx, hello-world。

![驗證nginx功能完整](./驗證nginx功能完整.png)

## Snapshot 清單
| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | 21:28 | 原始乾淨基線，僅安裝 Docker 引擎 | hostnamectl、docker --version |
| docker-ready | 21:32 | 開發就緒狀態，包含基礎映像檔 | sudo docker images、hello-world |

### Snapshot 結構與磁碟觀察
![差異磁碟機制](./差異磁碟機制.png)
![快照復原](./快照復原.png)

## 故障演練三階段對照
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 (.broken) | 是 |
| apt-cache policy | 有版本 | none | 有版本 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

### 故障注入與回復證據
- **故障證據**：顯示軟體源失效
![故障證據](./故障證據.png)
- **再次注入**：確保回復流程正確
![再次注入故障](./再次注入故障.png)

## 手動修復 vs Snapshot 回復
| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 約 1 分鐘 | 約 30 秒 |
| 適用情境 | 確切知道哪個設定檔被改動時 | 系統發生原因不明的毀滅性損壞時 |
| 風險 | 可能產生人為二次輸入錯誤 | 強制回到歷史健康時間點 |

## Snapshot 保留策略
- **新增條件**：每次安裝新工具或進行重大系統設定變更前，且確保當前環境已通過功能驗證。
- **保留上限**：最多保留 3 個活躍 snapshot。
- **刪除條件**：新進度節點已驗證穩定，且舊節點確認不再需要回溯時。

## 最小可重現命令鏈
\`\`\`bash
# 故障注入
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken && sudo apt update
# 驗證故障
apt-cache policy docker-ce | head -5
# 回復驗證 (Revert Snapshot 後執行)
ls /etc/apt/sources.list.d/docker.list && sudo docker run --rm hello-world
\`\`\`

## 排錯紀錄
- **症狀**：執行 \`curl http://localhost:8080\` 出現 \`Connection reset by peer\`。
- **診斷**：確認容器啟動中，判斷為服務初始化延遲。
- **修正**：在指令中加入 \`sleep 5\`。
- **驗證**：成功取得 Nginx 歡迎頁面。

## 設計決策
選擇 VMware Type 2 Hypervisor 方案，主因是其 Snapshot 機制提供強大的「環境沙盤」功能，適合進行破壞性故障演練而無須擔心影響 Host 端主機安全。
