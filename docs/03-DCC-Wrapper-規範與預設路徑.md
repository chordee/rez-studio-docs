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

| 軟體名稱 | Rez 套件版本 | Windows 官方預設安裝路徑 | Linux 官方預設安裝路徑 |
| :--- | :--- | :--- | :--- |
| **SideFX Houdini** | 20.5.278 | `C:/Program Files/Side Effects Software/Houdini 20.5.278` | `/opt/hfs20.5.278` |
| **SideFX Houdini** | 22.0.368 | `C:/Program Files/Side Effects Software/Houdini 22.0.368` | `/opt/hfs22.0.368` |
| **Autodesk Maya** | 2024 | `C:/Program Files/Autodesk/Maya2024` | `/usr/autodesk/maya2024` |
| **Autodesk Maya** | 2027 | `C:/Program Files/Autodesk/Maya2027` | `/usr/autodesk/maya2027` |
| **Foundry Nuke** | 15.1.1 (對應 15.1v1) | `C:/Program Files/Nuke15.1v1` | `/usr/local/Nuke15.1v1` |
| **Foundry Nuke** | 17.1.1 (對應 17.1v1) | `C:/Program Files/Nuke17.1v1` | `/usr/local/Nuke17.1v1` |

> [!IMPORTANT]
> **Maya 本機安裝與隨附外掛部署規範（防範 Module 雙重衝突）**
>
> Maya 預設的模組搜尋路徑包含系統共享目錄（Windows 為 `C:/Program Files/Common Files/Autodesk Shared/Modules/Maya/<ver>`）。
> 若在工作站或農場節點安裝 Maya 時一併勾選了隨附的 MayaUSD 外掛，該共享目錄下會預先產生 `mayausd.mod`，導致 Maya 啟動時同時看見「Rez 套件庫」與「本機安裝」兩份同名模組。
> 為確保環境純淨並避免版本混淆，**部署規範明確要求：安裝 Maya 時取消勾選內建的 MayaUSD，或在安裝後自本機共享 Modules 目錄中移除 `mayausd.mod`**（詳見 [10-Maya-USD外掛套件實作](10-Maya-USD外掛套件實作.md)）。

---

## 三、跨平台 Wrapper 撰寫七大金律

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
> [!NOTE]
> **`stop()` 的觸發時機**
>
> `commands()` 區塊是在**進入環境或執行指令時**才被直譯器執行。在中央派遣器使用 `rez-env -o job.rxt` 烘焙 Context 時並不會執行 `commands()`，因此派遣機本機未安裝 DCC 依然能成功計算相依性並導出 Context。`stop()` 會在農場節點實際載入環境時守門。

### 4. 嚴格定義根變數與二進位執行路徑
- **Houdini**：必須注入 `HFS` 與 `PATH.prepend("{hfs}/bin")`。
- **Maya**：必須注入 `MAYA_LOCATION` 與 `PATH.prepend("{maya}/bin")`。
- **Nuke**：必須將安裝根目錄加入 `PATH`，並為 Windows 提供 `bin/*.cmd` 實體 shim。

### 5. 理解 Rez 變數初次賦值的覆寫行為
> [!IMPORTANT]
> **變數覆寫機制與 HOUDINI_PATH**
>
> 在 Rez 體系中，**當一個 Context 第一次對某環境變數執行操作時，即使使用者的父環境中已存在該變數，Rez 也會直接將其覆寫**（除非該變數已在全域設定檔的 `parent_variables` 中聲明繼承）。
> 這也是為什麼在 Houdini Wrapper 中，**絕對不能寫 `if not defined("HOUDINI_PATH")`**；若判斷為 True 而跳過，後續外掛（如 HtoA）的 `prepend` 會被視為首次操作而直接覆寫全域，導致末端的 `&` 消失、原生節點全滅。因此基礎變數必須由宿主 Wrapper 無條件指定預設值。

### 6. 切勿將 DCC 內嵌 Python 任意注入全域 `PYTHONPATH`
> [!CAUTION]
> **避免全域 Python 環境污染**
>
> Maya、Houdini 各自攜帶特定編譯旗標與客製化版本的 Python（例如 Maya 帶有專屬的 `site-packages`）。若將 DCC 的 Python 目錄無條件加入全域 `PYTHONPATH`，當使用者在環境中同時執行其他系統工具時，極易引發二進位 ABI 不相容而崩潰（如 Segmentation Fault 或 DLL Load Failed）。DCC 內部模組應盡量限定在 DCC 自身啟動時由其內部環境讀取。

### 7. 依靠 RPATH 尋址，勿在 Linux 濫用 `LD_LIBRARY_PATH`
> [!CAUTION]
> **避免動態函式庫符號踩踏**
>
> Houdini、Maya、Nuke 原廠二進位檔皆已內嵌正確的 ELF `RPATH` / `RUNPATH`，軟體啟動時會優先讀取自帶的 .so。若在 Wrapper 中對全域 `LD_LIBRARY_PATH` 進行 prepend，極易導致同環境中混用的其他軟體或外部工具發生 GCC / Qt / C++ Runtime 符號版本衝突。

---

## 四、中央儲存庫目錄放置方式

所有 Wrapper 套件皆統一存放在中央 NAS 的 `packages/` 目錄：

```text
X:/rez-system/packages/ (Linux: /mnt/x/rez-system/packages/)
├── houdini/
│   ├── 20.5.278/
│   │   └── package.py
│   └── 22.0.368/
│       └── package.py
├── maya/
│   ├── 2024/
│   │   └── package.py
│   └── 2027/
│       └── package.py
└── nuke/
    ├── 15.1.1/
    │   ├── package.py
    │   └── bin/
    │       ├── nuke.cmd
    │       ├── nukex.cmd
    │       └── nukestudio.cmd
    └── 17.1.1/
        ├── package.py
        └── bin/
            ├── nuke.cmd
            ├── nukex.cmd
            └── nukestudio.cmd
```
