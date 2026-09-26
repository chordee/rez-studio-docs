---
tags:
  - dev
  - dev/pipeline
  - rez
  - payload
  - variants
aliases:
  - Rez 外掛 Payload 套件規範與 Variants 機制
---
# 外掛 Payload 套件規範與 Variants 多維度變體機制

本文檔解析 Rez 體系中與 Wrapper 完全相反的另一種核心套件型態：**實體負載套件（Payload Package / Self-contained Package）**。我們將透過外掛（Plugin）的維護視角，深入探討二進位相容性、`rez-bind` 基礎依賴、`variants` 巢狀路徑規則、`cachable` 快取宣告，以及 `{root}` 動態路徑解析機制。

---

## 一、Wrapper 套件 vs Payload 套件本質對比

| 比較維度 | Wrapper 套件（DCC 主程式） | Payload 套件（外掛 / 工具鏈） |
| :--- | :--- | :--- |
| **典型代表** | Houdini、Maya、Nuke 軟體本體 | Arnold (HtoA / MtoA)、Maya-USD、自製 C++ 外掛 |
| **檔案大小** | 僅數 KB（只有一個 `package.py`） | 幾百 MB 至數 GB（包含大量編譯好的二進位檔、外掛、腳本） |
| **實體檔案位置** | 各工作站與農場節點本機（`C:\Program Files` 等） | **直接存放在中央 NAS 套件庫**（`X:\rez-system\packages\...`） |
| **快取支援** | `cachable = False`（不複製） | 可宣告 `cachable = True` 與 `relocatable = True` |
| **尋址機制** | 在 `commands()` 依作業系統尋找本機預設安裝路徑 | 直接使用 `{root}`，指向當前 Variant 所在的網路儲存目錄 |

---

## 二、什麼是 `variants`（變體 / 多維度矩陣）？

在影視工業中，C++ 外掛（例如 Arnold 的 `.dll` 或 `.so`）具有嚴格的 **二進位應用程式介面（ABI）相依性**：
1. 為 Houdini 20.5.278 編譯的 Arnold 外掛，**不能**載入到其他未相容的 Houdini build（可能觸發未定義符號錯誤或 Crash）。
2. 在 Windows 上需要 `.dll`，在 Linux 上需要 `.so`。
3. **Linux 上的編譯器與 Python ABI 維度**：在 Linux 環境中，外掛不僅對齊 Houdini 版本，還區分 Python 版本（`HTOA_PY_SUFFIX`，如 `.py310`、`.py311`）與 GCC 版本（`HTOA_GCC_SUFFIX`，如 `_gcc9`、`_gcc11`）。放置於 Linux variant 目錄的 payload 必須與農場 Houdini 的 `houdini_platform_build` 完全匹配（Houdini 20.5 預設為 Python 3.11 + GCC 11.2）。

### 1. 基礎相依性前提：`platform` 必須已綁定
在套件中宣告 `platform-windows` 或 `platform-linux` 之前，中央儲存庫必須已依序執行過 `rez-bind platform`、`arch`、`os`（參見 [01-中央伺服器與全域配置](01-中央伺服器與全域配置.md)），否則解析時會報 `PackageFamilyNotFoundError: package family not found: platform`。

### 2. Rez 的宣告語法

在 `package.py` 中，透過一個二維陣列宣告該外掛所支援的宿主環境與平台組合：

```python
variants = [
    ["platform-windows", "houdini-20.5.278"],
    ["platform-linux", "houdini-20.5.278"]
]
```

---

## 三、中央儲存庫的實體目錄結構（Rez 標準巢狀規則）

> [!important] Rez 實體目錄的巢狀規則
> Rez 在儲存套件變體時，**不會**將多個 requirement 透過底線串接為單一資料夾，而是依照 `variants` 宣告的 requirement 順序，在檔案系統中建立**巢狀子目錄（Nested Subpaths）**。

```text
X:/rez-system/packages/htoa/6.3.3.0/ (Linux: /mnt/x/rez-system/packages/htoa/6.3.3.0/)
├── package.py                                  # 定義檔 (描述 variants 與 commands)
│
├── platform-windows/                           # 第 1 層：作業系統
│   └── houdini-20.5.278/                       # 第 2 層：宿主軟體 Build
│       ├── dso/                                # htoa.dll
│       ├── otls/                               # Arnold 專屬 HDA 數位資產
│       └── scripts/
│           ├── bin/ (kick.exe, maketx.exe, ai.dll)
│           └── python/
│
└── platform-linux/
    └── houdini-20.5.278/                       # 需存放對齊農場 ABI 之 build (預設 Python 3.11 + GCC 11.2)
        ├── dso/                                # htoa.so
        ├── otls/
        └── scripts/
            ├── bin/ (kick, maketx, libai.so)
            └── python/
```

> [!note] Linux Variant 實體 Build 放置規範
> 官方釋出之 Linux HtoA 包含多種 Python 及 GCC 組合（例如 `HtoA ... Linux (GCC 11.2) Python 3.11`）。在當前 `variants` 設計下，`platform-linux/houdini-20.5.278` 資料夾內必須精確放置與農場 Houdini `houdini_platform_build` 相符的二進位建置。若未來農場同時存在多種 Python 或 GCC 版本，則需在 variants 中進一步擴展相應維度。

---

## 四、`{root}` 動態展開與快取宣告（`cachable`）

### 1. `{root}` 的動態指向
在 Payload 套件中，**嚴禁寫死任何儲存路徑**。
`package.py` 內建提供了 `{root}`（或 `this.root`）關鍵字：
- 當在 Windows 上載入 `houdini-20.5.278` 時，`{root}` 會自動指向：
  `X:/rez-system/packages/htoa/6.3.3.0/platform-windows/houdini-20.5.278`
- 當在 Linux 上載入時，`{root}` 會自動指向：
  `/mnt/x/rez-system/packages/htoa/6.3.3.0/platform-linux/houdini-20.5.278`

### 2. 快取宣告：`cachable` 與 `relocatable`
為了讓算圖農場節點將此套件複製到本機高速 SSD（`cache_packages_path`）以保護 NAS 頻寬，必須在 `package.py` 明確宣告：

```python
# 宣告此套件內的二進位檔不依賴寫死的絕對路徑，可被移動至快取目錄
relocatable = True

# 允許 Rez 將此套件複製到 local package cache
cachable = True
```

若套件依賴了硬編碼的特定路徑，則切勿設定 `relocatable = True`，否則快取至本機後會因路徑失效而報錯。

---

## 五、實作篇章導覽

1. [09-HtoA-Arnold-Houdini外掛實作](09-HtoA-Arnold-Houdini外掛實作.md)：HtoA 二進位 Payload 完整目錄、`HOUDINI_PATH` 堆疊與 Standalone Kick 算圖配置。
2. [10-Maya-USD外掛套件實作](10-Maya-USD外掛套件實作.md)：Autodesk Maya-USD 模組化 Payload 完整目錄、`.mod` 註冊與 Proxy Shape 驗證。
