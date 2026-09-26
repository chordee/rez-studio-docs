---
tags:
  - dev
  - dev/pipeline
  - rez
  - houdini
  - arnold
  - htoa
  - payload
aliases:
  - Rez HtoA Arnold Houdini 外掛實作
---
# HtoA (Arnold for Houdini) 外掛套件實作指南

本文檔示範將 **Arnold for Houdini (HtoA)** 渲染器打包為 Rez 自包含實體負載（Payload）套件的標準流程。本套件支援 Windows 工作站與 Linux 農場節點，並結合 Local Package Cache 機制。

---

## 一、檔案存放位置與實體負載結構

### 1. 套件定義檔
放置於：`X:/rez-system/packages/htoa/6.3.3.0/package.py`（Linux: `/mnt/x/rez-system/packages/htoa/6.3.3.0/package.py`）

### 2. 實體目錄樹（Rez 標準巢狀變體階層）

> [!important] ABI 相容性驗證
> Autodesk 官方 HtoA 6.3.3.0 正式支援的 Houdini 20.5 系列 Build 為 **`20.5.278`**（亦有 20.0.751 / 19.5.805 等對應包）。在部署時必須下載官方標註支援該精確 Build 的壓縮包，解壓並對齊目錄。

```text
X:/rez-system/packages/htoa/6.3.3.0/
├── package.py                                  # Rez 套件設定檔
│
├── platform-windows/
│   └── houdini-20.5.278/                       # Windows + H20.5.278 巢狀子目錄
│       ├── arnold/
│       │   ├── bin/ (kick.exe, maketx.exe, ai.dll)
│       │   └── include/
│       ├── dso/ (htoa.dll, arnold_dso.dll)
│       ├── otls/ (arnold_operators.hda, arnold_vop.hda)
│       └── scripts/
│           ├── bin/
│           └── python/
│
└── platform-linux/
    └── houdini-20.5.278/                       # Linux + H20.5.278 巢狀子目錄
        ├── arnold/
        │   └── bin/ (kick, maketx, libai.so)
        ├── dso/ (htoa.so)
        ├── otls/
        └── scripts/
```

---

## 二、完整 `package.py` 程式碼

```python
# -*- coding: utf-8 -*-
name = "htoa"

version = "6.3.3.0"

description = "Solid Angle / Autodesk Arnold for SideFX Houdini (HtoA)"

authors = ["Autodesk", "Solid Angle"]

# 宣告支援本機快取與可重定位
relocatable = True
cachable = True

# 二進位 ABI 變體矩陣宣告（嚴格對齊官方編譯 Build）
variants = [
    ["platform-windows", "houdini-20.5.278"],
    ["platform-linux", "houdini-20.5.278"]
]

# 指令行工具清單
tools = [
    "kick",
    "maketx",
    "oiiotool",
    "arnold"
]

def commands():
    # 1. 核心注入：將當前 Variant 的根目錄置頂於 HOUDINI_PATH
    # 註：官方標準安裝通常產生一個 JSON 檔案至 Houdini packages 目錄；
    # 在 Rez 架構中，將 {root} 加入 HOUDINI_PATH 為業界常見的動態等價替代方案，
    # Houdini 啟動時會自動遍歷 {root} 底下的 dso, otls, scripts 等標準子資料夾。
    env.HOUDINI_PATH.prepend("{root}")

    # 2. 注入 Arnold Core 獨立執行檔 (kick, maketx)
    env.PATH.prepend("{root}/arnold/bin")
    env.PATH.prepend("{root}/scripts/bin")

    # 3. 作業系統專屬動態函式庫路徑配置
    if system.platform == "windows":
        env.PATH.prepend("{root}/dso")
    elif system.platform == "linux":
        env.LD_LIBRARY_PATH.prepend("{root}/arnold/bin")
        env.LD_LIBRARY_PATH.prepend("{root}/dso")

    # 4. Python API 與輔助模組
    env.PYTHONPATH.prepend("{root}/scripts/python")

    # 5. Arnold 外掛與著色器搜尋路徑初始化 (使用 Rex API defined)
    if not defined("ARNOLD_PLUGIN_PATH"):
        env.ARNOLD_PLUGIN_PATH = ""
    env.ARNOLD_PLUGIN_PATH.append("{root}/arnold/plugins")
```

---

## 三、技術細節與環境變數疊加機制

### 1. Houdini Path 疊加原理（Wrapper 與 Payload 共生）
在 [04-Houdini-Wrapper-套件實作](04-Houdini-Wrapper-套件實作.md) 中，Houdini Wrapper 宣告了 `env.HOUDINI_PATH = "&"`。
當同時載入 `houdini-20.5.278` 與 `htoa-6.3.3.0` 時：
* 最終合成的環境變數為：
  `X:/rez-system/packages/htoa/6.3.3.0/platform-windows/houdini-20.5.278;&`
* 末端的 `&` 確保 Houdini 原生內部節點正常讀取，前面的 `{root}` 確保 Arnold 節點優先載入。

### 2. 農場渲染指令相容性（Kick CLI）
照明合成任務可直接利用 Arnold Standalone 引擎運算 `.ass` 檔案：
```bash
rez-env htoa-6.3.3.0 -- kick -i /mnt/x/jobs/scene.ass -o /mnt/x/renders/beauty.exr
```

---

## 四、驗證測試指令

### 1. 驗證 HtoA 在 Houdini 內的 Python 模組載入
在 Windows 或 Linux 終端機執行：

```bash
rez-env houdini-20.5.278 htoa-6.3.3.0 -- hython -c "import htoa; print('HtoA 載入成功，路徑:', htoa.__file__)"
```

### 2. 驗證 Arnold 獨立渲染引擎版本
```bash
rez-env htoa-6.3.3.0 -- kick -info
```
