# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置
| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | Share with my Mac (NAT) | 172.16.109.140 | 對外上網、跳板機 |
| dev-a | NIC 2 | Private to my Mac (Host-only) | 172.16.49.132 | 內部通訊 |
| server-b | NIC 1 | Private to my Mac (Host-only) | 172.16.49.133 | 隔離伺服器 |

## 1. 基礎連線驗證

### 系統身份確認
![hostname-a](hostnamectl-a.jpg)
![hostname-b](hostnamectl-b.jpg)

### 網卡 IP 配置
![dev-a-ip](dev-a-ip.jpg)
![dev-b-ip](dev-b-ip.jpg)

### 連線測試 (NAT & Ping)
* **dev-a 連外網測試**：
![dev-a-nat-ping](dev-a-nat-ping.jpg)

* **雙向互 Ping 測試**：
![ping-a-to-b](ping-a-to-b.jpg)
![ping-b-to-a](ping-b-to-a.jpg)

---

## 2. SSH 服務與遠端操作驗證

### SSH 狀態與監聽
![server-b-ssh-status](server-b-ssh-status.jpg)
![server-b-ssh-listen](server-b-ssh-listen.jpg)

### 遠端登入與指令執行
![ssh-login-test](ssh-login-test.jpg)
![ssh-remote-cmd](ssh-remote-cmd.jpg)

### SCP 檔案傳輸驗證
![scp-transfer-test](scp-transfer-test.jpg)

### 隔離驗證 (server-b 確實連不上網)
![server-b-isolated](server-b-isolated.jpg)

---

## 3. 故障演練 (Troubleshooting)

### 故障前基線紀錄
![server-b 故障前](server-b%20故障前.jpg)
![dev-a 到 server-b 故障前](dev-a%20%E5%88%B0%20server-b%20%E6%95%85%E9%9A%9C%E5%89%8D.jpg)

### 故障演練一：L2 介面停用 (Interface Down)
* **故障注入**：`sudo ip link set ens160 down`
![b-interface-down](b-interface-down.jpg)

* **觀察失敗**：`No route to host`
![a-observe-failure-L2](a-observe-failure-L2.jpg)

* **回復驗證**：
![b-interface-recovery](b-interface-recovery.jpg)
![a-recovery-check](a-recovery-check.jpg)

### 故障演練二：L4 服務停止 (SSH Socket Stop)
* **故障注入**：徹底關閉 SSH Socket
![b-ssh-fully-stopped](b-ssh-fully-stopped.jpg)

* **觀察失敗**：`Connection refused` (注意：此時 Ping 是通的)
![a-observe-failure-L4](a-observe-failure-L4.jpg)

* **回復驗證**：
![回復 SSH 服務](%E5%9B%9E%E5%BE%A9%20SSH%20%E6%9C%8D%E5%8B%99.jpg)
![a-recovery-check-L4](a-recovery-check-L4.jpg)

---

## 4. 網路拓樸圖
![network-diagram](network-diagram.jpg)
