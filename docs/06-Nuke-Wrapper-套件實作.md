# Nuke Wrapper 套件跨平台實作指南

本文檔提供 Foundry Nuke 在混合作業系統下的標準 Wrapper `package.py` 實作程式碼，支援 Windows 藝術家工作站與 Linux/Windows 算圖農場節點，並徹底解決 Windows 下別名失效的問題。

---

## 一、檔案存放位置與 Shim 架構

### 1. 套件目錄樹
由於 Windows cmd 下的 `alias()` 是透過 `doskey` 實作，在非互動模式（批次檔、Deadline 算圖 Worker）下會完全失效；而在 PowerShell 中亦有引號路徑解析限制。
**最佳生產級解法是在 Wrapper 內建 `bin/*.cmd` 實體 Shim 腳本**：

```text
X:/rez-system/packages/nuke/ (Linux: /mnt/x/rez-system/packages/nuke/)
├── 15.1.1/
│   ├── package.py
│   └── bin/
│       ├── nuke.cmd             # Windows 指令 Shim
│       ├── nukex.cmd
│       └── nukestudio.cmd
└── 17.1.1/
    ├── package.py
    └── bin/
        ├── nuke.cmd
        ├── nukex.cmd
        └── nukestudio.cmd
```

### 2. Shim 腳本內容範例

**`bin/nuke.cmd`**：
```cmd
@echo off
@"%NUKE_LOCATION%\Nuke15.1.exe" %*
```

**`bin/nukex.cmd`**：
```cmd
@echo off
@"%NUKE_LOCATION%\Nuke15.1.exe" --nukex %*
```

**`bin/nukestudio.cmd`**：
```cmd
@echo off
@"%NUKE_LOCATION%\Nuke15.1.exe" --studio %*
```

17.1.1 目錄內的三個 Shim 結構相同，但執行檔名稱必須改為 `Nuke17.1.exe`；`nukex.cmd` 與 `nukestudio.cmd` 分別保留 `--nukex`、`--studio` 參數。

---

## 二、完整 `package.py` 程式碼

```python
# -*- coding: utf-8 -*-
name = "nuke"

# 版本命名採用標準 15.1.1（對應 Foundry 官方發行號 15.1v1）
# 避免 "15.1v1" 在 Rez 內部被拆成 [15, "1v1"] 導致 "nuke-15.1" 無法匹配
version = "15.1.1"

description = "Foundry Nuke DCC Wrapper Package (Default Installation)"

authors = ["Studio Pipeline Team"]

# Wrapper 僅為指標，禁止被本機快取系統複製
cachable = False

tools = [
    "nuke",
    "nukex",
    "nukestudio"
]

def commands():
    import os
    
    # 1. 由版本號 15.1.1 動態組裝 Foundry 官方安裝目錄命名 "15.1v1"
    # ver[0]=15, ver[1]=1, ver[2]=1 -> "15.1v1"
    v = this.version
    foundry_ver = f"{v[0]}.{v[1]}v{v[2]}"

    # 2. 依據作業系統解析官方預設安裝路徑
    if system.platform == "windows":
        nuke_root = f"C:/Program Files/Nuke{foundry_ver}"
    elif system.platform == "linux":
        nuke_root = f"/usr/local/Nuke{foundry_ver}"
    else:
        stop(f"不支援的作業系統架構: {system.platform}")

    # 3. 檢驗安裝實體是否存在（Fail Fast）
    if not os.path.exists(nuke_root):
        stop(
            f"[Rez Wrapper] 找不到 Foundry Nuke 安裝目錄！\n"
            f"預期路徑: {nuke_root}\n"
            f"請確認該機器是否已將 Nuke 安裝在官方預設路徑。"
        )

    # 4. 核心環境變數配置
    env.NUKE_LOCATION = nuke_root
    
    # 將 Nuke 官方根目錄加入 PATH (提供 Nuke15.1.exe / Nuke15.1 原生二進位)
    env.PATH.prepend(nuke_root)

    # 5. 指令封裝（解決 Windows 批次/農場呼叫問題）
    if system.platform == "windows":
        # 在 Windows 上將套件內建的 bin/ 目錄置頂，
        # 提供實體 nuke.cmd / nukex.cmd，無論在 cmd、PowerShell 或非互動批次檔中皆能穩定執行
        env.PATH.prepend("{root}/bin")
    else:
        # 在 Linux 上建立指向原生執行檔的 alias
        bin_base = f"Nuke{v[0]}.{v[1]}"
        alias("nuke", f'"{nuke_root}/{bin_base}"')
        alias("nukex", f'"{nuke_root}/{bin_base}" --nukex')
        alias("nukestudio", f'"{nuke_root}/{bin_base}" --studio')
```

`17.1.1/package.py` 使用相同內容，只需將 `version` 改為 `"17.1.1"`。通用轉換邏輯會將 Rez 版本組成 Foundry 發行號 `17.1v1`；Windows Shim 則仍須使用該版本對應的 `Nuke17.1.exe`。

---

## 三、環境變數與農場算圖陷阱

1. **捨棄 `alias()`，改用實體 Shim 腳本**：
   Rez 在 Windows cmd 環境下是透過 `doskey` 實作 alias，但在非互動式命令列（如 Deadline 算圖任務、CI 腳本）中 `doskey` 會完全失效；而在 PowerShell 中帶引號的路徑函式亦有語法陷阱。透過在 Wrapper 中內建 `bin/nuke.cmd`，完美支援任何外層 Shell 與農場調用。
2. **Nuke 命令列參數 `-c` 誤區**：
   > [!WARNING]
   > **Nuke 的 `-c` 不是執行 Python 字串**
   >
   > 與標準 Python 的 `python -c "print(1)"` 不同，Nuke CLI 的 `-c <size>` 代表**設定快取記憶體大小（Cache Size）**。若要透過命令列執行 Python 腳本測試，必須建立一個實體 `.py` 檔案並以 `nuke -t test_script.py` 執行。
3. **無須手動設定 `LD_LIBRARY_PATH`**：
   Nuke 原廠二進位自帶完整的 RPATH 尋址機制，避免手動 prepend `LD_LIBRARY_PATH` 造成與外部工具的動態函式庫衝突。

---

## 四、驗證測試指令

### 1. 終端機無介面（Terminal 模式）測試
建立一個快速測試腳本並執行：

```powershell
# 建立測試腳本
Set-Content -Path "$env:TEMP\test_nuke.py" -Value "import nuke; print('Nuke 核心載入成功！版本:', nuke.NUKE_VERSION_STRING)"

# 透過 rez-env 呼叫 Nuke Terminal 模式執行 (使用 nuke.cmd shim)
rez-env nuke-15.1.1 -- nuke -t "$env:TEMP\test_nuke.py"
```

預期輸出：
```text
Nuke 核心載入成功！版本: 15.1v1
```

第二個版本使用相同測試腳本驗證：

```powershell
rez-env nuke-17.1.1 -- nuke -t "$env:TEMP\test_nuke.py"
```

預期輸出：

```text
Nuke 核心載入成功！版本: 17.1v1
```

### 2. 農場渲染命令驗證
```bash
rez-env nuke-15.1.1 -- nuke -x -F 1-100 /mnt/x/jobs/comp/shot010_comp.nk
```
