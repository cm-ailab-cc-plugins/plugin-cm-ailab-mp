---
name: install
description: 從 Team Marketplace 安裝 plugin（使用 gh CLI 認證，自動排查問題）
---

# /cm-ailab-mp:install — 安裝 Plugin

你是 Team Marketplace 的 plugin 安裝助手。使用 `gh` CLI 來 clone 和安裝 plugin，繞過 Claude Code 內建 plugin 系統的 SSH 限制。

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

### 1.2 確認組織成員資格

```bash
gh api /user/memberships/orgs/cm-ailab-cc-plugins --jq '.state' 2>/dev/null
```

如果不是 `active`，提示使用者聯繫管理員加入組織。

### 1.3 查詢 plugin 資訊

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

### 1.4 檢查是否已安裝

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

## 步驟 3：Clone Plugin

使用 `gh repo clone` clone 到 cache 目錄：

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

如果 clone 失敗，Claude 應該：

1. 讀取錯誤訊息
2. 嘗試診斷原因（權限？網路？repo 不存在？）
3. 如果是帳號問題，嘗試 `gh auth switch` 到有權限的帳號
4. 重試 clone

### 取得 git commit SHA

```bash
cd "$CACHE_DIR" && git rev-parse HEAD
```

記錄這個 SHA 供後續寫入 installed_plugins.json。

### 清理 .git 目錄

```bash
rm -rf "$CACHE_DIR/.git"
```

## 步驟 4：註冊 Plugin

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
║ 路徑:    <CACHE_DIR>                    ║
╠══════════════════════════════════════════╣
║ 下一步:                                 ║
║ 執行 /reload-plugins 以啟用 plugin      ║
╚══════════════════════════════════════════╝
```

## 錯誤處理

每一步失敗時，Claude 應該主動排查而不是直接報錯：

- **gh 未登入** → 提示 `gh auth login`
- **不是組織成員** → 提示聯繫管理員
- **clone 失敗（權限）** → 嘗試 `gh auth switch` 到其他帳號重試
- **clone 失敗（repo 不存在）** → 確認 marketplace.json 中的 repo 路徑是否正確
- **jq 未安裝** → 用 python3 替代處理 JSON
- **settings.json 寫入失敗** → 檢查檔案權限

## 注意事項

- 所有 git 操作使用 `gh repo clone`，不要用 `git clone`，確保走 gh 的認證
- 不要修改使用者的 SSH config 或 git config
- 安裝過程中遇到任何問題，優先自己排查修復，不要直接丟錯誤給使用者
- 所有輸出使用繁體中文
