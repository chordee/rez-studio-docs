---
tags:
  - dev
  - dev/pipeline
  - rez
  - dcc
  - wrapper
aliases:
  - Rez DCC Wrapper 規範與預設路徑
---
# DCC Wrapper 套件規範與預設安裝路徑對照

本文檔定義大型商業數位內容創作軟體（DCC：Houdini、Maya、Nuke）在 Rez 體系下的封裝哲學、預設路徑規格與 `package.py` 撰寫規範。

---

## 一、為什麼大型商業 DCC 要採用 Wrapper 模式？

在 Rez 管理的套件中，通常有兩種類型：
1. **自包含套件（Self-contained / Payload Package）**：
   套件的原始碼、外掛二進位檔直接存放在中央 NAS 套件目錄下（例如後述的 Arnold HtoA、Maya-USD）。
2. **包裝套件（Wrapper Package）**：
   套件目錄下**只有一個 `package.py`**，沒有任何龐大的軟體本體檔案。它的職責是作為「指標與轉譯器」，引導系統去尋找已經預先安裝在電腦本機的 DCC 商業軟體，並組裝該軟體所需的環境變數。

### 採用 Wrapper 的三大決定性理由：
* **巨大體積與網路頻寬限制**：Maya、Houdini、Nuke 動輒 5GB ~ 20GB。若全丟上 NAS，工作站與農場數十台機器開工時，NAS 網路瞬間癱瘓。
* **各作業系統底層依賴**：Windows 依賴 MSI / 註冊表與 Visual C++ Redistributable，Linux 依賴 RPM / 系統 GLIBC 與 X11 函式庫。直接安裝於本機最不易發生動態連結相依性遺漏。
* **版本精準鎖定**：即便軟體裝在本機，只要透過 Rez Wrapper，藝術家在不同專案中依然能精準調用指定版本。

---

## 二、三主流 DCC 預設安裝路徑對照矩陣

在我們的模擬情境中，工作機與農場所有節點均嚴格採用官方標準預設安裝路徑：

| 軟體名稱 | 版本範例 | Windows 官方預設安裝路徑 | Linux 官方預設安裝路徑 |
| :--- | :--- | :--- | :--- |
| **SideFX Houdini** | 20.5.278 | `C:/Program Files/Side Effects Software/Houdini 20.5.278` | `/opt/hfs20.5.278` |
| **Autodesk Maya** | 2024 | `C:/Program Files/Autodesk/Maya2024` | `/usr/autodesk/maya2024` |
| **Foundry Nuke** | 15.1.1 (15.1v1) | `C:/Program Files/Nuke15.1v1` | `/usr/local/Nuke15.1v1` |

---

## 三、跨平台 Wrapper 撰寫六大金律

在撰寫 Wrapper 的 `package.py` 時，必須嚴格遵守以下標準：

### 1. 善用 `system.platform` 判斷作業系統
在 Rez `commands()` 區塊內，透過內建屬性 `system.platform` 動態得知當前執行環境是 `"windows"` 還是 `"linux"`。

### 2. 路徑分隔符一律使用正斜線 `/`
Python 與 Rez 在 Windows 上處理正斜線 `/` 均十分穩定，避免使用反斜線 `\` 導致跳脫字元錯誤。

### 3. 執行期檢驗路徑（Fail Fast）
如果本機根本沒有安裝該版本的 DCC，Wrapper 應呼叫 `stop()` 發出清晰的警示並中斷執行：
```python
import os
if not os.path.exists(dcc_root):
    stop(f"找不到指定的軟體安裝路徑: {dcc_root}")
```
> [!note] `stop()` 的觸發時機
> `commands()` 區塊是在**進入環境或執行指令時**才被直譯器執行。在中央派遣器使用 `rez-env -o job.rxt` 烘焙 Context 時並不會執行 `commands()`，因此派遣機本機未安裝 DCC 依然能成功計算相依性並導出 Context。`stop()` 會在農場節點實際載入環境時守門。

### 4. 嚴格定義根變數與二進位執行路徑
- **Houdini**：必須注入 `HFS` 與 `PATH.prepend("{hfs}/bin")`。
- **Maya**：必須注入 `MAYA_LOCATION` 與 `PATH.prepend("{maya}/bin")`。
- **Nuke**：必須將安裝根目錄加入 `PATH`。

### 5. 避免在 Linux 上為 DCC 設定全域 `LD_LIBRARY_PATH`
> [!caution] 依靠 RPATH 尋址，勿濫用 LD_LIBRARY_PATH
> Houdini、Maya、Nuke 原廠二進位檔皆已內嵌正確的 ELF `RPATH` / `RUNPATH`，軟體啟動時會優先讀取自帶的 .so。若在 Wrapper 中對全域 `LD_LIBRARY_PATH` 進行 prepend，極易導致同環境中混用的其他軟體或外部工具發生 GCC / Qt / C++ Runtime 符號版本衝突（如 Segfault）。

### 6. 環境變數追加無需預先初始化為空字串
在 Rez Rex API 中，當對一個尚未被賦值的環境變數直接執行 `append()` 或 `prepend()` 時，Rez 會自動將其直接設為該值。**切勿預先寫 `env.VAR = ""`**，否則路徑清單開頭會多出一個空字串（在 POSIX 下表現為多餘的開頭冒號 `:/path`，代表搜尋當前工作目錄，存在安全與路徑解析風險）。

---

## 四、中央儲存庫目錄放置方式

所有 Wrapper 套件皆統一存放在中央 NAS 的 `packages/` 目錄：

```text
X:/rez-system/packages/ (Linux: /mnt/x/rez-system/packages/)
├── houdini/
│   └── 20.5.278/
│       └── package.py
├── maya/
│   └── 2024/
│       └── package.py
└── nuke/
    └── 15.1.1/
        └── package.py
```
