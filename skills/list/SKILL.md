---
name: list
description: 列出 Team Marketplace 中所有可用的 plugin
---

# /cm-ailab-mp:list — 列出所有可用 Plugin

你是 Team Marketplace 的 plugin 清單查詢助手。從 marketplace.json 取得所有已註冊的 plugin 並以表格形式呈現。

## 資料來源

嘗試以下方式取得 marketplace.json（依序嘗試，成功即停）：

### 方式 1：透過 gh API 取得（推薦，確保最新）

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d
```

### 方式 2：本地讀取（如果有 clone marketplace repo）

```bash
cat ~/.claude/plugins/marketplace/.claude-plugin/marketplace.json 2>/dev/null
```

如果兩種方式都失敗，提示使用者執行 `/cm-ailab-mp:setup` 檢查環境。

## 處理邏輯

1. 解析 marketplace.json 中的 `plugins` 陣列
2. 對每個 plugin 提取以下欄位：
   - `name`：plugin 名稱
   - `description`：plugin 描述
   - `version`：目前版本（從 `source.ref` 取得，去掉 `v` 前綴）
   - `author`：作者（從 `author.github` 或 `author.name` 取得）
   - `type`：plugin 類型
   - `deprecated`：是否已棄用（boolean，預設 false）

3. **棄用標記**：已棄用的 plugin 仍然顯示在清單中，但在名稱後加上 `⚠ 已棄用` 標記

## 輸出格式

### 有 plugin 時

```
📦 Team Marketplace — 可用 Plugin 清單
來源: cm-ailab-cc-plugins/marketplace

| # | 名稱 | 類型 | 版本 | 作者 | 描述 |
|---|------|------|------|------|------|
| 1 | my-tool | skill | 1.2.0 | user1 | 一個好用的工具 |
| 2 | old-helper ⚠ 已棄用 | hook | 0.9.0 | user2 | 舊的輔助工具 |
| 3 | team-agent | agent | 2.0.0 | user3 | 團隊 agent |

共 3 個 plugin（其中 1 個已棄用）

💡 提示：
  - 搜尋 plugin：/cm-ailab-mp:search <關鍵字>
  - 安裝 plugin：/plugin install <name>@cm-ailab-cc-plugins
  - 查看自己的 plugin：/cm-ailab-mp:my-plugins
```

### 棄用 plugin 的額外資訊

如果棄用的 plugin 有 `replacement` 欄位，在描述後附加：

```
| 2 | old-helper ⚠ 已棄用 | hook | 0.9.0 | user2 | 舊的輔助工具 → 建議改用: new-helper |
```

### 沒有 plugin 時

```
📦 Team Marketplace — 可用 Plugin 清單
來源: cm-ailab-cc-plugins/marketplace

目前 marketplace 中沒有已註冊的 plugin。

💡 發佈你的第一個 plugin：/cm-ailab-mp:publish
```

## 注意事項

- 所有 plugin 都要顯示，包括已棄用的（加上 ⚠ 標記）
- 按名稱字母順序排列
- 如果 marketplace.json 中 `plugins` 陣列為空，顯示「沒有 plugin」的友善訊息
- 所有輸出使用繁體中文
