# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- **Host OS**: macOS (M4 Mac)
- **VM 名稱**: vct-w01-412630708
- **Ubuntu 版本**: 25.10 (Questing)
- **Docker**: 29.3.0 / **Compose**: v5.1.0

## VM 資源配置驗證
| 項目 | VMware 設定值 | VM 內輸出 |
|---|---|---|
| CPU / RAM | 2 vCPU / 4 GB | CPU(s): 2 / Mem: 3.3Gi |
| Disk / Hyper | 20 GB / VMware | /dev/nvme0n1p2 19G / Vendor: VMware |

## 四層驗收證據
- [x] ① **Repository**: `docker.list` 已配置 `questing stable`。
- [x] ② **Engine**: `docker-ce` (5:29.3.0) 狀態為 `ii` (已安裝)。
- [x] ③ **Daemon**: `docker.service` 狀態為 **active (running)**。
- [x] ④ **端到端**: `hello-world` 執行成功，輸出 **Hello from Docker!**。
- [x] **Compose**: `docker compose version` 輸出 `v5.1.0`。

![四層驗收](./復原後完整驗證.png)

## 容器操作紀錄
- [x] **nginx**: `curl localhost:8080` 成功取得 `Welcome to nginx!` HTML 內容。
- [x] **alpine**: 進入 `/bin/sh` 並執行 `ls`，確認內部檔案系統結構完整。
- [x] **映像列表**: 包含 `alpine`, `nginx`, `hello-world`, `httpd` 等基礎映像。

![驗證nginx功能完整](./驗證nginx功能完整.png)

## Snapshot 清單
| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | 21:28 | 原始乾淨基線，僅安裝 Docker 引擎 | docker --version |
| docker-ready | 21:32 | 開發就緒狀態，包含基礎映像檔 | sudo docker images |

## 故障演練對照
| 項目 | 故障前 | 故障中 (注入) | 回復後 (Snapshot) |
|---|---|---|---|
| docker.list | 存在 | 檔案更名為 `.broken` | 恢復正常 |
| 系統狀態 | 健康 | 軟體源失效 | 驗證通過 |

## 最小可重現命令鏈
# 故障注入
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken && sudo apt update
# 回復驗證
ls /etc/apt/sources.list.d/docker.list && sudo docker run --rm hello-world


## 排錯紀錄
- 症狀：curl 出現 Connection reset by peer。
- 診斷：容器啟動後的服務初始化延遲。
- 修正：在指令鏈中加入 sleep 5 確保服務就緒。


## 設計決策
選擇 VMware Type 2 Hypervisor 方案，主因是其 Snapshot 機制提供強大的「環境沙盤」功能，適合進行破壞性故障演練而無須擔心影響 Host 端主機安全。
