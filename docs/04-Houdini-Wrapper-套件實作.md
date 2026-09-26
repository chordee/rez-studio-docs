---
tags:
  - dev
  - dev/pipeline
  - rez
  - houdini
  - wrapper
aliases:
  - Rez Houdini Wrapper 套件實作
---
# Houdini Wrapper 套件跨平台實作指南

本文檔提供 SideFX Houdini 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點。

---

## 一、檔案存放位置

放置於中央網路儲存庫：
`X:/rez-system/packages/houdini/20.5.278/package.py`（Linux: `/mnt/x/rez-system/packages/houdini/20.5.278/package.py`）

---

## 二、完整 `package.py` 程式碼

```python
# -*- coding: utf-8 -*-
name = "houdini"

version = "20.5.278"

description = "SideFX Houdini DCC Wrapper Package (Default Installation)"

authors = ["Studio Pipeline Team"]

# Wrapper 僅為指標，禁止被本機快取系統複製
cachable = False

tools = [
    "houdini",
    "houdinifx",
    "hython",
    "hbatch",
    "hscript",
    "ginfo",
    "gconvert",
    "hrender"
]

def commands():
    import os
    
    # 取得套件版本字串
    ver_str = str(this.version)

    # 1. 依據作業系統解析官方預設安裝路徑
    if system.platform == "windows":
        hfs_root = f"C:/Program Files/Side Effects Software/Houdini {ver_str}"
    elif system.platform == "linux":
        hfs_root = f"/opt/hfs{ver_str}"
    else:
        stop(f"不支援的作業系統架構: {system.platform}")

    # 2. 檢驗安裝實體是否存在（Fail Fast）
    if not os.path.exists(hfs_root):
        stop(
            f"[Rez Wrapper] 找不到 Houdini 安裝目錄！\n"
            f"預期路徑: {hfs_root}\n"
            f"請確認該機器是否已將 Houdini 安裝在官方預設路徑。"
        )

    # 3. 核心環境變數配置
    env.HFS = hfs_root
    env.HB = f"{hfs_root}/bin"
    
    # 將 Houdini 的二進位執行檔置於 PATH 優先列
    env.PATH.prepend(f"{hfs_root}/bin")

    # 4. Houdini 專屬外掛與資產搜尋路徑初始化 (使用 Rex API defined)
    # 預留 '&' 符號確保 Houdini 原生內部配置與預設路徑不受破壞
    if not defined("HOUDINI_PATH"):
        env.HOUDINI_PATH = "&"

    # 5. 作業系統底層動態連結庫 (DSO)
    if system.platform == "windows":
        env.PATH.append(f"{hfs_root}/custom/houdini/dsolib")
    elif system.platform == "linux":
        env.LD_LIBRARY_PATH.prepend(f"{hfs_root}/dsolib")
```

---

## 三、環境變數與架構設計說明

1. **`HFS` 核心指標**：
   Houdini 所有內部工具、PySide 封裝及 HOM（Houdini Object Model）皆高度依賴 `HFS`。只要 `HFS` 與 `PATH` 設定完成，執行 `hython` 時系統會自動找到其內建的 Python 模組，**無須手動將內嵌 Python site-packages 強行塞入全域 `PYTHONPATH`**，避免污染外部 Python 工具。
2. **`HOUDINI_PATH = "&"` 與 Rex API**：
   使用 Rex 標準語法 `if not defined("HOUDINI_PATH")` 檢查。末端的 `&` 代表保留 Houdini 原生路徑；後續的外掛（如 Arnold HtoA）只需呼叫 `env.HOUDINI_PATH.prepend("{root}")`，即可無損完成層次堆疊。
3. **`cachable = False`**：
   顯式宣告此套件不可快取，防止農場節點誤將空包 Wrapper 目錄判定為需要同步的快取項目。

---

## 四、驗證測試指令

### 1. 終端機無介面驗證（Hython 測試）
在 Windows 工作站或 Linux 農場節點執行：

```bash
rez-env houdini-20.5.278 -- hython -c "import hou; print('Houdini Version:', hou.applicationVersionString())"
```

預期輸出：
```text
Houdini Version: 20.5.278
```

### 2. 農場渲染命令驗證
```bash
rez-env houdini-20.5.278 -- hrender -e -f 1 10 -d mantra1 /mnt/x/jobs/shot010/fx.hip
```
