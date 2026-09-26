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

本文檔提供 SideFX Houdini 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點，並包含完整的環境隔離防護。

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

    # 4. HOUDINI_PATH 基礎錨點：無條件設定為 "&"
    # 注意：絕對不可用 if not defined() 判斷！
    # 若外層環境已存在 HOUDINI_PATH，一旦判斷為 True 導致此處未設定，
    # 後續外掛 (如 htoa) 的 prepend 動作會被 Rez 判定為該變數的初次賦值而覆寫全域，
    # 導致 "&" 遺失、Houdini 原生節點全部消失。因此此處必須無條件重設為 "&"。
    env.HOUDINI_PATH = "&"

    # 5. 生產環境嚴格隔離防護 (防止使用者本地檔案污染農場)
    # 禁止讀取使用者的本機 houdini.env 檔案
    env.HOUDINI_NO_ENV_FILE = 1
    
    # 限制 package json 搜尋路徑，避免使用者個人目錄下 (如 ~/houdini20.5/packages) 的舊外掛被偷偷載入
    env.HOUDINI_PACKAGE_DIR = f"{hfs_root}/packages"
```

---

## 三、環境變數與架構設計說明

1. **`HFS` 核心指標**：
   Houdini 所有內部工具、PySide 封裝及 HOM（Houdini Object Model）皆高度依賴 `HFS`。只要 `HFS` 與 `PATH` 設定完成，執行 `hython` 時系統會自動找到其內建的 Python 模組，**無須手動將內嵌 Python site-packages 強行塞入全域 `PYTHONPATH`**。
2. **`HOUDINI_PATH = "&"` 無條件初始化**：
   末端的 `&` 代表保留 Houdini 原生路徑；後續的外掛（如 Arnold HtoA）在載入時執行 `env.HOUDINI_PATH.prepend("{root}")`，即可形成 `<htoa_root>;&` 的完美搜尋順序。
3. **隔離本機 `houdini.env` 與 `packages/`**：
   美術人員常在本機手動安裝各類測試外掛，這些配置若未被 Rez 接管，會造成「美術本機可開，農場算圖噴錯」的經典不一致問題。透過 `HOUDINI_NO_ENV_FILE=1` 與鎖定 `HOUDINI_PACKAGE_DIR`，強制所有外掛必須由 Rez 控制。
4. **移除多餘的 `dsolib` 與 `LD_LIBRARY_PATH`**：
   - Windows 的 `custom/houdini/dsolib` 放的是 HDK 編譯專用的 `.lib` 靜態連結檔，非執行期 DLL，加入 `PATH` 無實質意義。
   - Linux 版 Houdini 主程式自帶 ELF `RPATH`，無需手動 prepend `LD_LIBRARY_PATH`，避免與系統庫或其它 DCC 混用時發生衝突。

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

### 2. 農場渲染命令驗證（Karma 渲染測試）
```bash
rez-env houdini-20.5.278 -- hrender -e -f 1 10 -d karma1 /mnt/x/jobs/shot010/fx.hip
```
