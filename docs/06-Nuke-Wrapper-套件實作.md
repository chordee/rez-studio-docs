---
tags:
  - dev
  - dev/pipeline
  - rez
  - nuke
  - wrapper
aliases:
  - Rez Nuke Wrapper 套件實作
---
# Nuke Wrapper 套件跨平台實作指南

本文檔提供 Foundry Nuke 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點。

---

## 一、檔案存放位置

放置於中央網路儲存庫：
`X:/rez-system/packages/nuke/15.1v1/package.py`（Linux: `/mnt/x/rez-system/packages/nuke/15.1v1/package.py`）

---

## 二、完整 `package.py` 程式碼

```python
# -*- coding: utf-8 -*-
name = "nuke"

version = "15.1v1"

description = "Foundry Nuke DCC Wrapper Package (Default Installation)"

authors = ["Studio Pipeline Team"]

# Wrapper 僅為指標，禁止被本機快取系統複製
cachable = False

tools = [
    "nuke",
    "nukex",
    "nukestudio"
]

def commands():
    import os
    
    # 取得套件版本字串 (例如 "15.1v1")
    ver_str = str(this.version)

    # 1. 依據作業系統解析官方預設安裝路徑
    # 注意：Foundry Nuke 的執行檔通常直接置於根目錄，無獨立 bin/ 資料夾
    if system.platform == "windows":
        nuke_root = f"C:/Program Files/Nuke{ver_str}"
    elif system.platform == "linux":
        nuke_root = f"/usr/local/Nuke{ver_str}"
    else:
        stop(f"不支援的作業系統架構: {system.platform}")

    # 2. 檢驗安裝實體是否存在
    if not os.path.exists(nuke_root):
        stop(
            f"[Rez Wrapper] 找不到 Foundry Nuke 安裝目錄！\n"
            f"預期路徑: {nuke_root}\n"
            f"請確認該機器是否已將 Nuke 安裝在官方預設路徑。"
        )

    # 3. 核心環境變數配置
    env.NUKE_LOCATION = nuke_root
    
    # 將 Nuke 根目錄置於 PATH 最前列
    env.PATH.prepend(nuke_root)

    # 4. 初始化 Nuke 外掛與自訂 Gizmo 搜尋路徑 (NUKE_PATH，使用 Rex API defined)
    if not defined("NUKE_PATH"):
        env.NUKE_PATH = ""

    # 5. Linux 專屬動態函式庫路徑配置
    if system.platform == "linux":
        env.LD_LIBRARY_PATH.prepend(nuke_root)

    # 6. 命令列別名封裝 (Alias)
    # 解決 Windows 執行檔名稱帶有版本號 (如 Nuke15.1.exe) 的調用差異
    major_minor = ".".join(ver_str.split("v")[0].split(".")[:2])
    
    if system.platform == "windows":
        exe_base = f"Nuke{major_minor}.exe"
        alias("nuke", f'"{nuke_root}/{exe_base}"')
        alias("nukex", f'"{nuke_root}/{exe_base}" --nukex')
        alias("nukestudio", f'"{nuke_root}/{exe_base}" --studio')
    else:
        bin_base = f"Nuke{major_minor}"
        alias("nuke", f'"{nuke_root}/{bin_base}"')
        alias("nukex", f'"{nuke_root}/{bin_base}" --nukex')
        alias("nukestudio", f'"{nuke_root}/{bin_base}" --studio')
```

---

## 三、環境變數與架構設計說明

1. **目錄結構特性（無獨立 `bin/`）**：
   Foundry 官方安裝包將核心主程式 `Nuke15.1.exe`（Linux: `Nuke15.1`）以及眾多動態函式庫直接釋放在安裝根目錄下。因此必須將 `nuke_root` 本身加入 `PATH`。
2. **`alias` 命令列別名封裝**：
   官方執行檔常包含主版本號（例如 `Nuke15.1`），為了讓農場算圖腳本能統一以 `nuke -t` 或 `nukex` 觸發，Wrapper 內建建立了 `alias`，抹平指令名稱差異。
3. **`NUKE_PATH` 堆疊機制**：
   使用 Rex API `if not defined("NUKE_PATH")` 初始化後，後續的專案外掛只需執行 `env.NUKE_PATH.append(...)` 即可自動註冊 Gizmo 與選單腳本。

---

## 四、驗證測試指令

### 1. 終端機無介面（Terminal 模式）測試
在 Windows 工作站或 Linux 農場節點執行：

```bash
rez-env nuke-15.1v1 -- nuke -t -c "import nuke; print('Nuke Version:', nuke.NUKE_VERSION_STRING)"
```

預期輸出：
```text
Nuke Version: 15.1v1
```

### 2. 農場渲染命令驗證
```bash
rez-env nuke-15.1v1 -- nuke -x -F 1-100 /mnt/x/jobs/comp/shot010_comp.nk
```
