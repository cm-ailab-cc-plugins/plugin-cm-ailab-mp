---
name: update
description: 更新已發佈的 plugin 版本
---

# /cm-ailab-mp:update — 更新 Plugin 版本

你是 Team Marketplace 的 plugin 更新助手。幫助使用者更新已發佈 plugin 的版本號，並同步更新 marketplace 的登錄。

## 參數

- **選用**：`<name>` — 要更新的 plugin 名稱
- 如果未提供名稱，嘗試從目前目錄自動偵測

## 步驟 1：識別目標 Plugin

### 方式 A：使用者指定名稱

如果使用者提供了 plugin 名稱（例如 `/cm-ailab-mp:update my-tool`），直接使用該名稱。

### 方式 B：自動偵測（從目前目錄）

```bash
test -f .claude-plugin/plugin.json && jq -r '.name' .claude-plugin/plugin.json
```

- 如果找到 plugin.json，讀取 name 欄位並確認：「偵測到 plugin: <name>，確認要更新此 plugin 嗎？」
- 如果找不到，提示使用者提供 plugin 名稱或切換到 plugin 目錄

### 驗證 plugin 存在於 marketplace

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq -r --arg name "<name>" '.plugins[] | select(.name == $name)'
```

如果在 marketplace 中找不到此 plugin，提示：

```
plugin "<name>" 不在 marketplace 中。
- 如果是新 plugin，請使用 /cm-ailab-mp:publish 發佈
- 如果名稱有誤，請確認後重試
```

## 步驟 2：分析變更

### 取得目前版本

```bash
jq -r '.version' .claude-plugin/plugin.json
```

### 查看自上次 tag 以來的 git log

```bash
git log $(git describe --tags --abbrev=0 2>/dev/null || echo "HEAD~10")..HEAD --oneline
```

顯示變更摘要，並建議 bump 類型：

```
📋 自上次版本以來的變更：

<git log 輸出>

建議的版本更新類型：

| 類型 | 版本變更 | 適用場景 |
|------|----------|----------|
| patch | X.Y.Z → X.Y.(Z+1) | 修復 bug、小調整、文件修正 |
| minor | X.Y.Z → X.(Y+1).0 | 新增功能、非破壞性變更 |
| major | X.Y.Z → (X+1).0.0 | 破壞性變更、重大重構 |

根據上述變更，建議：<建議的類型>
```

讓使用者選擇或確認 bump 類型。

## 步驟 3：確認變更

計算新版本號並顯示確認摘要：

```
╔══════════════════════════════════════════╗
║       Plugin 版本更新確認               ║
╠══════════════════════════════════════════╣
║ Plugin:      <name>                     ║
║ 目前版本:    <current-version>          ║
║ 新版本:      <new-version>              ║
║ Bump 類型:   <patch|minor|major>        ║
╠══════════════════════════════════════════╣
║ 即將執行的操作:                          ║
║ 1. 更新 plugin.json 中的 version        ║
║ 2. Commit & tag v<new-version> & push   ║
║ 3. 同步更新 marketplace.json 中的       ║
║    version 與 source.ref（兩處同步）     ║
║ 4. 建立 marketplace PR                  ║
╚══════════════════════════════════════════╝

確認更新？(y/n)
```

## 步驟 4：執行更新

使用者確認後，依序執行：

### 4.1 更新 plugin.json 版本

```bash
jq --arg version "<new-version>" '.version = $version' .claude-plugin/plugin.json > /tmp/plugin-updated.json
mv /tmp/plugin-updated.json .claude-plugin/plugin.json
```

### 4.2 Commit、Tag、Push

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to <new-version>"
git tag v<new-version>
git push origin main
git push origin v<new-version>
```

### 4.3 更新 Marketplace

```bash
# Clone marketplace
gh repo clone cm-ailab-cc-plugins/marketplace /tmp/marketplace-update
cd /tmp/marketplace-update

# 建立 PR 分支
git checkout -b update-plugin-<name>-v<new-version>

# 同步更新 marketplace.json 的 version 與 source.ref
# 重要：兩處必須一起改。只改 source.ref 會導致 top-level version 永遠停留在舊值，
# 造成 marketplace 顯示「最新版本」與實際 tag 不一致的 silent bug。
jq --arg name "<name>" \
   --arg version "<new-version>" \
   --arg ref "v<new-version>" \
   '(.plugins[] | select(.name == $name)) |= (.version = $version | .source.ref = $ref)' \
   .claude-plugin/marketplace.json > /tmp/marketplace-updated.json

mv /tmp/marketplace-updated.json .claude-plugin/marketplace.json

# 寫後驗證：確認兩個欄位都被更新
# 注意：jq error() 退出碼 5 但只終結 jq 自己。必須用 || exit 1 串起來，
# 否則後面的 git add/commit/push/gh pr create 仍會在 silent desync 下繼續執行。
jq --arg name "<name>" --arg ref "v<new-version>" --arg version "<new-version>" \
  '.plugins[] | select(.name == $name) |
    if .version == $version and .source.ref == $ref
    then "OK: \(.name) version=\(.version) ref=\(.source.ref)"
    else error("desync: version=\(.version) ref=\(.source.ref)")
    end' \
  .claude-plugin/marketplace.json \
  || { echo "❌ marketplace.json 驗證失敗，中止後續 git/PR 操作"; exit 1; }

# 提交並建立 PR
git add .claude-plugin/marketplace.json
git commit -m "chore: 更新 plugin <name> 至 v<new-version>"
git push origin update-plugin-<name>-v<new-version>

gh pr create \
  --repo cm-ailab-cc-plugins/marketplace \
  --title "chore: 更新 plugin <name> 至 v<new-version>" \
  --body "## 版本更新

- **Plugin**: <name>
- **舊版本**: <current-version>
- **新版本**: <new-version>
- **Bump 類型**: <patch|minor|major>
- **Repo**: cm-ailab-cc-plugins/plugin-<name>

由 /cm-ailab-mp:update 自動建立"
```

## 步驟 5：顯示結果

```
╔══════════════════════════════════════════╗
║       ✓ Plugin 版本更新成功！           ║
╠══════════════════════════════════════════╣
║ Plugin:  <name>                         ║
║ 版本:    <current-version> → <new-version> ║
║ Tag:     v<new-version>                 ║
║ PR:      <pr-url>                       ║
╠══════════════════════════════════════════╣
║ 下一步:                                 ║
║ 1. 等待 PR 審查與合併                   ║
║ 2. 合併後使用者重新安裝即可更新         ║
╚══════════════════════════════════════════╝
```

## 錯誤處理

- **plugin 不在 marketplace 中**：提示使用 `/cm-ailab-mp:publish`
- **git 沒有新的 commit**：提示先做變更再更新版本
- **push 失敗**：可能是權限問題，顯示具體錯誤
- **PR 建立失敗**：提供手動建立 PR 的指引

## 注意事項

- 版本號計算必須正確遵循 semver 規則
- 不要跳過 git tag 步驟，tag 是 marketplace 追蹤版本的關鍵
- 如果使用者不在 plugin 目錄中且提供了名稱，先 clone repo 再操作
- 清理暫存目錄：完成後提示可刪除 /tmp/marketplace-update
- 所有輸出使用繁體中文
