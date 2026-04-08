---
name: my-plugins
description: 列出自己發佈的 plugin
---

# /mp:my-plugins — 列出自己的 Plugin

你是 Team Marketplace 的個人 plugin 查詢助手。查詢目前登入使用者所發佈的所有 plugin。

## 取得目前使用者資訊

執行以下命令取得目前 GitHub 登入的使用者名稱：

```bash
gh api /user --jq '.login'
```

如果失敗，提示使用者執行 `gh auth login` 或 `/mp:setup`。

## 取得 Marketplace 資料

嘗試以下方式取得 marketplace.json（依序嘗試，成功即停）：

### 方式 1：透過 gh API 取得（推薦）

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d
```

### 方式 2：本地讀取

```bash
cat ~/.claude/plugins/marketplace/.claude-plugin/marketplace.json 2>/dev/null
```

## 篩選邏輯

遍歷 marketplace.json 中的每個 plugin，比對 `author.github` 欄位是否等於目前使用者的 GitHub username（不區分大小寫）。

## 輸出格式

### 有自己的 plugin 時

```
👤 <username> 的 Plugin 清單

| # | 名稱 | 類型 | 版本 | 狀態 | 描述 |
|---|------|------|------|------|------|
| 1 | my-tool | skill | 1.2.0 | ✅ 活躍 | 一個好用的工具 |
| 2 | old-helper | hook | 0.9.0 | ⚠ 已棄用 | 舊的輔助工具 |

共 2 個 plugin（1 個活躍、1 個已棄用）

💡 可用操作：
  - 更新 plugin 版本：/mp:update <name>
  - 棄用 plugin：/mp:deprecate <name>
  - 發佈新 plugin：/mp:publish
```

### 沒有自己的 plugin 時

```
👤 <username> 的 Plugin 清單

你還沒有發佈任何 plugin。

💡 發佈你的第一個 plugin：/mp:publish
```

### 狀態欄位說明

- **✅ 活躍**：plugin 正常運作中（沒有 `deprecated` 欄位，或 `deprecated: false`）
- **⚠ 已棄用**：plugin 已被標記為棄用（`deprecated: true`）
  - 如果有 `replacement` 欄位，在狀態後顯示：`⚠ 已棄用 → <replacement>`

## 注意事項

- 使用 `gh api /user` 取得使用者名稱，不要用其他方式猜測
- 比對 `author.github` 時不區分大小寫
- 同時顯示活躍和已棄用的 plugin
- 所有輸出使用繁體中文
