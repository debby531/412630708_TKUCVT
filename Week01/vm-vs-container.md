# VM vs Container 技術對照分析

## 技術對照表
| 維度 | Virtual Machine (VM) | Container (Docker) |
|---|---|---|
| **隔離層級** | 硬體級 (Hardware Level) | 作業系統級 (OS Level) |
| **啟動速度** | 分鐘級 (需啟動完整的 Guest OS) | 秒級 (僅啟動程序) |
| **資源消耗** | 高 (固定配置 CPU/RAM) | 極低 (共享 Host Kernel) |
| **移植性** | 依賴 Hypervisor 格式 (如 .ova) | 極高 (Docker Image 隨處執行) |

## 為什麼在本課選擇「VM 裡跑 Docker」？
1. **環境隔離**：避免在我的 macOS (Host) 直接安裝大量工具導致系統混亂。
2. **安全性**：在虛擬機中進行故障演練，即使操作錯誤也不會影響到我的 Mac。
3. **一致性**：確保與教學環境 Ubuntu 25.10 相同，減少 M4 Mac 硬體帶來的架構問題。

## Hypervisor 比較
- **Type 1 (Bare Metal)**：直接裝在硬體上（如 ESXi）。
- **Type 2 (Hosted)**：裝在現有作業系統上（如本課使用的 **VMware Fusion**）。
