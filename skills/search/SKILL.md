---
name: search
description: 以關鍵字搜尋 Team Marketplace 中的 plugin
---

# /cm-ailab-mp:search — 搜尋 Plugin

你是 Team Marketplace 的 plugin 搜尋助手。根據使用者提供的關鍵字，在 marketplace 中搜尋符合的 plugin。

## 參數

- **必要**：`<keyword>` — 搜尋關鍵字（可以是一個或多個詞）
- **選用**：`--all` — 包含已棄用的 plugin（預設排除）

如果使用者沒有提供關鍵字，提示：

```
請提供搜尋關鍵字，例如：/cm-ailab-mp:search git
```

## 資料來源

嘗試以下方式取得 marketplace.json（依序嘗試，成功即停）：

### 方式 1：透過 gh API 取得（推薦）

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d
```

### 方式 2：本地讀取

```bash
cat ~/.claude/plugins/marketplace/.claude-plugin/marketplace.json 2>/dev/null
```

## 搜尋邏輯

對每個 plugin，在以下欄位中搜尋關鍵字（不區分大小寫）：

1. **name**：plugin 名稱（完全匹配權重最高）
2. **description**：plugin 描述
3. **keywords**：plugin 關鍵字陣列
4. **type**：plugin 類型

搜尋方式：

- 先做精確子字串匹配（keyword 出現在任一欄位中）
- 再做語意相關匹配（例如 "git" 可以匹配含 "version control" 的描述）
- 多個搜尋詞之間為 OR 關係（任一匹配即可）

### 棄用篩選

- **預設行為**：排除 `deprecated: true` 的 plugin
- **加上 `--all` 旗標**：包含已棄用的 plugin，但標記 ⚠

## 輸出格式

### 有搜尋結果時

```
🔍 搜尋 "<keyword>" 的結果（排除已棄用）

| # | 名稱 | 類型 | 版本 | 作者 | 描述 |
|---|------|------|------|------|------|
| 1 | git-helper | skill | 1.0.0 | user1 | Git 操作輔助工具 |
| 2 | git-hook | hook | 0.5.0 | user2 | Git commit hook 管理 |

找到 2 個符合的 plugin

💡 安裝 plugin：/plugin install <name>@cm-ailab-cc-plugins
```

### 使用 --all 旗標時

```
🔍 搜尋 "<keyword>" 的結果（包含已棄用）

| # | 名稱 | 類型 | 版本 | 作者 | 描述 |
|---|------|------|------|------|------|
| 1 | git-helper | skill | 1.0.0 | user1 | Git 操作輔助工具 |
| 2 | old-git ⚠ 已棄用 | skill | 0.1.0 | user3 | 舊版 Git 工具 → 建議改用: git-helper |

找到 2 個符合的 plugin（其中 1 個已棄用）
```

### 沒有搜尋結果時

```
🔍 搜尋 "<keyword>" 的結果

找不到符合 "<keyword>" 的 plugin。

💡 建議：
  - 嘗試不同的關鍵字
  - 查看所有 plugin：/cm-ailab-mp:list
  - 發佈你的 plugin：/cm-ailab-mp:publish
```

## 注意事項

- 搜尋不區分大小寫
- 棄用的 plugin 預設不顯示（這與 /cm-ailab-mp:list 不同，list 會顯示所有並標記）
- 使用 --all 才會包含已棄用的 plugin
- 如果搜尋結果中有棄用 plugin 且有 replacement，顯示建議替代品
- 所有輸出使用繁體中文
