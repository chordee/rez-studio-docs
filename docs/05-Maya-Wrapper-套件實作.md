---
tags:
  - dev
  - dev/pipeline
  - rez
  - maya
  - wrapper
aliases:
  - Rez Maya Wrapper 套件實作
---
# Maya Wrapper 套件跨平台實作指南

本文檔提供 Autodesk Maya 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點。

---

## 一、檔案存放位置

放置於中央網路儲存庫：
`X:/rez-system/packages/maya/2024/package.py`（Linux: `/mnt/x/rez-system/packages/maya/2024/package.py`）

---

## 二、完整 `package.py` 程式碼

```python
# -*- coding: utf-8 -*-
name = "maya"

version = "2024"

description = "Autodesk Maya DCC Wrapper Package (Default Installation)"

authors = ["Studio Pipeline Team"]

# Wrapper 僅為指標，禁止被本機快取系統複製
cachable = False

tools = [
    "maya",
    "mayapy",
    "mayabatch",
    "Render"
]

def commands():
    import os
    
    # 取得套件版本字串 (例如 "2024")
    ver_str = str(this.version)

    # 1. 依據作業系統解析官方預設安裝路徑
    if system.platform == "windows":
        maya_root = f"C:/Program Files/Autodesk/Maya{ver_str}"
    elif system.platform == "linux":
        maya_root = f"/usr/autodesk/maya{ver_str}"
    else:
        stop(f"不支援的作業系統架構: {system.platform}")

    # 2. 檢驗安裝實體是否存在
    if not os.path.exists(maya_root):
        stop(
            f"[Rez Wrapper] 找不到 Autodesk Maya 安裝目錄！\n"
            f"預期路徑: {maya_root}\n"
            f"請確認該機器是否已將 Maya 安裝在官方預設路徑。"
        )

    # 3. 核心環境變數配置
    env.MAYA_LOCATION = maya_root
    
    # 將 Maya 的二進位執行檔目錄置頂 (包含 maya, mayapy, Render 以及 Windows 的 mayabatch.exe)
    env.PATH.prepend(f"{maya_root}/bin")
```

---

## 三、環境變數與架構設計說明

1. **`MAYA_LOCATION` 的權威性**：
   Autodesk 原生工具、算圖指令 `Render` 與 Python 直譯器 `mayapy` 均高度依賴 `MAYA_LOCATION`。`mayapy` 啟動時會自動解析並組裝內部的 Python 環境，**無須手動將 Maya 內嵌 Python 加入全域 `PYTHONPATH`**，確保環境隔離。
2. **`Render` 算圖入口支援**：
   `Render` 是 Deadline 等農場派遣系統啟動 Maya 算圖任務的標準 CLI 入口。透過 `env.PATH.prepend("{maya_root}/bin")`，農場還原環境後可直接觸發 `Render`。
3. **`mayabatch` 與 Linux 差異**：
   `mayabatch.exe` 在 Windows 上原生位於 `{maya_root}/bin` 目錄下，因為該目錄已加入 `PATH`，可直接呼叫，**無須設置多餘的 `alias()`**（避免遇到 Windows cmd 下 doskey 的批次失效問題）。在 Linux 上執行無介面批次作業時，標準呼叫方式為 `maya -batch`。
4. **移除 Linux `LD_LIBRARY_PATH` 與空字串初始化**：
   - Maya 本身依賴自身的 RPATH 讀取庫檔，不應全域 prepend `LD_LIBRARY_PATH`。
   - 諸如 `MAYA_MODULE_PATH` 等變數無需在 Wrapper 中預先初始化為空字串，外掛套件（如 Maya-USD）在首次 `append()` 時即可正確建立變數。

---

## 四、驗證測試指令

### 1. `mayapy` 無介面測試
在終端機執行 standalone 啟動指令：

```bash
rez-env maya-2024 -- mayapy -c "import maya.standalone; maya.standalone.initialize(); import maya.cmds as cmds; print('Maya Version:', cmds.about(v=True))"
```

預期輸出：
```text
Maya Version: 2024
```

### 2. 農場渲染指令測試
驗證 `Render` 指令是否正確綁定：

```bash
rez-env maya-2024 -- Render -help
```
