# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置
| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 172.16.109.140 | 對外上網、跳板機 |
| dev-a | NIC 2 | Host-only | 172.16.49.132 | 內部通訊 |
| server-b | NIC 1 | Host-only | 172.16.49.133 | 隔離伺服器 |

## 連線驗證紀錄
- [x] **dev-a NAT 可上網**：執行 `ping -c 4 google.com` 成功。
- [x] **雙向互 ping 成功**：`172.16.49.132` 與 `172.16.49.133` 互相通訊正常。
- [x] **SSH 連線成功**：從 dev-a 執行 `ssh debby@172.16.49.133` 成功登入。
- [x] **SCP 傳檔成功**：成功傳輸測試檔並驗證內容。
- [x] **server-b 不能上網**：驗證 Host-only 隔離有效。

## 故障演練一：介面停用 (L2 層故障)
- **故障注入**：`sudo ip link set ens160 down`
- **觀測結果**：dev-a ping 不通，SSH 顯示 `No route to host`。
- **回復驗證**：重新 `up` 介面後連線恢復。

## 故障演練二：SSH 服務停止 (L4 層故障)
- **故障注入**：`sudo systemctl stop ssh.socket`
- **觀測結果**：L3 (Ping) 正常通訊，但 L4 (SSH) 顯示 `Connection refused`。
- **診斷關鍵**：單純關閉 ssh.service 可能因 socket 觸發而失效，須關閉 socket 才能徹底模擬故障。

## 網路拓樸圖
![網路拓樸圖](network-diagram.png)
