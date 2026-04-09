---
name: install
description: 從 Team Marketplace 安裝 plugin（先試內建安裝，失敗則用 gh CLI fallback，再失敗則排查環境）
---

# /cm-ailab-mp:install — 安裝 Plugin

你是 Team Marketplace 的 plugin 安裝助手。安裝流程採用 **逐層 fallback** 策略：

1. **先試內建安裝**（`/plugin install`）— 最乾淨，由 Claude Code 原生管理
2. **失敗則用 `gh repo clone` 手動安裝** — 繞過 SSH 問題，用 gh token 認證
3. **再失敗則排查環境** — 診斷 SSH / gh / git 狀態並修復

## 參數

- **必要**：`<name>` — 要安裝的 plugin 名稱

如果使用者沒有提供名稱，提示：

```
請提供要安裝的 plugin 名稱，例如：/cm-ailab-mp:install ops-jira

查看所有可用 plugin：/cm-ailab-mp:list
```

## 步驟 1：前置檢查

### 1.1 確認 gh 已登入

```bash
gh auth status 2>&1 | head -5
```

如果未登入，提示使用者執行 `gh auth login`。

### 1.2 查詢 plugin 資訊

從 marketplace.json 取得目標 plugin 的完整資訊：

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d
```

解析 JSON，找到 `plugins` 陣列中 `name` 等於目標名稱的條目。需要取得：

- `source.repo` — GitHub repo 路徑（例如 `cm-ailab-cc-plugins/plugin-ops-jira`）
- `source.ref` — Git tag（例如 `v1.0.0`）
- `version` — 版本號

如果找不到，提示：

```
找不到 plugin "<name>"。

查看所有可用 plugin：/cm-ailab-mp:list
```

### 1.3 檢查是否已安裝

```bash
cat ~/.claude/plugins/installed_plugins.json 2>/dev/null | jq -r --arg id "<name>@cm-ailab-cc-plugins" '.plugins[$id] // empty'
```

如果已存在，提示：

```
plugin "<name>" 已經安裝。

- 如果要更新版本：/cm-ailab-mp:update <name>
- 如果要重新安裝：請先回答 "繼續" 以覆蓋安裝
```

等待使用者確認是否覆蓋。

## 步驟 2：選擇安裝 scope

詢問使用者安裝範圍：

```
請選擇安裝範圍：

| # | Scope | 說明 |
|---|-------|------|
| 1 | user | 全域安裝 — 所有專案都能使用 |
| 2 | project | 專案安裝 — 僅在目前專案目錄中使用 |

選擇 (1/2):
```

- 選擇 1 → `scope = "user"`
- 選擇 2 → `scope = "project"`，並記錄 `projectPath` 為目前工作目錄

## 步驟 3：嘗試安裝（逐層 fallback）

### 3.1 第一層：內建 `/plugin install`

告訴使用者先嘗試內建安裝：

```
請先嘗試內建安裝：

/plugin install <name>@cm-ailab-cc-plugins

安裝完成後告訴我結果。如果失敗了，把錯誤訊息貼給我，我會用其他方式安裝。
```

等待使用者回報結果：

- **成功** → 跳到步驟 5（顯示結果），不需要步驟 4
- **失敗** → 繼續 3.2

### 3.2 第二層：`gh repo clone` 手動安裝

內建安裝失敗時，改用 `gh repo clone`：

```bash
# 定義路徑
CACHE_DIR="$HOME/.claude/plugins/cache/cm-ailab-cc-plugins/<name>/<version>"

# 確保父目錄存在
mkdir -p "$(dirname "$CACHE_DIR")"

# 如果目標已存在，先移除
rm -rf "$CACHE_DIR"

# Clone（gh 自動用 active 帳號的 token，走 HTTPS）
gh repo clone <source.repo> "$CACHE_DIR" -- --depth 1 --branch <source.ref>
```

- **成功** → 取得 SHA、清理 .git、繼續步驟 4 註冊
- **失敗** → 繼續 3.3

#### 取得 git commit SHA

```bash
cd "$CACHE_DIR" && git rev-parse HEAD
```

#### 清理 .git 目錄

```bash
rm -rf "$CACHE_DIR/.git"
```

### 3.3 第三層：環境排查

兩層都失敗時，執行完整環境診斷：

```bash
# 1. gh 登入狀態與 active 帳號
gh auth status 2>&1

# 2. SSH 連線測試
ssh -T git@github.com 2>&1

# 3. git protocol 設定
gh config get git_protocol 2>&1

# 4. 組織成員資格（用每個 gh 帳號嘗試）
gh api /user/memberships/orgs/cm-ailab-cc-plugins --jq '.state' 2>/dev/null

# 5. 目標 repo 是否存在且可存取
gh api repos/<source.repo> --jq '.full_name' 2>&1

# 6. 嘗試直接 HTTPS clone（繞過 gh）
git clone --depth 1 --branch <source.ref> https://github.com/<source.repo>.git /tmp/test-plugin-clone 2>&1
```

根據診斷結果，針對性修復：

| 診斷結果 | 修復方式 |
|----------|----------|
| SSH 認證到錯誤帳號 | 建議 `gh auth switch` 或修改 `~/.ssh/config` |
| gh 的 active 帳號不在 org 裡 | `gh auth switch --user <有權限的帳號>` |
| repo 不存在 | 確認 marketplace.json 中的 repo 路徑 |
| HTTPS clone 成功但 gh clone 失敗 | 用 HTTPS 結果繼續安裝 |
| 完全無法存取 | 提示聯繫管理員確認 org 成員資格和 repo 權限 |

修復後重試 clone。

## 步驟 4：註冊 Plugin

（只有第二層 `gh repo clone` 安裝時需要，內建安裝會自動處理）

### 4.1 更新 installed_plugins.json

讀取 `~/.claude/plugins/installed_plugins.json`，在 `plugins` 物件中新增或更新條目：

**Key**: `<name>@cm-ailab-cc-plugins`

**Value**（user scope）:
```json
[{
  "scope": "user",
  "installPath": "<CACHE_DIR 的完整路徑>",
  "version": "<version>",
  "installedAt": "<ISO 8601 timestamp>",
  "lastUpdated": "<ISO 8601 timestamp>",
  "gitCommitSha": "<git SHA>"
}]
```

**Value**（project scope）:
```json
[{
  "scope": "project",
  "projectPath": "<目前工作目錄的完整路徑>",
  "installPath": "<CACHE_DIR 的完整路徑>",
  "version": "<version>",
  "installedAt": "<ISO 8601 timestamp>",
  "lastUpdated": "<ISO 8601 timestamp>",
  "gitCommitSha": "<git SHA>"
}]
```

使用 jq 更新：

```bash
PLUGIN_ID="<name>@cm-ailab-cc-plugins"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")

# user scope
jq --arg id "$PLUGIN_ID" \
   --arg path "$CACHE_DIR" \
   --arg ver "<version>" \
   --arg ts "$TIMESTAMP" \
   --arg sha "<git SHA>" \
   '.plugins[$id] = [{
     scope: "user",
     installPath: $path,
     version: $ver,
     installedAt: $ts,
     lastUpdated: $ts,
     gitCommitSha: $sha
   }]' \
   ~/.claude/plugins/installed_plugins.json > /tmp/installed_plugins_updated.json

mv /tmp/installed_plugins_updated.json ~/.claude/plugins/installed_plugins.json
```

project scope 的版本加上 `projectPath` 欄位。

### 4.2 更新 settings.json

根據 scope 決定更新哪個 settings.json：

- **user scope** → `~/.claude/settings.json`
- **project scope** → `<projectPath>/.claude/settings.json`

在 `enabledPlugins` 中新增條目：

```bash
SETTINGS_FILE="<對應的 settings.json 路徑>"

# 確保 settings.json 存在
if [ ! -f "$SETTINGS_FILE" ]; then
  echo '{}' > "$SETTINGS_FILE"
fi

jq --arg id "$PLUGIN_ID" \
   '.enabledPlugins[$id] = true' \
   "$SETTINGS_FILE" > /tmp/settings_updated.json

mv /tmp/settings_updated.json "$SETTINGS_FILE"
```

## 步驟 5：顯示結果

```
╔══════════════════════════════════════════╗
║       ✓ Plugin 安裝成功！               ║
╠══════════════════════════════════════════╣
║ Plugin:  <name> v<version>              ║
║ Scope:   <user 或 project>              ║
║ 方式:    <內建安裝 或 gh clone>         ║
╠══════════════════════════════════════════╣
║ 下一步:                                 ║
║ 執行 /reload-plugins 以啟用 plugin      ║
╚══════════════════════════════════════════╝
```

## 注意事項

- 優先使用內建 `/plugin install`，它由 Claude Code 原生管理，更新和解除安裝更方便
- 只有內建安裝失敗時才 fallback 到 `gh repo clone`
- `gh repo clone` 安裝時，所有 git 操作用 `gh`，不要用 `git clone`，確保走 gh 的認證
- 不要修改使用者的 SSH config 或 git config
- 安裝過程中遇到任何問題，優先自己排查修復，不要直接丟錯誤給使用者
- 所有輸出使用繁體中文
