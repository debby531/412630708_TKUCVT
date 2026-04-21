# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 172.16.109.146 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 172.16.49.132 | — |
| app | 應用層 | NIC 1 | Host-only | 172.16.49.133 | SSH from 172.16.49.0/24 |
| db | 資料層 | NIC 1 | Host-only | 172.16.49.134 | SSH from app (172.16.49.133) + bastion (172.16.49.132) |

## SSH 金鑰認證

- 金鑰類型：ed25519
- 公鑰部署到：app 和 db 的 `~/.ssh/authorized_keys`（透過 `ssh-copy-id` 各部署 1 把金鑰）
- 免密碼登入驗證：
  - bastion → app：
    ```
    $ ssh debby@172.16.49.133 "echo '金鑰認證成功'"
    金鑰認證成功
    ```
  - bastion → db：
    ```
    $ ssh debby@172.16.49.134 "echo '金鑰認證成功'"
    金鑰認證成功
    ```

## 防火牆規則

### app 的 ufw status

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To          Action     From
--          ------     ----
22/tcp      ALLOW IN   192.168.56.0/24
```

### db 的 ufw status

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To          Action     From
--          ------     ----
22/tcp      ALLOW IN   172.16.49.133
22/tcp      ALLOW IN   172.16.49.132
```

### 防火牆確實在擋的證據

從 bastion 對 app 的 8080 port 執行 curl，防火牆封鎖連線，連線逾時：

```
debby@dev-a:~$ curl -m 5 http://172.16.49.133:8080 2>&1
curl: (28) Connection timed out after 5001 milliseconds
```

## ProxyJump 跳板連線

- 指令（`~/.ssh/config` 設定）：

```
Host bastion
    HostName 172.16.49.132
    User debby

Host app
    HostName 172.16.49.133
    User debby
    ProxyJump bastion

Host db
    HostName 172.16.49.134
    User debby
    ProxyJump bastion
```

- 驗證輸出：

```
# 一跳：從 Host 連到 bastion
$ ssh bastion "hostname"
bastion

# 兩跳：透過 bastion 跳板連到 db
$ ssh bastion "ssh debby@172.16.49.134 hostname"
db
```

- SCP 傳檔驗證：

```
# 1. 在 Mac 上建立測試用文字檔
echo "Test file via ProxyJump" > /tmp/proxy-test.txt

# 2. 靠跳板傳送到 db（自動使用 ProxyJump）
scp /tmp/proxy-test.txt db:/tmp/
proxy-test.txt    100%    24    36.5KB/s    00:00

# 3. 驗證檔案是否真的傳到了 db
ssh db "cat /tmp/proxy-test.txt"
Test file via ProxyJump
```

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules（允許 22/tcp from bastion） | deny all（incoming + outgoing 全封） | active + rules（重設後允許 22/tcp from 172.16.49.132） |
| bastion ping app | 成功（0% packet loss） | 成功（ICMP 不受 ufw TCP 規則封鎖，ping 仍通） | 成功（0% packet loss） |
| bastion SSH app | 成功 | **timed out**（`ssh: connect to host 172.16.49.133 port 22: Connection timed out`） | 成功（hostname 回傳 `app`） |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp \| grep :22 | 有監聽（`0.0.0.0:22` 及 `[::]:22`） | 無監聽（`sudo systemctl stop ssh.service ssh.socket` 後無輸出） | 有監聽（`sudo systemctl start ssh` 後恢復 `0.0.0.0:22` 及 `[::]:22`） |
| bastion ping app | 成功（0% packet loss） | 成功（0% packet loss，網路層 L3 正常，SSH 停止不影響 ICMP） | 成功（0% packet loss） |
| bastion SSH app | 成功 | **refused**（`ssh: connect to host 172.16.49.133 port 22: Connection refused`） | 成功（`ssh debby@172.16.49.133 "hostname"` → `app`） |

## timeout vs refused 差異

**Connection timed out（連線逾時）**：
封包送出去之後完全沒有任何回應，TCP 握手沒有完成也沒有收到 RST，客戶端只能等到超時才放棄。這代表封包被「丟棄」了，通常是防火牆（如 ufw `deny`）或網路路由問題造成的。排錯方向：先檢查防火牆規則（`sudo ufw status`）、確認網路路由是否正常（`ping`）。

**Connection refused（連線拒絕）**：
封包送達目標主機之後，主機立刻回傳 TCP RST（重設），表示該 port 沒有任何程式在監聽。這代表網路層是通的，但服務層（SSH daemon）沒有啟動。排錯方向：確認服務是否在跑（`ss -tlnp | grep :22`）、確認服務狀態（`sudo systemctl status ssh`）。

**總結**：`timed out` 指向防火牆／網路層問題；`refused` 指向服務未啟動問題。兩者差異的關鍵在於對方有沒有「回應」，有回應（即使是拒絕）代表網路通，沒回應才是被牆擋住。

## 網路拓樸圖

![network-diagram](./拓樸圖.png)

```
Internet / Host
     │
     │ NAT（172.16.109.146）
     ▼
┌─────────────┐
│   bastion   │──── Host-only（172.16.49.132）────┐
└─────────────┘                                   │
                                       ┌───────────┴───────────┐
                             ┌─────────▼──────────┐  ┌─────────▼──────────┐
                             │        app          │  │         db          │
                             │   172.16.49.133     │  │   172.16.49.134     │
                             └────────────────────┘  └────────────────────┘
                              SSH: from bastion         SSH: from app + bastion
```

## 排錯紀錄

- 症狀：從 Host 使用 `ssh -J debby@172.16.49.132 debby@172.16.49.133 "hostname"` 出現 `zsh: unknown file attribute: J` 及 `Permission denied (publickey)` 錯誤，無法透過跳板連線到 app。
- 診斷：首先確認 bastion 本身可以連通（`ssh bastion "hostname"` 成功回傳 `bastion`）；接著確認 bastion 到 db 的連線（`ssh bastion "ssh debby@172.16.49.134 hostname"` 回傳 `db`），判斷問題出在 Host 端的 `-J` 語法解析失敗，以及 app 的公鑰尚未正確部署。
- 修正：改用 `~/.ssh/config` 設定 `ProxyJump bastion`，替代在命令列直接使用 `-J` 參數；同時確認公鑰已透過 `ssh-copy-id` 部署到 app 的 `~/.ssh/authorized_keys`。
- 驗證：執行 `ssh app "hostname"` 成功回傳 `app`，`ssh db "hostname"` 成功回傳 `db`，無需輸入密碼，確認 ProxyJump 與金鑰認證均正常運作。

## 設計決策

**為什麼 db 允許 bastion 直連，而不是只允許從 app 跳？**

本週的設計讓 db 同時允許來自 app（172.16.49.133）與 bastion（172.16.49.132）的 SSH 連線，而非只開放 app。

從純粹安全角度來看，只允許 app → db 是更嚴格的設計，可以確保 db 只被應用層存取，符合最小暴露原則。但在實務維運上，若 app 發生故障（服務掛掉或 SSH 停止），管理員就完全無法直接登入 db 進行排查或緊急修復，必須先修好 app 才能繼續，增加了維運風險。

因此選擇同時開放 bastion 直連 db，是在「安全性」與「可維運性」之間的取捨：犧牲一點理論上的最嚴格隔離，換取管理員在緊急情況下仍能透過受信任的跳板機直接進入 db 排錯的能力。這也是實務上常見的設計——bastion 本身已是受控的進入點，透過 bastion 直連 db 的風險是可以接受的。
