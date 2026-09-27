# Houdini Wrapper 套件跨平台實作指南

本文檔提供 SideFX Houdini 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點，並包含農場級的真實環境隔離機制。

---

## 一、檔案存放位置

放置於中央網路儲存庫：

- `X:/rez-system/packages/houdini/20.5.278/package.py`（Linux: `/mnt/x/rez-system/packages/houdini/20.5.278/package.py`）
- `X:/rez-system/packages/houdini/22.0.368/package.py`（Linux: `/mnt/x/rez-system/packages/houdini/22.0.368/package.py`）

以下以 20.5.278 為完整範例；22.0.368 使用相同內容，只需將 `version` 改為 `"22.0.368"`。安裝路徑會由 `this.version` 動態組成。

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
    # 避免外層環境變數導致判斷跳過，使後續外掛 prepend 覆寫丟失末端 "&"
    env.HOUDINI_PATH = "&"

    # 5. 生產環境嚴格隔離防護
    # (A) 禁止讀取使用者的本機 houdini.env 檔案
    env.HOUDINI_NO_ENV_FILE = 1

    # (B) 條件式農場隔離策略：僅在算圖節點重定向 HOUDINI_USER_PREF_DIR
    # 注意：HOUDINI_PACKAGE_DIR 只是追加目錄，Houdini 預設依然會掃描 $HOUDINI_USER_PREF_DIR/packages。
    # 透過父環境的 STUDIO_FARM_NODE 變數識別農場節點（defined() 可正確讀取父環境）：
    # - 美術工作站：維持預設家目錄（保留使用者個人的 Desktop、Shelf、Hotkey 等偏好設定）
    # - 算圖農場：導向部署時建立、僅 Worker 可寫入的目錄（見 02 第二節），避免本機 ~/houdiniX.X/packages 被帶入農場
    if defined("STUDIO_FARM_NODE"):
        prefs_root = "/var/cache/farm-prefs" if system.platform == "linux" else "C:/farm-prefs"
        env.HOUDINI_USER_PREF_DIR = f"{prefs_root}/houdini/__HVER__"
```

---

## 三、環境變數與架構設計說明

1. **`HFS` 核心指標**：
   Houdini 所有內部工具、PySide 封裝及 HOM（Houdini Object Model）皆高度依賴 `HFS`。只要 `HFS` 與 `PATH` 設定完成，執行 `hython` 時系統會自動找到其內建的 Python 模組，**無須手動將內嵌 Python site-packages 強行塞入全域 `PYTHONPATH`**。
2. **`HOUDINI_PATH = "&"` 無條件初始化**：
   末端的 `&` 代表保留 Houdini 原生路徑；後續的外掛（如 Arnold HtoA）在載入時執行 `env.HOUDINI_PATH.prepend("{root}")`，即可形成 `<htoa_root>;&` 的完美搜尋順序。
3. **`HOUDINI_USER_PREF_DIR` 條件式農場隔離機制**：
   SideFX 官方規格中，`HOUDINI_PACKAGE_DIR` 是**額外追加**搜尋目錄，無法阻止 Houdini 去讀取使用者的 `~/houdini20.5/packages/*.json`。
   若無條件覆寫 `HOUDINI_USER_PREF_DIR`，會導致美術工作站每次啟動都被導向空目錄而遺失個人工具架與熱鍵，且在 Windows 上會造成多使用者共用同一暫存目錄。
   因此透過 `if defined("STUDIO_FARM_NODE"):` 進行條件式隔離（`defined()` 可正確讀取父環境變數）：
   - **工作站（美術）**：維持預設路徑，確保個人操作習慣與設定不被破壞。
   - **算圖農場（Worker）**：由農場環境變數宣告 `STUDIO_FARM_NODE=1`，將偏好目錄重定向至 `/var/cache/farm-prefs/houdini/__HVER__` 或 `C:/farm-prefs/houdini/__HVER__`，讓農場上的 `&` 不會帶入使用者家目錄中的本機測試外掛。
     - **根目錄由部署建立並收緊權限**：Windows 由 `setup_client.ps1 -FarmNode` 建立 `C:\farm-prefs`，Linux 由 systemd `CacheDirectory=` 建立 `/var/cache/farm-prefs`；只有 SYSTEM／root、系統管理員與 Worker 執行身分可寫入（見 [02 第二節](02-客戶端與算圖農場部署.md)）。不要改用 `C:\temp` 或 `/var/tmp` 這類任何帳號都能預先建立內容的位置。
     - **版本子目錄**：`houdini/__HVER__` 是否由 Houdini 首次啟動時自動建立，尚待農場實機確認；若未自動建立，部署時請預先建立對應版本的子目錄（例如 `houdini/20.5`）。
     - **限制**：同一節點上同版本的任務會共用這個目錄；需要逐任務隔離時，由派送端另行處理。
4. **移除無效的 `dsolib` 與 `LD_LIBRARY_PATH`**：
   - Windows 的 `custom/houdini/dsolib` 放的是 HDK 編譯專用的 `.lib` 靜態連結檔，非執行期 DLL。
   - Linux 版 Houdini 主程式自帶 ELF `RPATH`，無需手動 prepend `LD_LIBRARY_PATH`。

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

第二個版本使用相同方式驗證：

```bash
rez-env houdini-22.0.368 -- hython -c "import hou; print('Houdini Version:', hou.applicationVersionString())"
```

預期輸出：

```text
Houdini Version: 22.0.368
```

### 2. 農場渲染命令驗證（Karma 渲染測試）
```bash
rez-env houdini-20.5.278 -- hrender -e -f 1 10 -d karma1 /mnt/x/jobs/shot010/fx.hip
```
