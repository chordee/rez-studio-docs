# HtoA (Arnold for Houdini) 外掛套件實作指南

本文檔示範將 **Arnold for Houdini (HtoA)** 渲染器打包為 Rez 自包含實體負載（Payload）套件的標準流程。本套件支援 Windows 工作站與 Linux 農場節點，並結合 Local Package Cache 機制。

---

## 一、檔案存放位置與實體負載結構

### 1. 套件定義檔
放置於：`X:/rez-system/packages/htoa/6.3.3.0/package.py`（Linux: `/mnt/x/rez-system/packages/htoa/6.3.3.0/package.py`）

### 2. 實體目錄樹與路徑規則

> [!IMPORTANT]
> **路徑設定與 ABI 配對說明**
>
> 1. **執行檔路徑與內容依據**：官方 HtoA installer 產生的 `htoa.json` 經實查會將 `HTOA_ROOT` prepend 至 `HOUDINI_PATH`，並將 `$HTOA_ROOT/scripts/bin` prepend 至 `PATH`；但 JSON 不會列出該目錄包含哪些檔案。本文列出的 Arnold CLI 工具（`kick`, `maketx`）與核心動態函式庫（`ai.dll` / `libai.so`）位置，以對應版本官方安裝包的實際解壓結果為準；部署每個 release 時都必須重新核對。
> 2. **Houdini 與 HtoA 版本配對**：本文以 HtoA 6.3.3.0 與 Houdini 20.5.278 作為示範配對案例。實際生產環境請依據工作室採用的具體 Houdini Production/Daily Build，下載官方釋出之完全對應 Build。
> 3. **跨平台 Python 與編譯器 ABI 選型規範**：實查 HtoA 6.5.1.0／Houdini 21.0.631 的官方 `htoa.json`，其中只針對 Linux `gcc.11.2` 宣告 `_gcc11` 的 `HTOA_GCC_SUFFIX`，沒有宣告 Python suffix；因此不可把單一 JSON 當成跨版本 Python ABI 的通用依據。安裝包必須與目標機台 Houdini 的 ABI 完全吻合。**切勿主觀臆測特定版本（如 H20.5）的預設編譯器或 Python 版本**，必須於目標機台執行 `hython -c "import hou; print(hou.applicationPlatformInfo())"` 實機查詢真實的 platform build 字串（Windows 輸出如 `windows-x86_64-cl19.xx`，Linux 輸出如 `linux-x86_64-gcc...`），並比對 Python 直譯器版本，再下載放置完全相符的 HtoA release，否則載入 DSO 時將引發二進位不相容崩潰。

```text
X:/rez-system/packages/htoa/6.3.3.0/
├── package.py                                  # Rez 套件設定檔
│
├── platform-windows/
│   └── houdini-20.5.278/                       # Windows + H20.5.278 巢狀子目錄
│       ├── config/                             # UI、Icon 與外掛設定
│       ├── dso/                                # htoa.dll, arnold_dso.dll
│       ├── otls/                               # arnold_operators.hda, arnold_vop.hda
│       ├── soho/                               # Arnold ROP / SOHO 整合（必要）
│       ├── toolbar/                            # Houdini 工具列
│       └── scripts/
│           ├── bin/                            # kick.exe, maketx.exe, ai.dll
│           └── python/                         # htoa 模組與 arnold.py
│
└── platform-linux/
    └── houdini-20.5.278/                       # Linux + H20.5.278 巢狀子目錄
        ├── config/
        ├── dso/                                # htoa.so
        ├── otls/
        ├── soho/
        ├── toolbar/
        └── scripts/
            ├── bin/                            # kick, maketx, libai.so
            └── python/
```

> [!IMPORTANT]
> **Payload 必須保留完整官方 release root**
>
> 上圖只列出主要目錄，不是可挑選複製的白名單。應使用 HtoA 安裝器的「僅解壓」結果作為各 variant 的完整根目錄，保留該 release 的所有原始子目錄與檔案。Autodesk 特別指出 `soho/` 必須位於 Houdini factory SOHO 路徑之前；缺少它可能使 Arnold ROP 與輸出流程不完整。`HOUDINI_PATH.prepend("{root}")` 會讓 Houdini 從完整 root 推導 `dso`、`otls`、`scripts`、`soho`、`toolbar` 與 `config` 等對應搜尋路徑。

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
    # Houdini 啟動時會從完整 release root 推導 dso、otls、scripts、soho、toolbar、config 等搜尋路徑。
    env.HOUDINI_PATH.prepend("{root}")

    # 2. 注入 Arnold Core 獨立執行檔 (kick, maketx) 與動態庫搜尋路徑
    # 官方 installer 產生的 htoa.json 將 PATH 指向 scripts/bin；實際內容以安裝包解壓結果為準
    env.PATH.prepend("{root}/scripts/bin")

    # 註：不設定 PYTHONPATH。
    # Houdini 載入 hou 時會把每個 $HOUDINI_PATH 項目的 scripts/python 加入 sys.path，
    # 因此 HtoA 的 Python 模組已可在 Houdini／hython 內 import；官方 htoa.json 也只設定 HOUDINI_PATH 與 PATH。
    # 額外加入 PYTHONPATH 會把 HtoA 模組暴露給整個 Rez 環境與所有子行程（包含非 Houdini 的 Python）。

    # 3. 注意：切勿將 {root}/dso 加入 ARNOLD_PLUGIN_PATH！
    # dso/ 內存放的是 Houdini 的 DSO 二進位檔（htoa.dll / htoa.so），已由 HOUDINI_PATH 自動載入。
    # 若加入 ARNOLD_PLUGIN_PATH，Arnold 會嘗試將其當成 Arnold 外掛載入而引發 Warning 或 Crash。
    # HtoA 自身會處理內建外掛路徑，ARNOLD_PLUGIN_PATH 應保留給工作室自製著色器套件。
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
> [!IMPORTANT]
> **依賴相依提醒**
>
> 由於 `htoa` 的 variants 宣告了 `houdini-20.5.278`，因此執行 `rez-env htoa -- kick` 時，Rez 會**連帶解析出 Houdini Wrapper 套件**。
> 這意味著純算圖節點若要使用 HtoA 內建的 `kick` 算圖，該節點本機也必須安裝有對應的 Houdini 軟體（否則會被 Houdini Wrapper 的 `stop()` 攔截）。若工作室需要完全脫離 Houdini 安裝的純 CPU/GPU 算圖節點，建議另外封裝獨立的 `arnold_core` 套件。

---

## 四、驗證測試指令

### 1. 驗證 HtoA 在 Houdini 內的 Python 模組載入
在 Windows 或 Linux 終端機執行：

```bash
rez-env houdini-20.5.278 htoa-6.3.3.0 -- hython -c "import htoa; print('HtoA 載入成功，路徑:', htoa.__file__)"
```

由於 `package.py` 未設定 `PYTHONPATH`，此測試同時驗證 Houdini 能經由 `HOUDINI_PATH` 推導出的 `{root}/scripts/python` 找到 HtoA 模組；印出的路徑應位於該 HtoA variant 目錄內。

### 2. 驗證 Arnold 獨立渲染引擎版本
```bash
rez-env houdini-20.5.278 htoa-6.3.3.0 -- kick -info
```
