# W06｜Docker Image 與 Dockerfile

**姓名：** 劉芷庭  
**學號：** 412630708  
**實驗環境：** app VM（無外網）；映像與 Python 套件由 Mac 經閘道 HTTP 傳入後 `docker load`／離線 wheels 安裝。

---

## 映像組成

- **Layers 是什麼：** 每一條 Dockerfile 指令（如 `RUN`、`COPY`）會產生一層唯讀的檔案系統差異；多個映像可共用相同 layer（對應 overlay2 的 lower 層），不會重複佔滿硬碟。
- **Config 是什麼：** 描述容器如何啟動的 JSON 中繼資料，例如 `Cmd`、`Entrypoint`、`Env`、`WorkingDir`、`User`、`ExposedPorts`，不包含應用程式原始碼本體。
- **Manifest 是什麼：** 把各 layer 的 digest 與 config 綁在一起的索引；registry 與 daemon 靠它辨識完整映像由哪些層組成。

---

## python:3.12-slim inspect 摘錄

![Part A 步驟 1](w06-partA-01-docker-images-python.png)

![Part A 步驟 2](w06-partA-02-docker-history.png)

![Part A 步驟 3](w06-partA-03-docker-image-inspect.png)

![Part A 步驟 4](w06-partA-04-shared-layers.png)

> 載入正確映像後，與本實驗 build 所用 base 一致。

- **Config.Cmd：** `["python3"]`
- **Config.Env：** `PATH=/usr/local/bin:/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin`、`LANG=C.UTF-8`、`PYTHON_VERSION=3.12.13` 等
- **Config.WorkingDir：** 未設定（空，預設 `/`）
- **RootFS.Layers 數量：** **4**

---

## Layer 快取實驗

| 情境 | build 時間 |
|---|---|
| v1 首次 build | **0m1.052s** |
| v1 改 app.py 後 rebuild | **0m1.051s** |
| v2 首次 build | **0m1.164s** |
| v2 改 app.py 後 rebuild | **0m0.128s** |

![v1 首次 build](w06-partB-08-build-v1.png)

![v1 全 CACHED 重 build](w06-partC-10-rebuild-cached.png)

![v1 改程式後 rebuild](w06-partC-11-v1-rebuild-after-change.png)

![Dockerfile.v2](w06-partC-12-Dockerfile-v2.png)

![v2 兩次 build 對照](w06-partC-13-build-v2-compare.png)

**觀察（用自己的話寫）：**  
v1 把 `COPY app/` 放在 `pip install` 之前，改一行 `app.py` 會讓 `COPY` 層 cache miss，後面所有層（含 `pip install`）都要重跑，所以時間幾乎跟首次一樣。  
v2 先 `COPY requirements.txt` 再 `pip install`，最後才 `COPY app/`；只改程式時，裝套件那幾層仍是 CACHED，只有複製程式與之後的層重算，所以第二次只要 0.128s。

---

## CMD vs ENTRYPOINT 實驗

| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | `argv = ['show_args.py', 'default1', 'default2']`、`PID = 7` | `exec: "extra1": executable file not found` |
| CMD exec form | `argv = ['show_args.py', 'default1', 'default2']`、`PID = 1` | `exec: "extra1": executable file not found` |
| ENTRYPOINT + CMD | `argv = ['show_args.py', 'default1', 'default2']`、`PID = 1` | `argv = ['show_args.py', 'extra1', 'extra2']`、`PID = 1` |

![不帶參數執行](w06-partD-18-run-no-args.png)

![帶參數執行](w06-partD-19-run-with-args.png)

![show_args.py](w06-partD-15-show-args.png)

![三種 Dockerfile](w06-partD-16-three-dockerfiles.png)

![build argtest](w06-partD-17-build-argtest.png)

**結論（用自己的話寫）：**  
`docker run` 後面的參數會整條覆蓋 `CMD`，不會覆蓋 `ENTRYPOINT`。固定主程式應用 ENTRYPOINT（exec form），預設參數放 CMD，執行時才能附加參數。exec form 的 PID 1 是 Python 本身，較能正確處理 SIGTERM；shell form 多一層 sh，PID 1 不是應用程式。

---

## Multi-stage 大小對照

| Image | SIZE |
|---|---|
| python:3.12（builder base） | 本環境未拉取；builder 改以 `python:3.12-slim` 代替 |
| python:3.12-slim（runtime base） | **144MB** |
| myapp:v2（單階段） | **159MB** |
| myapp:multi（多階段） | **149MB** |

![docker images myapp](w06-partE-21-docker-images-myapp.png)

![Dockerfile.multi](w06-partE-22-Dockerfile-multi.png)

![build multi](w06-partE-23-build-multi-size.png)

![history v2 vs multi](w06-partE-24-history-v2-multi.png)

![執行 multi](w06-partE-25-run-multi.png)

![docker images -a](w06-partE-26-images-a.png)

**解釋（用自己的話寫）：**  
builder stage 的 layer 不會進最終 `myapp:multi` 的映像，所以 `docker history myapp:multi` 看不到 builder 裡的 `pip install`。它們以 `<untagged>` 等形式留在本機 build cache（`docker images -a` 可見），push 到 registry 的通常只有 runtime stage。

---

## .dockerignore 故障注入

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| `du -sh .` | **712K** | **151M** | **716K** |
| build context 傳輸大小 | — | **794B** | **794B** |
| build 時間 | — | **0m0.163s** | **0m0.122s** |

![故障前](w06-partF-27-baseline-du.png)

![故障注入](w06-partF-28-fault-inject.png)

![故障中 build](w06-partF-29-build-dirty.png)

![建立 .dockerignore](w06-partF-30-dockerignore.png)

![回復後 build](w06-partF-31-build-clean.png)

![清理後](w06-partF-32-cleanup.png)

---

## 排錯紀錄

- **症狀：** `docker build` 時 `pip: not found`；`python:3.12-slim` 只有約 8.7MB；`python3 --version` 失敗。
- **診斷：** 本機映像實為 alpine 卻被標成 `python:3.12-slim`；app VM 無外網無法 `pull`／容器內無法連 PyPI。
- **修正：** Mac 匯出映像經 HTTP 傳入後 `docker load`；下載 Linux aarch64 wheels 至 `app/wheels/`，Dockerfile 使用 `--no-index --find-links=wheels`。
- **驗證：** `Python 3.12.13`；`myapp:v1` build 成功；`curl` 取得 `Hey from ... | version=final`。

![最終驗證](w06-final-verify.png)

---

## 設計決策

**Runtime 選 `python:3.12-slim` 而不是 `alpine`：**  
slim 使用 glibc，多數 Python 套件有現成 wheel，較少 musl 編譯問題。alpine 雖更小，但常需額外安裝編譯工具；本實驗以穩定、可離線完成為優先。

**無外網環境：**  
base 映像離線 `docker load`；依賴套件預先放入 `app/wheels/`。multi-stage 的 builder 因無法取得 `python:3.12` 完整版，改以 `python:3.12-slim` 代替；本專案僅 Flask，無需編譯 C extension，仍可完成實驗。

---

## Part B 補充截圖

![建立目錄](w06-partB-05-mkdir.png)

![app 檔案](w06-partB-06-app-files.png)

![Dockerfile.v1](w06-partB-07-Dockerfile-v1.png)

![v1 curl 驗證](w06-partB-09-run-curl.png)

---

## 可重跑最小命令鏈

```bash
cd ~/virt-container-labs/w06
docker build -f Dockerfile.multi -t myapp:multi .
docker run -d --name myapp-final -p 8080:80 -e APP_VERSION=final myapp:multi app.py
sleep 3
curl http://localhost:8080/
docker stop myapp-final && docker rm myapp-final
