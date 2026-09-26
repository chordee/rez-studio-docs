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

本文檔示範將 **Autodesk Maya-USD** 開源外掛打包為 Rez 自包含實體負載（Payload）套件的標準流程。示範如何結合 Autodesk 模組化規範（Maya Module `.mod`）與 Rez 環境變數注入機制，並遵守套件不可變性原則。

---

## 一、檔案存放位置與實體負載結構

### 1. 套件定義檔
放置於：`X:/rez-system/packages/maya_usd/0.28.0/package.py`（Linux: `/mnt/x/rez-system/packages/maya_usd/0.28.0/package.py`）

### 2. 實體檔案目錄樹（Rez 標準巢狀變體階層）

```text
X:/rez-system/packages/maya_usd/0.28.0/
├── package.py                                  # Rez 套件設定檔
│
├── platform-windows/
│   └── maya-2024/                              # Windows + Maya 2024 巢狀子目錄
│       ├── mayaUsd.mod                         # Autodesk 官方模組定義檔
│       ├── plugin/adsk/plugin/                 # mayaUsdPlugin.mll
│       ├── lib/
│       │   ├── mayaUsd.dll
│       │   ├── python/ (mayaUsd/, ufe/ 模組)
│       │   └── usd/    (USD Schema 函式庫)
│       └── resources/
│
└── platform-linux/
    └── maya-2024/                              # Linux + Maya 2024 巢狀子目錄
        ├── mayaUsd.mod
        ├── plugin/adsk/plugin/                 # mayaUsdPlugin.so
        ├── lib/
        │   ├── libmayaUsd.so
        │   ├── python/
        │   └── usd/
        └── resources/
```

---

## 二、完整 `package.py` 程式碼

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
    # 1. 核心載入方式：註冊 Autodesk Maya Module (.mod)
    # 將包含 mayaUsd.mod 的根目錄注入，供 Maya GUI 自動掛載選單與資源
    env.MAYA_MODULE_PATH.append("{root}")

    # 2. 額外強化注入：確保在無介面批次模式 (mayapy) 下亦能無縫引用
    env.MAYA_PLUG_IN_PATH.append("{root}/plugin/adsk/plugin")
    env.MAYA_SCRIPT_PATH.append("{root}/plugin/adsk/scripts")
    env.PYTHONPATH.append("{root}/lib/python")

    # 3. 作業系統二進位動態連結庫相依
    if system.platform == "windows":
        env.PATH.prepend("{root}/lib")
        env.PXR_USD_WINDOWS_RESOURCE_PATH.append("{root}/lib/usd")
    elif system.platform == "linux":
        env.LD_LIBRARY_PATH.prepend("{root}/lib")

    # 4. Pixar USD 核心 Schema 與外掛宣告路徑 (使用 Rex API defined)
    if not defined("PXR_PLUGINPATH_NAME"):
        env.PXR_PLUGINPATH_NAME = ""
    env.PXR_PLUGINPATH_NAME.append("{root}/lib/usd")
```

---

## 三、套件不可變性（Package Immutability）守則

> [!danger] 嚴禁修改已發布的 `package.py` 來追加 Variant
> **千萬不可**在已經正式發布的 `maya_usd/0.28.0/package.py` 內直接修改程式碼去加入 `maya-2025` 變體！
> 
> 原因如下：
> 1. **破壞 Context 可重現性**：過去已烘焙存檔的 `.rxt` Context 依賴當初的套件定義特徵碼，修改既有檔案會導致環境還原不一致。
> 2. **破壞快取一致性**：Memcached 上的 Package Definition 與各農場節點上的 Local Cache 都會產生快取脫節。
> 
> **正確做法**：
> 當專案需要升級支援 Maya 2025 時，必須編譯新成品並發布為**新版本**（例如 `0.28.1` 或 `0.29.0`），在新版本的 `package.py` 內定義新的 variants 矩陣。

---

## 四、深入驗證測試指令

單純執行 `from pxr import Usd` 只能證明 Pixar USD Python 繫結存在，不能證明 Maya 專屬的 USD 外掛、UFE（Universal Front End）與 Maya Proxy Shape 正常。

在 Windows 或 Linux 上執行完整的 Standalone 冒煙測試：

```powershell
rez-env maya-2024 maya_usd-0.28.0 -- mayapy -c "
import maya.standalone
maya.standalone.initialize()
import maya.cmds as cmds

# 1. 測試載入 Maya-USD 核心外掛
cmds.loadPlugin('mayaUsdPlugin')
print('Maya-USD 外掛載入成功！版本:', cmds.pluginInfo('mayaUsdPlugin', q=True, v=True))

# 2. 測試建立 Maya USD Proxy Shape 節點
node = cmds.createNode('mayaUsdProxyShape', name='testUsdStage')
print('USD Proxy Shape 節點建立成功:', node)

# 3. 測試 Python UFE 與 USD Stage 存取
import mayaUsd.ufe
stage = mayaUsd.ufe.getStage('|testUsdStage|testUsdStageShape')
print('UFE Stage 取得正常:', stage)
"
```

預期輸出：
```text
Maya-USD 外掛載入成功！版本: 0.28.0
USD Proxy Shape 節點建立成功: testUsdStage
UFE Stage 取得正常: ...
```
三項測試皆通過，方能確認 Maya、Maya-USD、UFE、二進位 DLL 及 Python API 皆已 100% 正確組裝。
