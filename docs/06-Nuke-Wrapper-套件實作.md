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

# 版本命名注意：
# 若設定為 "15.1v1"，Rez 內部會將版本切分為 [15, "1v1"] 兩個 token。
# 請求 "nuke-15.1" 會因 "1" != "1v1" 而找不到套件；必須請求精確的 "nuke-15.1v1" 或 "nuke-15"。
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
    # Foundry Nuke 的主程式通常直接置於根目錄，無獨立 bin/ 資料夾
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

    # 4. 命令列別名封裝 (Alias) 與批次執行注意事項
    major_minor = ".".join(ver_str.split("v")[0].split(".")[:2])
    
    if system.platform == "windows":
        exe_base = f"Nuke{major_minor}.exe"
        # 互動式 shell 提供 doskey alias
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

## 三、環境變數與農場算圖陷阱

1. **Windows `alias()` 在批次/農場下的失效陷阱**：
   Rez 在 Windows cmd 環境下是透過 `doskey` 實作 alias。然而 **`doskey` 僅在互動式命令列下生效**。若在 Deadline、批次檔（.bat / .cmd）或無介面呼叫中執行 `rez-env nuke -- nuke -t`，Windows 會回報找不到 `nuke` 指令。
   - **解法一（農場推薦）**：算圖排程系統提交時直接呼叫完整名稱（如 `Nuke15.1.exe -t`）。
   - **解法二**：在 package 目錄下建立一個 `bin/nuke.cmd` 的 shim 腳本，將其加入 PATH 供系統自動調用。
2. **Nuke 命令列參數 `-c` 誤區**：
   > [!warning] Nuke 的 `-c` 不是執行 Python 字串
   > 與標準 Python 的 `python -c "print(1)"` 不同，Nuke CLI 的 `-c <size>` 代表**設定快取記憶體大小（Cache Size）**。若要透過命令列執行 Python 腳本測試，必須建立一個實體 `.py` 檔案並以 `nuke -t test_script.py` 執行。
3. **無須手動設定 `LD_LIBRARY_PATH`**：
   Nuke 原廠自帶完整的 RPATH 尋址機制，避免手動 prepend `LD_LIBRARY_PATH` 造成與外部工具的動態函式庫衝突。

---

## 四、驗證測試指令

### 1. 終端機無介面（Terminal 模式）測試
建立一個快速測試腳本並執行：

```powershell
# 建立測試腳本
Set-Content -Path "$env:TEMP\test_nuke.py" -Value "import nuke; print('Nuke 核心載入成功！版本:', nuke.NUKE_VERSION_STRING)"

# 透過 rez-env 呼叫 Nuke Terminal 模式執行
rez-env nuke-15.1v1 -- nuke -t "$env:TEMP\test_nuke.py"
```

預期輸出：
```text
Nuke 核心載入成功！版本: 15.1v1
```

### 2. 農場渲染命令驗證
```bash
rez-env nuke-15.1v1 -- nuke -x -F 1-100 /mnt/x/jobs/comp/shot010_comp.nk
```
