# W07｜Docker Compose 與資料持久化

## 拓樸圖
（mermaid 或 ASCII，標出 app、db、default network、db-data volume）

## 從 docker run 到 compose.yaml
（自己的話：你最有感的一個改善是什麼？）

## 三種掛載對照
| 掛載類型 | 路徑（host） | 容器砍重起資料還在嗎 | 重啟容器資料狀態 | 適合情境 |
|---|---|---|---|---|
| named volume |  |  |  |  |
| bind mount |  |  |  |  |
| tmpfs |  |  |  |  |

## healthcheck 前後對照
| 寫法 | curl /healthz t=1s | t=3s | t=5s | t=10s |
|---|---|---|---|---|
| 只 depends_on |  |  |  |  |
| service_healthy |  |  |  |  |

觀察（自己的話）：

## 排錯紀錄
- 症狀：
- 診斷：
- 修正：
- 驗證：

## 設計決策
（為什麼 db 用 named volume 而不是 bind mount？為什麼不能在生產用 tmpfs 存資料庫？）
