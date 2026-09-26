---
tags:
  - dev
  - dev/pipeline
  - rez
  - maya
  - usd
  - maya_usd
  - payload
aliases:
  - Rez Maya-USD 外掛套件實作
---
# Maya-USD 外掛套件實作指南

本文檔示範將 **Autodesk Maya-USD** 開源外掛打包為 Rez 自包含實體負載（Payload）套件的標準流程。示範如何透過 Autodesk 模組化規範（Maya Module `.mod`）實現乾淨註冊，並探討套件不可變性與治理策略。

---

## 一、檔案存放位置與實體負載結構

### 1. 套件定義檔
放置於：`X:/rez-system/packages/maya_usd/0.28.0/package.py`（Linux: `/mnt/x/rez-system/packages/maya_usd/0.28.0/package.py`）

### 2. 實體檔案目錄樹（官方 Release 二層結構）

官方 Maya-USD release 編譯成品在解壓後通常包含 `MayaUSD`（外掛本體）與 `USD`（Pixar USD 核心函式庫）兩大結構：

```text
X:/rez-system/packages/maya_usd/0.28.0/
├── package.py                                  # Rez 套件設定檔
│
├── platform-windows/
│   └── maya-2024/                              # Windows + Maya 2024 巢狀子目錄
│       ├── mayaUSD.mod                         # Autodesk 模組定義檔（CMake 標準命名）
│       ├── mayausd/
│       │   ├── MayaUSD/                        # MayaUSD 外掛本體 (plugin, scripts)
│       │   └── USD/                            # Pixar USD 核心 dll, schemas, python
│       └── ...
│
└── platform-linux/
    └── maya-2024/                              # Linux + Maya 2024 巢狀子目錄
        ├── mayaUSD.mod
        ├── mayausd/
        │   ├── MayaUSD/
        │   └── USD/
        └── ...
```

### 3. 重要前置步驟：改寫 `mayaUSD.mod` 為相對路徑（保證可重定位）

CMake 建置產出的官方檔名為 **`mayaUSD.mod`**（Linux 系統區分大小寫，務必統一）。Autodesk 安裝程式產生的 `.mod` 通常寫死安裝位置的本機絕對路徑（如 `C:\Program Files\Autodesk\MayaUSD\...`）。
當將外掛移至中央 NAS 套件庫（`X:/rez-system/...`）或啟用 `relocatable = True` 搭配 `cache_packages_path` 快取至本機 NVMe 時，寫死的絕對路徑會導致外掛依然讀取原安裝路徑，使重定位與快取完全失效。

> [!warning] Maya 模組語法規則與路徑陷阱
> 參照官方模組範本（如 `mayaUSD_Win.mod.template`），一個完整的 Maya-USD 包含三大模組段落：
> 1. `+ USD <USD_VERSION> <INSTALL_DIR>/mayausd/USD`
> 2. `+ MayaUSD_LIB <MAYAUSD_VERSION> <INSTALL_DIR>/mayausd/MayaUSD`（在此定義核心 DLL `PATH+:=lib`、Python 模組 `PYTHONPATH+:=lib/python` 與 Schema `PXR_PLUGINPATH_NAME+:=lib/usd`）
> 3. `+ MayaUSD <MAYAUSD_VERSION> <INSTALL_DIR>/mayausd/MayaUSD/plugin/adsk`（在此定義 `plug-ins: plugin`、`icons:` 等 UI 資源）
> 
> **關鍵語法規則**：`.mod` 內的 `icons:`、`plug-ins:`、`scripts:` 等子路徑，是**相對於各段 `+` 宣告的模組根目錄（Module Root Path），而非相對於 `.mod` 檔案本身**！且 `<USD_VERSION>` 等版本號是由建置系統注入的確切數值。
> **正確做法**：絕不更動內部的模組定義或子路徑，**僅將每個 `+` 開頭行末尾的絕對安裝根目錄改為相對路徑（`./...`）**。

在發布套件前，執行以下 Python 腳本完成路徑相對化：

```python
import logging
from pathlib import Path

log = logging.getLogger(__name__)

def relativize_mod(mod_file: Path, install_root: str) -> None:
    root = install_root.replace("\\", "/").rstrip("/")
    lines: list[str] = []
    for line in mod_file.read_text(encoding="utf-8").splitlines():
        if line.startswith("+") and root in line.replace("\\", "/"):
            line = line.replace("\\", "/").replace(root, ".")
            log.info("rewrote: %s", line)
        lines.append(line)
    mod_file.write_text("\n".join(lines) + "\n", encoding="utf-8")

# 針對發布庫中的 mayaUSD.mod 進行相對路徑改寫
relativize_mod(
    Path("X:/rez-system/packages/maya_usd/0.28.0/platform-windows/maya-2024/mayaUSD.mod"),
    "C:/Program Files/Autodesk/MayaUSD/Maya2024/0.28.0",  # ← 填入原 .mod 內記錄的安裝根目錄
)
```

改寫完成後，每個模組根目錄皆相對於 `mayaUSD.mod` 所在目錄（`./mayausd/...`），無論掛載於何處或複製至本機快取目錄，Maya 皆能正確定位。

---

## 二、完整 `package.py` 程式碼

在正式生產環境中，Maya-USD 自帶的 `mayaUSD.mod` 內部已經完整定義了該外掛專屬的 `PATH`、`PYTHONPATH`、`MAYA_PLUG_IN_PATH` 與 `PXR_PLUGINPATH_NAME`。

因此在 Rez 中**最乾淨且防止重複註冊的做法是直接宣告 `MAYA_MODULE_PATH`**，僅針對 Windows Python 3.8+ 特殊的獨立 Python 載入機制補上 `PXR_USD_WINDOWS_DLL_PATH`：

```python
# -*- coding: utf-8 -*-
name = "maya_usd"

version = "0.28.0"

description = "Autodesk Maya USD Plugin (Universal Scene Description)"

authors = ["Autodesk", "Pixar"]

# 宣告支援快取與可重定位
relocatable = True
cachable = True

# 變體矩陣：綁定作業系統與對應的 Maya 版本
variants = [
    ["platform-windows", "maya-2024"],
    ["platform-linux", "maya-2024"]
]

def commands():
    # 1. 單一來源授權：透過 Maya Module (.mod) 管理完整外掛路徑
    # 避免手動重複追加 MAYA_PLUG_IN_PATH 與 PYTHONPATH 導致雙重註冊或版本踩踏
    env.MAYA_MODULE_PATH.append("{root}")

    # 2. Windows Python 3.8+ 專用 DLL 尋址補丁（供 mayapy / 獨立腳本使用）
    # Python 3.8+ 在 Windows 上不再依賴 PATH 搜尋 .pyd 相依的 DLL，
    # Pixar USD 官方設計透過 PXR_USD_WINDOWS_DLL_PATH 自動呼叫 os.add_dll_directory
    if system.platform == "windows":
        env.PXR_USD_WINDOWS_DLL_PATH.append("{root}/mayausd/USD/lib")
        env.PXR_USD_WINDOWS_DLL_PATH.append("{root}/mayausd/MayaUSD/lib")
```

> [!warning] `PXR_USD_WINDOWS_DLL_PATH` 的跨 DCC 副作用與隔離原則
> 當系統定義了 `PXR_USD_WINDOWS_DLL_PATH` 變數時，Pixar USD 的 C++ 載入器在 Windows 下將**優先讀取該變數並停止從全域 `PATH` 尋址**。
> 若在同一個 Rez Context 環境中同時載入 `maya_usd` 與 `houdini`，Houdini 內建的 USD 函式庫（`hython` / Solaris / Karma）也可能會讀取該變數，導致 Houdini 意外載入 Maya-USD 的 USD DLL，引發嚴重的二進位相容性崩潰。
> **管線治理準則**：
> 1. **嚴禁跨 DCC 混用外掛 Payload**：絕不要在同一個執行環境中同時請求 `maya_usd` 與 `houdini`。
> 2. 若純粹在 Maya 內部作業（GUI 啟動），Maya 的 `.mod` 機制本就具備完整的 DLL 導向功能；若無需在外部透過獨立 Python 存取 `pxr`，亦可評估省略此變數以最大化隔離安全性。

---

## 三、套件不可變性與追加 Variant 的治理策略

### 1. Rez 的底層行為
從 Rez 技術底層來看：
- 若為已存在的版本（如 `0.28.0`）**在 `variants` 尾端追加新項目**（例如追加 `["platform-windows", "maya-2025"]`），既有的 variant index 索引不會改變，舊有的 `.rxt` 依然可繼續解析。
- `package.py` 的 mtime 改變時，Memcached 的 package definition 快取也會自動失效更新。

### 2. 工作室政策建議（強烈推薦提升版本號）
儘管技術上允許尾端追加，但在大型團隊治理中，**將套件視為完全不可變（Immutable）** 是最佳實踐：
- 避免因編輯 `package.py` 時手民之誤破壞了舊有 variant 的相依字串。
- 若支援新版 DCC（如 Maya 2025），建議發布 `0.28.1` 或 `0.29.0`，明確標註更新日誌，確保生產線各專案版本追溯透明清晰。

---

## 四、深入驗證測試指令

### 1. 驗證 Module 註冊路徑（嚴格確認非本機絕對路徑）
透過 `mayapy` 執行 `cmds.getModulePath(moduleName='MayaUSD')`，確認回傳的路徑為中央 NAS（`X:/...`）或本機快取目錄（`C:/rez-cache/...`），而非原始安裝的 `C:/Program Files/...` 絕對路徑：

```powershell
rez-env maya-2024 maya_usd-0.28.0 -- mayapy -c "
import maya.standalone
maya.standalone.initialize()
import maya.cmds as cmds

mod_path = cmds.getModulePath(moduleName='MayaUSD')
print('MayaUSD Module Path:', mod_path)
assert 'Program Files' not in mod_path, '錯誤：模組路徑依然指向本機安裝目錄而非 Rez 套件庫！'
print('模組相對路徑重定位驗證成功！')
"
```

### 2. 功能與節點階層驗證（修復 DAG Hierarchy）

在驗證 Maya-USD 外掛時，若直接建立 `mayaUsdProxyShape`，Maya 會預設建立一個 `transform1` 作為父節點。因此在測試腳本中，**必須明確建立 Transform 父節點**，以驗證 UFE 與 Stage 正確繫結：

```powershell
rez-env maya-2024 maya_usd-0.28.0 -- mayapy -c "
import maya.standalone
maya.standalone.initialize()
import maya.cmds as cmds

# (A) 測試載入 Maya-USD 核心外掛
cmds.loadPlugin('mayaUsdPlugin')
print('Maya-USD 外掛載入成功！版本:', cmds.pluginInfo('mayaUsdPlugin', q=True, v=True))

# (B) 正確建立 Transform 與 Proxy Shape 階層
transNode = cmds.createNode('transform', name='testUsdStage')
shapeNode = cmds.createNode('mayaUsdProxyShape', name='testUsdStageShape', parent=transNode)
print('USD Proxy Shape 節點建立成功:', shapeNode)

# (C) 測試 Python UFE 與 USD Stage 存取
import mayaUsd.ufe
stage = mayaUsd.ufe.getStage('|testUsdStage|testUsdStageShape')
print('UFE Stage 取得正常:', stage)
"
```

預期輸出：
```text
MayaUSD Module Path: X:/rez-system/packages/maya_usd/0.28.0/platform-windows/maya-2024/mayausd/MayaUSD/plugin/adsk
模組相對路徑重定位驗證成功！
Maya-USD 外掛載入成功！版本: 0.28.0
USD Proxy Shape 節點建立成功: testUsdStageShape
UFE Stage 取得正常: ...
```
各項測試皆通過，方能確認 Maya、Maya-USD、UFE、二進位 DLL 及 Python API 皆已 100% 正確組裝且路徑重定位無誤。
