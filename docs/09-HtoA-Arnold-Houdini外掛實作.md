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

### 2. 實體目錄樹（對齊官方 HtoA 實際解壓結構）

> [!important] ABI 相容性驗證與實體目錄
> Autodesk 官方 HtoA 6.3.3.0 正式支援的 Houdini 20.5 系列 Build 為 **`20.5.278`**。
> 官方 HtoA 解壓後的實際檔案分佈中，Arnold 核心二進位檔（`ai.dll` / `libai.so`）與 CLI 工具（`kick`, `maketx`）皆統一存放於 **`scripts/bin/`** 底下，並無獨立的 `arnold/bin/`。

```text
X:/rez-system/packages/htoa/6.3.3.0/
├── package.py                                  # Rez 套件設定檔
│
├── platform-windows/
│   └── houdini-20.5.278/                       # Windows + H20.5.278 巢狀子目錄
│       ├── dso/                                # htoa.dll, arnold_dso.dll
│       ├── otls/                               # arnold_operators.hda, arnold_vop.hda
│       └── scripts/
│           ├── bin/                            # kick.exe, maketx.exe, ai.dll
│           └── python/                         # htoa 模組與 arnold.py
│
└── platform-linux/
    └── houdini-20.5.278/                       # Linux + H20.5.278 巢狀子目錄
        ├── dso/                                # htoa.so
        ├── otls/
        └── scripts/
            ├── bin/                            # kick, maketx, libai.so
            └── python/
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
    "oiiotool"
]

def commands():
    # 1. 核心注入：將當前 Variant 的根目錄置頂於 HOUDINI_PATH
    # 註：官方標準安裝通常產生一個 JSON 檔案至 Houdini packages 目錄；
    # 在 Rez 架構中，將 {root} 加入 HOUDINI_PATH 為業界常見的動態等價替代方案，
    # Houdini 啟動時會自動遍歷 {root} 底下的 dso, otls, scripts 等標準子資料夾。
    env.HOUDINI_PATH.prepend("{root}")

    # 2. 注入 Arnold Core 獨立執行檔 (kick, maketx) 與 DLL 搜尋路徑
    # 官方 HtoA 二進位檔均置於 scripts/bin
    env.PATH.prepend("{root}/scripts/bin")

    # 3. Python API 與輔助模組
    env.PYTHONPATH.prepend("{root}/scripts/python")

    # 4. Arnold 外掛與著色器搜尋路徑（無需預先空字串初始化，直接 append）
    env.ARNOLD_PLUGIN_PATH.append("{root}/dso")
```

---

## 三、技術細節與環境變數疊加機制

### 1. Houdini Path 疊加原理（Wrapper 與 Payload 共生）
在 [04-Houdini-Wrapper-套件實作](04-Houdini-Wrapper-套件實作.md) 中，Houdini Wrapper 宣告了 `env.HOUDINI_PATH = "&"`。
當同時載入 `houdini-20.5.278` 與 `htoa-6.3.3.0` 時：
* 最終合成的環境變數為：
  `X:/rez-system/packages/htoa/6.3.3.0/platform-windows/houdini-20.5.278;&`
* 末端的 `&` 確保 Houdini 原生內部節點正常讀取，前面的 `{root}` 確保 Arnold 節點優先載入。

### 2. Standalone Kick 算圖的依賴約束
> [!important] 依賴相依提醒
> 由於 `htoa` 的 variants 宣告了 `houdini-20.5.278`，因此執行 `rez-env htoa -- kick` 時，Rez 會**連帶解析出 Houdini Wrapper 套件**。
> 這意味著純算圖節點若要使用 HtoA 內建的 `kick` 算圖，該節點本機也必須安裝有對應的 Houdini 軟體（否則會被 Houdini Wrapper 的 `stop()` 攔截）。若工作室需要完全脫離 Houdini 安裝的純 CPU/GPU 算圖節點，建議另外封裝獨立的 `arnold_core` 套件。

---

## 四、驗證測試指令

### 1. 驗證 HtoA 在 Houdini 內的 Python 模組載入
在 Windows 或 Linux 終端機執行：

```bash
rez-env houdini-20.5.278 htoa-6.3.3.0 -- hython -c "import htoa; print('HtoA 載入成功，路徑:', htoa.__file__)"
```

### 2. 驗證 Arnold 獨立渲染引擎版本
```bash
rez-env houdini-20.5.278 htoa-6.3.3.0 -- kick -info
```
