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
│       ├── mayausd.mod                         # Autodesk 模組定義檔（官方實際產出為全小寫）
│       ├── mayausd/
│       │   ├── MayaUSD/                        # MayaUSD 外掛本體 (plugin, scripts)
│       │   └── USD/                            # Pixar USD 核心 dll, schemas, python
│       └── ...
│
└── platform-linux/
    └── maya-2024/                              # Linux + Maya 2024 巢狀子目錄
        ├── mayausd.mod
        ├── mayausd/
        │   ├── MayaUSD/
        │   └── USD/
        └── ...
```

### 3. 重要前置步驟：檢視與改寫 `mayausd.mod`（實體驗證分析）

實測檢視 Autodesk 官方安裝目錄（例如 `C:\Program Files\Autodesk\MayaUSD\Maya2027\0.36.0\mayausd.mod`）與原廠範本，揭示了真實的模組結構：

1. **檔名標準**：官方安裝產物統一為全小寫的 **`mayausd.mod`**。
2. **完整模組架構**：官方定義包含 `USD`、`MayaUSD_LIB`、`MayaUSD` 以及 `MAYAHYDRA` 四大模組段落。
3. **DLL 搜尋路徑已原生內建**：
   在 `mayausd.mod` 內部，官方已原生宣告了 Windows Python 3.8+ 所需的 DLL 尋址：
   ```text
   PXR_USD_WINDOWS_DLL_PATH+:=bin
   PXR_USD_WINDOWS_DLL_PATH+:=lib
   PXR_USD_WINDOWS_DLL_PATH+:=plugin/usd
   ```
   這證實了外掛在 Maya 啟動時會由模組系統內部自動配妥 DLL 路徑，**Rez 的 `package.py` 完全無需（亦不應）在全域環境重複宣告**。
4. **相對路徑重定位規則**：
   - 位於安裝根目錄（`MayaUSD/<MayaVer>/<Version>/`）下的 `mayausd.mod`，其 `+` 宣告預設即為相對路徑（例如 `+ USD 0.25.11 mayausd/USD`）。
   - 若由系統共享註冊目錄（`Common Files/Autodesk Shared/Modules/Maya/...`）取得的 `.mod`，其路徑已被展開為本機絕對路徑（`C:\Program Files\Autodesk\MayaUSD\...`）。
   - **關鍵語法規則**：`.mod` 內的 `icons:`、`plug-ins:`、`scripts:` 等子路徑，是**相對於各段 `+` 宣告的模組根目錄（Module Root Path），而非相對於 `.mod` 檔案本身**。因此若取得絕對路徑版本，**僅需將每個 `+` 開頭行末尾的絕對安裝根目錄置換為相對路徑（`./...` 或 `mayausd/...`），其餘所有宣告內容完全保持原樣**。

若需將絕對路徑的 `mayausd.mod` 轉為相對路徑，可執行以下 Python 腳本：

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

# 針對發布庫中的 mayausd.mod 進行相對路徑改寫
relativize_mod(
    Path("X:/rez-system/packages/maya_usd/0.28.0/platform-windows/maya-2024/mayausd.mod"),
    "C:/Program Files/Autodesk/MayaUSD/Maya2024/0.28.0",  # ← 填入原 .mod 內記錄的安裝根目錄
)
```

改寫完成後，每個模組根目錄皆相對於 `mayausd.mod` 所在目錄（`./mayausd/...`），無論掛載於何處或複製至本機快取目錄，Maya 皆能正確定位。

### 4. 模組載入優先序與雙重 Module 衝突防範（實機驗證）

Maya 啟動時會先搜尋 `MAYA_MODULE_PATH` 額外加入的路徑，再搜尋平台預設路徑。額外路徑依環境變數中的排列順序處理；為確保 Rez 套件優先，本文使用 `prepend()` 將 `{root}` 放在最前面。

Autodesk 文件列出的 Windows/Linux 預設路徑順序為：
1. 軟體安裝目錄模組資料夾（`<MAYA_LOCATION>/modules`）
2. 使用者版本目錄（Windows：`~/Documents/maya/<version>/modules`；Linux：`~/maya/<version>/modules`）
3. 使用者共用目錄（Windows：`~/Documents/maya/modules`；Linux：`~/maya/modules`）
4. 系統全域共享模組目錄（Windows：`C:/Program Files/Common Files/Autodesk Shared/Modules/maya/<version>`；Linux：`/usr/autodesk/modules/maya/<version>` 與 `/usr/autodesk/modules/maya`）

> [!IMPORTANT]
> **雙重 Module 優先權實測與部署規範**
>
> 若工作站或農場節點在安裝 Maya 時勾選了隨附的 MayaUSD，系統共享目錄（`Common Files`）便會存在一份本機的 `mayausd.mod`。此時若透過 Rez 載入環境，Maya 將同時面臨兩份同名模組宣告。
>
> **實機驗證（以 Maya 實測）**：
> - 當 Rez 注入 `env.MAYA_MODULE_PATH.prepend("{root}")` 時，Rez 路徑位於所有額外模組路徑與平台預設路徑之前，Maya 會優先載入 Rez 的模組。
> - 在本機保留共享 `Common Files/.../mayausd.mod` 的狀態下執行 `cmds.getModulePath(moduleName='MayaUSD')`，確認回傳的是 Rez 套件路徑而非本機路徑（即後述的 `assert 'Program Files' not in mod_path` 順利通過），證明 Rez 能覆寫內建模組。
>
> **生產線部署規範**：
> 雖然 Rez 在搜尋順序上具有優先權，但為避免工程混淆、杜絕環境變數未正確載入時意外回退至本機過期外掛，**強烈要求在安裝 Maya 時取消勾選隨附的 MayaUSD；若軟體已安裝，應由 IT/TD 腳本將系統共享目錄中的 `mayausd.mod` 移除或改名停用**。

---

## 二、完整 `package.py` 程式碼

在正式生產環境中，Maya-USD 自帶的 `mayausd.mod` 內部已經完整定義了該外掛專屬的 `PATH`、`PYTHONPATH`、`MAYA_PLUG_IN_PATH`、`PXR_PLUGINPATH_NAME` 以及 `PXR_USD_WINDOWS_DLL_PATH`。

因此在 Rez 中**最乾淨且防止重複註冊的做法是直接宣告 `MAYA_MODULE_PATH`，將環境變數控制權完全交由 Maya Module 機制**：

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
    # 單一來源授權：透過 Maya Module (.mod) 管理完整外掛路徑
    # 避免手動重複追加 MAYA_PLUG_IN_PATH 與 PYTHONPATH 導致雙重註冊或版本踩踏。
    # 由於 mayausd.mod 內部已原生宣告 PXR_USD_WINDOWS_DLL_PATH，
    # 外部 package.py 嚴禁手動宣告，徹底防止該變數洩漏污染其他 DCC (如 Houdini / Solaris)。
    env.MAYA_MODULE_PATH.prepend("{root}")
```

> [!IMPORTANT]
> **跨 DCC 零污染架構**
>
> 過去若在 `package.py` 內手動宣告全域 `PXR_USD_WINDOWS_DLL_PATH`，依據 OpenUSD 原始碼（`pxr/base/tf/__init__.py`），在 Windows 下 Python 會改由此變數指定的路徑呼叫 `os.add_dll_directory` 進行 DLL 目錄註冊，這主要影響 `import pxr` 的載入行為；若與 Houdini 混用環境，Houdini 內建的 USD 可能會被誤導載入 Maya-USD 的動態庫而引發崩潰或符號衝突。
> **透過實體驗證確認 `mayausd.mod` 內部已原生自帶該變數宣告**，將其完全收斂於 Maya 進程內部，外部環境變數維持極致乾淨，徹底消除了跨 DCC 的 DLL 衝突風險！

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

> [!IMPORTANT]
> **`mayapy` 呼叫順序規範：必須在 `maya.standalone.initialize()` 之後才 `import pxr`**
>
> 在 `mayapy` 批次環境中，`.mod` 檔案中所定義的各項環境變數（包含 `PATH`、`PYTHONPATH`、`PXR_USD_WINDOWS_DLL_PATH`）是在執行 `maya.standalone.initialize()` 時才會被 Maya 模組系統解析並套用進執行期環境。
> 若測試或生產腳本在 `initialize()` **之前**就嘗試 `import pxr` 或 `import mayaUsd`，在 Windows 下 Python 尚未獲得由 `.mod` 注入的 DLL 尋址路徑，將直接噴出 `ImportError: DLL load failed`。因此腳本中必須嚴格遵守「先初始化 Standalone，後引用外掛模組」的調用順序。

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
