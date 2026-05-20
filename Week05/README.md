# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

![抓取 Docker 驅動資訊](pic/抓取%20Docker%20驅動資訊.png)

- Storage Driver：overlay2
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：runc

補充：Docker version 29.3.0, build 5927d80；Runtimes: io.containerd.runc.v2 runc。

cgroup v2 掛載點確認：

![確認 cgroup v2 掛載點](pic/確認%20cgroup%20v2%20掛載點.png)

`cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)` — 所有控制器掛在同一棵樹下，路徑統一為 `/sys/fs/cgroup/`，能看到 `cpu.max`、`memory.max` 這類 v2 統一檔名。

---

## Namespace 觀察

### 六種 namespace 用途（用自己的話）

- **PID**：把 process ID 空間切成獨立隔間。容器內看到的 PID 1 是自己跑的程式，host 看到的是一個普通的大數字 PID。同一支 process，兩個視角。
- **NET**：每個容器有自己的網路協定堆疊——獨立的 lo、eth0、路由表、iptables。host 那一堆 docker0、veth 容器都看不到。
- **MNT**：容器有自己的 mount 視圖，包括自己的 `/`。看不到 host 的 `/home`、`/var/lib/docker` 這些東西。
- **UTS**：隔離 hostname 和 domain name。容器改自己的 hostname 不會影響 host,反之亦然。
- **IPC**：隔離 System V IPC 和 POSIX message queue。不同容器之間不能用共享記憶體或 semaphore 互通。
- **USER**：UID/GID 對映。容器內的 root（uid=0）可以對映到 host 的非特權使用者,提高隔離強度。預設 Docker 不開,所以容器內外 user namespace inode 一樣。

### Host vs 容器 inode 對照

啟動長生命週期容器：

![啟動長生命週期容器](pic/啟動長生命週期容器.png)

取得容器在 host 上的 PID：

![取得容器在 Host 上的 PID](pic/取得容器在Host上的PID.png)

容器 `sleep 3600` 在 host 上 PID = **5141**。

檢視容器 process 的 namespace（host 視角）：

![檢視容器進程 Namespace](pic/檢視容器進程Namespace.png)

對比 Host PID 1 的 namespace：

![對比 Host 進程 Namespace](pic/對比Host進程Namespace.png)

對照表（同步存在 `namespace-table.md`）：

| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
| :--- | :--- | :--- | :--- |
| pid  | pid:[4026531836]  | pid:[4026533512]  | 否 |
| net  | net:[4026531833]  | net:[4026533514]  | 否 |
| mnt  | mnt:[4026531832]  | mnt:[4026532782]  | 否 |
| uts  | uts:[4026531838]  | uts:[4026533510]  | 否 |
| ipc  | ipc:[4026531839]  | ipc:[4026533511]  | 否 |
| user | user:[4026531837] | user:[4026531837] | 是 |

五種 namespace inode 不同 → 隔離生效；user namespace inode 相同 → 預設 Docker 沒開 user namespace remap，容器內的 root 就是 host 的 root。

用 nsenter 從 host 直接跳進容器 namespace 驗證：

![使用 nsenter 跳進容器 Namespace](pic/使用nsenter跳進容器Namespace.png)

UTS / NET 隔離效果（hostname、ip addr 兩邊不同）：

![驗證 UTS 與 NET 隔離效果](pic/驗證UTS與NET隔離效果.png)

### 容器內 `ps aux` 輸出

![進容器看 PID 隔離效果](pic/進容器看PID隔離效果.png)

容器內 `ps aux | wc -l` = 5（含 header 跟 `ps` 自己），實際 process 只看到三支：`sleep 3600`（PID 1）、`sh`、`ps aux`。

**為什麼只看到三支？** PID namespace 把 process ID 空間切開了 —— 容器只看得到「自己這個 namespace 裡」的 process。host 上同時跑著 dockerd、containerd、systemd、各種 daemon，但對容器來說那些都在另一個 PID namespace 裡，根本不可見。容器內的 PID 1 就是它的 init（這裡是 `sleep`），是這個 PID namespace 的祖宗。

---

## Cgroups 實驗

啟動資源限制容器：

![啟動資源限制容器](pic/啟動資源限制容器.png)

```
docker run -d --name cg-demo --memory=256m --cpus=0.5 alpine sleep 3600
```

### 容器內讀到的限制

![從容器內讀取 cgroup 限制值](pic/從容器內讀取cgroup限制值.png)

- memory.max：**268435456**（bytes，= 256 MiB，對到 `--memory=256m`）
- cpu.max：**50000 100000**（每 100ms 最多用 50ms CPU 時間 = 0.5 核，對到 `--cpus=0.5`）

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）

![Host 端 cgroup 路徑與數值對照](pic/Host端cgroup路徑與數值對照.png)

路徑：`/sys/fs/cgroup/system.slice/docker-${CID}.scope`

- memory.max：**268435456** ← 與容器內讀到的完全一致
- cpu.max：**50000 100000** ← 與容器內讀到的完全一致
- memory.current（執行時某一刻）：**344064**（約 336 KiB，sleep 幾乎不吃記憶體）

容器內外讀到一致的數值，證明 `docker run --memory=256m --cpus=0.5` 這兩個 flag 經 dockerd → containerd → runc 後，最終就是寫進這個 cgroup scope 目錄裡的 `memory.max` 和 `cpu.max` 檔案，kernel 依此執行限制。

CPU 限制實測：在 cg-demo 內背景跑 `dd if=/dev/zero of=/dev/null bs=1M count=50000`，從 host 端 `top` 觀察 %CPU 卡在 60.0% 附近（受採樣與 burst 影響在 50% 上下浮動），沒有衝到 100%，印證 0.5 核限制生效：

![觀察 CPU 限制表現](pic/觀察CPU限制表現.png)

### OOM 故障三階段

故障注入：

![故障注入觸發 OOM](pic/故障注入觸發OOM.png)

抓取 OOM 核心證據（dmesg 與 docker inspect）：

![抓取 OOM 核心證據](pic/抓取OOM核心證據.png)

回復測試（先因 alpine 預設 /dev/shm tmpfs 大小限制踩到 `No space left on device`，把 count 改成 40MB 後成功跑完）：

![資源放寬回復測試成功](pic/資源放寬回復測試成功.png)

| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | `Memory cgroup out of memory: Killed process 5775 (dd) ... oom_memcg=/system.slice/docker-8a90748e....scope` | 無 OOM |

**為什麼寫 `/dev/shm` 才能乾淨觸發 OOM？** `/dev/shm` 是 tmpfs，掛在記憶體裡，寫進去的資料會算進 memory cgroup；如果寫到 `/tmp/fill`（overlay2 可寫層），那是磁碟、不吃 RAM，永遠不會 OOM。用 `dd` 而不是 `stress-ng` 是因為 `dd` 是單一程序、PID 1 直接被殺，能乾淨拿到 ExitCode=137；stress-ng 的 fork-supervisor 設計會讓 worker 被殺後 parent 自己處理 SIGCHLD，最後容器整體 ExitCode=0 但 OOMKilled=true，教學上會混淆。

---

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量

> 排錯說明：`docker pull nginx:1.27-alpine` 與 `nginx:1.26-alpine` 因 DNS 解析失敗（`server misbehaving`）無法拉取，改用 host 上已存在的 `nginx:latest` 觀察 layer 結構。

![拉取同源映像檔](pic/拉取同源映像檔.png)

![檢視現有 Nginx 映像分層](pic/檢視現有Nginx映像分層.png)

`nginx:latest` 的 RootFS.Layers 共 **8 層** sha256：

```
92c4615992c3683ccd14763ed9c3820e97913e6bd7c6a1842df46225103a52ac
15826a0a30e43ec710260fb6379fbf1a64d20d45a8c29a3cccee9ca5489655b2
60b233ca54bacad3b5a7dd091aa7a1ab4e489a38e4a4491f110a5ec010f108
a5f65bcfce751cb0db38b9161ac0a9df560ff57ee82ae639c24f11650e4fe041
f3bb380dab96ec037d6823a76d247a989d998fa0e26115a44b44057e3c7d04e8
84120cfee353c5793a6b52e1b6255096c5a3cf909e986d9622fe1e020c790dba
fcf03f03e4d7740b09961d0de7700b6f9c8a43d17c1f516f76d0fa575234b42d
```

`docker history` 看每層對應的 Dockerfile 指令（CMD / STOPSIGNAL / EXPOSE / ENTRYPOINT / 一串 COPY 與 RUN 安裝 nginx 套件 / 各種 ENV / debian 基底）：

![檢視 Nginx 歷史分層與建置指令](pic/檢視Nginx歷史分層與建置指令.png)

### 兩個同源 image 共享 layer 的證據

啟動兩個基於 `nginx:latest` 的容器（fs-demo、fs-demo2）後，`/var/lib/docker/overlay2/` 總大小 = **444M**：

![驗證 Layer 共享機制](pic/驗證Layer共享機制.png)

兩個容器共用同一份 image，磁碟並沒有翻倍 → 大部分 lower layer 是兩個容器共享的同一份實體目錄，每個容器只多出自己那薄薄一層 upperdir（diff 目錄）。這就是內容定址（sha256）+ CoW 帶來的空間節省。

實體 overlay2 目錄與可寫層檔案：

![檢視實體 overlay2 目錄與可寫層檔案](pic/檢視實體overlay2目錄與可寫層檔案.png)

- merged：`/var/lib/docker/overlay2/084251deb58c...be18/merged`（容器內看到的統一視圖）
- upper：`/var/lib/docker/overlay2/084251deb58c...be18/diff`（可寫層的實體位置）
- lower：`...be18-init/diff:/var/lib/docker/overlay2/8b21712.../diff:.../e9b63a59.../diff:...`（一串冒號分隔的唯讀層）

upper 目錄裡看到 `tmp/hello.txt`、`etc/nginx/conf.d/custom.conf`，還有一個 `default.conf` 變成 `c--------- 0, 0` —— 這是 overlay2 的 **whiteout**，用 character device（major/minor=0,0）標記「此檔已被刪除」，讓 merged view 把下層同名檔案遮掉。

### `docker diff` 輸出範例與解讀

![觀察容器可寫層差異 diff](pic/觀察容器可寫層差異diff.png)

在 `fs-demo` 容器內執行：

```
echo hello > /tmp/hello.txt
rm /etc/nginx/conf.d/default.conf
echo custom > /etc/nginx/conf.d/custom.conf
```

`docker diff fs-demo` 輸出：

```
C /tmp
A /tmp/hello.txt
C /etc
C /etc/nginx
C /etc/nginx/conf.d
A /etc/nginx/conf.d/custom.conf
D /etc/nginx/conf.d/default.conf
```

解讀：
- **A**（Added）= 新增的檔案 → `/tmp/hello.txt`、`/etc/nginx/conf.d/custom.conf`
- **C**（Changed）= 目錄因為底下內容改變被標記為 changed → `/tmp`、`/etc`、`/etc/nginx`、`/etc/nginx/conf.d`
- **D**（Deleted）= 刪除的檔案 → `/etc/nginx/conf.d/default.conf`（在 overlay2 裡實際上是個 whiteout char device，把下層的同名檔案遮蔽掉）

所有變更只發生在這個容器的 upperdir，唯讀的 lower layers 完全沒動 → 其他容器看到的還是原版 image。

---

## OCI 呼叫鏈

檢視背景 daemon 進程：

![檢視背景 Daemon 進程](pic/檢視背景Daemon進程.png)

觀察 shim 與容器進程的父子關係：

![觀察 Shim 與容器進程關係](pic/觀察Shim與容器進程關係.png)

```
root  1608  /usr/bin/containerd
root  1653  /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root  5117  containerd-shim-runc-v2 ... id 59423c89ea2d (ns-demo)
  └─ root  5141  sleep 3600
root  5395  containerd-shim-runc-v2 ... id b5491936a17 (cg-demo)
  └─ root  5418  sleep 3600
root  6596  containerd-shim-runc-v2 ... id 35f1c1c9a733 (oci-demo)
  └─ root  6620  sleep 3600
```

OCI bundle 目錄與 `config.json`：

![查看 runc 的 OCI 封包](pic/查看runc的OCI封包.png)

![讀取 OCI 規範 JSON 內容](pic/讀取OCI規範JSON內容.png)

### 各層職責（用自己的話）

- **dockerd**：使用者友善的高階 API。處理 image build、network、volume、CLI 互動。本身不直接管容器生命週期，把生命週期相關工作丟給 containerd。
- **containerd**：image 管理（pull/push/儲存）、snapshot 管理、容器生命週期管理。對 dockerd 暴露 gRPC API，對下調用 runc 來真正起容器。Kubernetes 也是直接用 containerd 而不用 dockerd。
- **containerd-shim-runc-v2**：每個容器一支 shim。`runc` 在啟動容器時呼叫 `clone()` 把容器 process 丟出去就退出了，shim 留下來「接住」容器，持續持有它的 stdio、收集 exit code。這樣設計的好處是 containerd 自己可以重啟或升級，而不會把所有正在跑的容器一起帶走。
- **runc**：OCI Runtime Spec 的參考實作。讀 `config.json`，呼叫 `unshare()` / `clone(CLONE_NEW*)` 建立 namespace、把 PID 寫進 cgroup 控制檔、setcap 限制 capability、`pivot_root` 切到容器的 rootfs、最後 `execve` 容器 process。真正「執行容器」的就是它。

### OCI Runtime Spec `config.json` 對應到的 namespace / cgroup 設定

從本次截圖的 `config.json` 可以看到：

- `ociVersion`: "1.3.0" — 符合 OCI Runtime Spec 1.3.0
- `process.args`: `["sleep", "3600"]` — 容器 PID 1 要跑的指令
- `process.env`: `["PATH=...", "HOSTNAME=35f1c1c9a733"]`
- `process.cwd`: "/"
- `process.user`: `{uid: 0, gid: 0, additionalGids: [...]}` — 容器內以 root 跑
- `process.capabilities`: bounding / effective / permitted / inheritable / ambient 五組 cap 白名單，看到 `CAP_CHOWN`、`CAP_DAC_OVERRIDE`、`CAP_NET_RAW`、`CAP_NET_BIND_SERVICE`、`CAP_SYS_CHROOT`、`CAP_KILL` 等 14 個。預設容器不給 `CAP_SYS_ADMIN`、`CAP_NET_ADMIN` 這類危險 cap。
- `linux.namespaces`：列出 runc 要用 `unshare`/`clone` 建立的 namespace 類型（pid、network、mount、uts、ipc，預設不含 user）。**這就是 namespace 隔離設定的源頭。**
- `linux.resources`：cgroup 限制 → `memory.limit`、`cpu.shares` / `cpu.quota` / `cpu.period`、`pids.limit` 等。**`docker run --memory=256m --cpus=0.5` 就是被翻譯成這裡的欄位，runc 再寫進 `/sys/fs/cgroup/.../memory.max`、`cpu.max`。**
- `root.path`：指向容器的 rootfs 目錄（runc 會 pivot_root 到這裡）。
- `mounts`：容器內所有 mount point 的設定（/proc、/dev、/sys、tmpfs 等）。

OCI 規範保證了「同一份 config.json 任何符合規範的 runtime（runc、crun、kata-runtime）跑出來的容器是等價的」，這也是為什麼 Podman、CRI-O 能直接吃 Docker build 出來的 image。

---

## 排錯紀錄

### 排錯 1：docker pull nginx 失敗
- 症狀：`docker pull nginx:1.27-alpine` 與 `nginx:1.26-alpine` 都回 `Error response from daemon: Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io on 127.0.0.53:53: server misbehaving`。
- 診斷：systemd-resolved 在這個時段 DNS 解析不穩，但 host 上已經有 `nginx:latest` image 可以用。
- 修正：改用 `nginx:latest` 做後續的 layer 觀察與容器啟動。
- 驗證：`docker images | grep nginx` 看到 `nginx:latest 8d8e80999f5f 181MB`；`docker image inspect nginx:latest` 跟 `docker history nginx:latest` 都能正常輸出 8 層 sha256 與 Dockerfile 指令。

### 排錯 2：OOM 回復測試第一次反而失敗
- 症狀：放寬到 `--memory=256m` 重跑 `dd if=/dev/zero of=/dev/shm/big bs=1M count=200`，依然失敗，但訊息變成 `dd: error writing '/dev/shm/big': No space left on device`，跑了 64MB 就停。
- 診斷：這次不是 OOM。Alpine 容器 `/dev/shm` 預設 tmpfs 大小是約 64MB（看到 64.0MB copied 後就 No space），跟 `--memory=256m` 沒關係。`OOMKilled=false`、ExitCode=1 而不是 137 也可以佐證不是 OOM Kill。
- 修正：把 `count` 改成 40（40MB），低於 /dev/shm tmpfs 限制也遠低於 256MB 記憶體限制。
- 驗證：dd 成功跑完印出 `40+0 records in / 40+0 records out / 41943040 bytes (40.0MB) copied / DONE`，exit code 0，沒有任何 dmesg OOM log。

### 排錯 3：sudo 認證失敗
- 症狀：執行 `sudo ls -la /proc/$CPID/ns/` 跳出 `sudo: Authentication failed, try again.`
- 診斷：密碼輸錯。
- 修正：重新輸入正確密碼。
- 驗證：第二次成功列出容器 process 的 7 個 namespace symlink。

---

## 想一想（回答 3 題）

### 1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？

不是同一支。容器內的 PID 1 是 `sleep 3600`，host 視角它是 PID 5141；host 上的 PID 1 是 systemd。PID namespace 把同一支 process 從兩邊看，分別給它兩個編號。

容器內執行 `kill -9 1`：PID 1 是這個 PID namespace 的 init，kernel 對「PID namespace 內 init 自己送給自己的 fatal signal」會直接忽略（特殊保護，避免 namespace 沒 init 就崩掉），所以容器內自己對自己 `kill -9 1` 沒效果。但 host 上 root 用真正的 host PID（5141）送 SIGKILL 是可以殺的，殺掉之後 PID namespace 內所有 process 連帶被清掉，容器整體退出。

延伸：Docker 建議用 `--init` 是因為 PID 1 還有另一個責任 —— **回收 zombie**。一般應用程式（nginx、python script）沒寫 SIGCHLD handler，它派生的子孫 process 死掉後會變 zombie 卡在 process table。`--init` 會塞一個小的 init（tini 或 docker-init）當 PID 1 來幫忙 reap，順便正確轉送 SIGTERM 給子 process，讓 `docker stop` 能優雅關閉。

### 2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？

**共用**。Union FS（overlay2）把 image 拆成內容定址（sha256）的 layer，layer 內容一樣就是同一份實體目錄。兩個容器都基於 ubuntu:24.04 → 它們的 LowerDir 指到同一組 layer 目錄，磁碟只一份。各自只多出自己的 upperdir（diff 目錄），裡面只裝那個容器自己寫過的差異。

驗證方法（已在本次實驗中印證）：

```
docker pull ubuntu:24.04
du -sh /var/lib/docker/overlay2/                  # 記下 baseline
docker run -d --name a ubuntu:24.04 sleep 3600
du -sh /var/lib/docker/overlay2/                  # 多一個 upperdir，增加幾 KB
docker run -d --name b ubuntu:24.04 sleep 3600
du -sh /var/lib/docker/overlay2/                  # 又多一個 upperdir，再增加幾 KB
docker inspect a --format '{{.GraphDriver.Data.LowerDir}}'
docker inspect b --format '{{.GraphDriver.Data.LowerDir}}'   # 兩邊 LowerDir 指到同一組目錄
```

本次實驗的對應證據：開了 fs-demo、fs-demo2 兩個基於 `nginx:latest` 的容器後，`/var/lib/docker/overlay2/` 總大小 = 444M，遠小於 image 大小（181MB）× 2 + image 自己的 layer 量。

### 3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？

**不能稱為強隔離**。容器跟 host **共用同一個 kernel**，namespace 跟 cgroup 都是 kernel 提供的功能。kernel 本身爆漏洞（例如 Dirty COW CVE-2016-5195、CVE-2022-0185 等），容器內的 root 拿到 host root 只是時間問題 —— 攻擊者只要在容器內觸發那個 syscall path 就能跨出 namespace 邊界。歷史上有 runC 自己的漏洞（CVE-2019-5736）也讓容器逃逸過。

跟 VM 的根本差別：VM 多了一層 hypervisor。容器逃逸只要打穿 kernel；VM 逃逸要先打穿 guest kernel，再打穿 hypervisor（KVM、Xen）。hypervisor 的攻擊面小很多，而且 hypervisor 程式碼相對凍結、不像 Linux kernel 每天都在改 driver。

這也是為什麼 **Kata Containers / Firecracker / gVisor** 存在：

- **Kata Containers / Firecracker**：每個容器跑在一個極輕量的 microVM 裡，有自己的 guest kernel，獲得 VM 等級的隔離邊界，但啟動時間和資源開銷壓到接近原生容器（Firecracker 啟動 < 125ms）。AWS Lambda 跟 Fargate 就是用 Firecracker 跑多租戶 workload。
- **gVisor**：在用戶空間實作一個 syscall 攔截層（Sentry），容器的 syscall 不直接打 host kernel 而是先過 gVisor，等於再加一層 sandbox，攻擊面從整個 kernel 縮到 gVisor 自己這小塊程式碼。

一句話：**普通容器是同棟公寓不同房間，VM 是不同棟，Kata/Firecracker 是把房間蓋成迷你公寓**。多租戶或不可信代碼執行（CI runner、Lambda 之類）就應該用後兩者。
