# Rez 安裝與導入順序

本文檔說明 Rez 本體的安裝來源與安裝方式，以及整套系統從零開始的導入順序。後續 01 至 10 篇皆假設已完成本篇步驟。

---

## 一、安裝來源：直接使用官方 repo

* **來源**：官方 [`AcademySoftwareFoundation/rez`](https://github.com/AcademySoftwareFoundation/rez)，鎖定 tag **`3.4.0`**（支援 Python 3.8 至 3.13）。
* **不建議 fork**：工作室的客製化需求都能透過中央 `rezconfig.py` 與 package 定義解決，不需要修改 Rez 原始碼。只有在必須修補 Rez 本身、且上游尚未合併時才 fork，並以獨立 tag（例如 `3.4.0-studio.1`）區分。
* **離線安裝包**：將官方 Release 附帶的固定 Asset 下載一份放到中央 NAS，部署腳本會優先讀取 NAS，農場節點不需要連上 GitHub：
  ```text
  來源：https://github.com/AcademySoftwareFoundation/rez/releases/download/3.4.0/3.4.0.zip
  放置：X:/rez-system/installers/rez-3.4.0.zip（Linux：/mnt/x/rez-system/installers/rez-3.4.0.zip）
  ```
* **鎖定安裝包雜湊**：
  GitHub 自動產生的 tag source archive（`archive/refs/tags/...`）可能因後端壓縮演算法更新而改變檔案雜湊（見 [GitHub archive 穩定性說明](https://docs.github.com/en/repositories/working-with-files/using-files/downloading-source-code-archives)）。
  因此本架構採用 Release 附帶的固定 Asset（`releases/download/3.4.0/3.4.0.zip`），避免自動產生 archive 的壓縮格式變動。目前 SHA-256 由 [GitHub Release API](https://api.github.com/repos/AcademySoftwareFoundation/rez/releases/tags/3.4.0) 公布並由部署腳本鎖定：
  - **官方公布 SHA-256**：`898fbd182825009d5b293bac870d5ef507c97b14daccaa960c9b1328e069dd2e`
  兩支部署腳本皆已內建此固定雜湊，在解壓前進行嚴格比對。放上 NAS 後亦可執行指令核對：
  ```bash
  sha256sum /mnt/x/rez-system/installers/rez-3.4.0.zip                              # Linux
  Get-FileHash -Algorithm SHA256 X:\rez-system\installers\rez-3.4.0.zip           # Windows PowerShell
  ```
  雜湊應寫在受版本控管的腳本內，不要與 zip 一起放在 NAS 上，否則檔案被替換時雜湊也可能一併被替換。若上游資產遭替換，雜湊驗證會安全中止。
* **升級流程**：新版本先在 TD 機台安裝驗證，再同時更新部署腳本中的版本常數、安裝包雜湊與 NAS 上的安裝包。

---

## 二、三種安裝方式比較

| 方式 | 產出 | 適用情境 |
| :--- | :--- | :--- |
| **`install.py <DEST>`**（本文採用） | 隔離的 production install；CLI 位於 `<DEST>/bin/rez/`（Windows：`<DEST>\Scripts\rez\`） | 工作站、農場與 TD 機台的正式部署 |
| `pip install rez` | 一般 Python 套件；Rez 執行時會提示 `Pip-based rez installation detected` | 僅適合快速試用 |
| `install.py -p <REPO>` | 把 Rez 本身安裝成一個 rez 套件，只提供 API、不含 CLI | 自研工具需要在 Rez 環境中 `import rez` 時 |

> [!IMPORTANT]
> **為什麼正式部署一定要用 `install.py`**
>
> `install.py` 產生的 CLI 使用隔離的直譯器與 `sys.path`。進入 `rez-env` 後，即使套件（例如 HtoA）修改了 `PYTHONPATH`，同一個 shell 內執行的 `rez` 指令也不會受到影響。pip 安裝的 CLI 沒有這層隔離。

> [!NOTE]
> **`install.py -p` 的前置條件**
>
> `-p` 產生的套件會宣告 `[platform-*, arch-*, os-*, python-*]` 形式的 variant（例如 `platform-linux`、`arch-x86_64`、`os-Ubuntu-24.04`、`python-3.11`）。因此目標套件庫必須先綁定 `platform`、`arch`、`os`，以及 `python`（`rez-bind python`），否則解析時會找不到這些套件。一般部署不需要使用 `-p`。

---

## 三、導入順序（從零開始）

| 步驟 | 內容 | 執行者／機台 | 參考 |
| :--- | :--- | :--- | :--- |
| 1 | 建立 NAS 的 `config/`、`packages/`、`installers/` 目錄並設定 ACL | IT | [01 第二節](01-中央伺服器與全域配置.md) |
| 2 | 放置中央 `rezconfig.py` | Pipeline TD | [01 第五節](01-中央伺服器與全域配置.md) |
| 3 | 將 `rez-3.4.0.zip` 放入 `installers/`，並核對 SHA-256 與兩支部署腳本內建值一致 | Pipeline TD | 本篇第一節 |
| 4 | 在 TD 管理機安裝 Rez（本機共用安裝，需系統管理員權限） | Pipeline TD | Windows：[02 第一節](02-客戶端與算圖農場部署.md)；Linux：本篇第四節 |
| 5 | 驗證 Rez 與設定檔載入 | Pipeline TD | 本篇第五節 |
| 6 | 在 Windows 與 Linux 機台各執行一次 `rez-bind platform`、`arch`、`os` | Pipeline TD | [01 第一節](01-中央伺服器與全域配置.md) |
| 7 | 發布 DCC Wrapper 與外掛套件 | Pipeline TD | 03 至 10 |
| 8 | 部署美術工作站與算圖農場節點（每台電腦以系統管理員執行一次，所有帳號共用） | IT／Pipeline TD | [02](02-客戶端與算圖農場部署.md) |

> [!WARNING]
> **`rez-bind` 必須在各平台、以具寫入權限的帳號執行**
>
> `rez-bind platform` 綁定的是「執行當下這台機器」的平台，因此 `platform-windows` 必須在 Windows 上綁定，`platform-linux` 必須在 Linux 上綁定。依 01 的 ACL 規劃，農場節點對 `packages/` 只有唯讀權限，所以 Linux 端的綁定要在一台以 TD 帳號、讀寫掛載 NAS 的 Linux 機台上執行，不能直接用農場節點的一般帳號。

---

## 四、Linux 安裝（農場節點與 Linux TD 機台）

Windows 工作站與 Windows 農場節點請使用 [02](02-客戶端與算圖農場部署.md) 的 `setup_client.ps1`。Linux 端使用以下腳本，需以 root 或具有安裝目錄寫入權限的帳號執行：

```bash
#!/usr/bin/env bash
# setup_rez_linux.sh - Linux 農場節點／TD 機台 Rez 部署腳本（需 root 或安裝目錄寫入權限）
set -euo pipefail

REZ_VERSION="3.4.0"
# 官方 Release Asset 3.4.0.zip 的 SHA-256（取得方式見本篇第一節）
REZ_ARCHIVE_SHA256="${REZ_ARCHIVE_SHA256:-898fbd182825009d5b293bac870d5ef507c97b14daccaa960c9b1328e069dd2e}"
REZ_ROOT="${REZ_ROOT:-/mnt/x/rez-system}"
INSTALL_DIR="${INSTALL_DIR:-/opt/rez-client/venv}"
PYTHON="${PYTHON:-python3}"

if [[ ! "$REZ_ARCHIVE_SHA256" =~ ^[0-9a-fA-F]{64}$ ]]; then
    echo "錯誤：尚未設定 REZ_ARCHIVE_SHA256，請先填入 rez-$REZ_VERSION.zip 的 SHA-256" >&2
    exit 1
fi

# 1. 檢查 Python 版本（Rez 3.4.0 支援 3.8-3.13）與 venv 模組
"$PYTHON" -c 'import sys, venv, ensurepip; v = sys.version_info[:2]; sys.exit(0 if (3, 8) <= v < (3, 14) else f"Python {v[0]}.{v[1]} 不在 Rez 3.4.0 支援範圍 3.8-3.13")'

# 2. 優先使用 NAS 上的安裝包，找不到才從 GitHub 下載
tmp_dir="$(mktemp -d)"
trap 'rm -rf "$tmp_dir"' EXIT
archive="$tmp_dir/rez-$REZ_VERSION.zip"
nas_archive="$REZ_ROOT/installers/rez-$REZ_VERSION.zip"
if [[ -f "$nas_archive" ]]; then
    cp "$nas_archive" "$archive"
else
    echo ">>> NAS 找不到 $nas_archive，改從 GitHub 下載官方 Release Asset" >&2
    curl -fsSL -o "$archive" "https://github.com/AcademySoftwareFoundation/rez/releases/download/$REZ_VERSION/$REZ_VERSION.zip"
fi

# 解壓前驗證安裝包雜湊，避免 NAS 檔案損壞或被替換
echo "${REZ_ARCHIVE_SHA256,,}  $archive" | sha256sum -c --quiet - || {
    echo "錯誤：安裝包 SHA-256 不符，已中止安裝" >&2
    exit 1
}

# 3. 解壓並以官方 install.py 建立 production install（重複執行會覆蓋更新）
"$PYTHON" -m zipfile -e "$archive" "$tmp_dir"
"$PYTHON" "$tmp_dir/rez-$REZ_VERSION/install.py" -v "$INSTALL_DIR"

# 4. Smoke Test：比對安裝後的版本與鎖定版本
installed_ver="$("$INSTALL_DIR/bin/rez/rez" --version | grep -oE 'Rez [0-9]+\.[0-9]+\.[0-9]+' | cut -d' ' -f2 || true)"
if [[ "$installed_ver" != "$REZ_VERSION" ]]; then
    echo "錯誤：安裝後的 Rez 版本為 ${installed_ver:-未知}，與鎖定版本 $REZ_VERSION 不符" >&2
    exit 1
fi
echo ">>> 驗證成功！目前 Rez 版本: $installed_ver"
```

執行方式：

```bash
sudo ./setup_rez_linux.sh
# 自訂 NAS 根目錄、安裝位置或 Python：
sudo REZ_ROOT=/mnt/x/rez-system INSTALL_DIR=/opt/rez-client/venv PYTHON=/usr/bin/python3.11 ./setup_rez_linux.sh
# 也可以用環境變數傳入雜湊，不修改腳本（貼上第一節記錄的值，務必保留引號）：
sudo env REZ_ARCHIVE_SHA256='<貼上已記錄的 SHA-256>' ./setup_rez_linux.sh
```

> [!NOTE]
> Debian／Ubuntu 的系統 Python 可能未內建 `venv`，腳本第 1 步會直接失敗；請先安裝對應的 `python3-venv` 套件。

### 環境變數設定

* **互動式 shell**（TD 登入 Linux 機台操作）：寫入 `/etc/profile.d/rez.sh`：
  ```bash
  export PATH="/opt/rez-client/venv/bin/rez:$PATH"
  export REZ_CONFIG_FILE="/mnt/x/rez-system/config/rezconfig.py"
  export STUDIO_REZ_ROOT="/mnt/x/rez-system"
  ```
* **Deadline 等 systemd 服務**：systemd 不會讀取 `/etc/profile.d/`，必須在 unit override 中設定，包含 `PATH`，詳見 [02 第二節](02-客戶端與算圖農場部署.md)。

---

## 五、安裝後驗證

```bash
# 1. 確認 Rez 版本與安裝位置
rez --version

# 2. 確認中央設定檔有被載入（應列出 Rez 內建 rezconfig.py 與中央 rezconfig.py 兩行）
rez-config --source-list

# 3. 確認套件搜尋路徑指向中央庫
rez-config packages_path
```

`rez-config --source-list` 若只列出 Rez 內建的 `rezconfig.py`，代表 `REZ_CONFIG_FILE` 沒有生效，請檢查環境變數設定，並重新開啟終端機或重啟服務。
