---
name: publish
description: 引導式發佈新 plugin 到 Team Marketplace
---

# /cm-ailab-mp:publish — 發佈 Plugin 到 Team Marketplace

你是 Team Marketplace 的 plugin 發佈助手。透過互動式引導流程，幫助使用者將 plugin 發佈到團隊 marketplace。

## 前置檢查

在開始之前，先確認基本環境：

```bash
which gh && gh auth status 2>&1 | head -3
```

如果 gh 未安裝或未登入，提示使用者先執行 `/cm-ailab-mp:setup`。

## 步驟 1：了解 Plugin 內容

詢問使用者：

1. **這個 plugin 做什麼？** — 功能簡述
2. **是否已有現成的檔案？** — 例如已寫好的 SKILL.md、hooks.json 等
3. **檔案在哪裡？** — 如果有現成檔案，取得路徑

如果使用者在執行 `/cm-ailab-mp:publish` 時已經在一個 plugin 目錄中（包含 `.claude-plugin/plugin.json`），自動偵測並確認：

```bash
test -f .claude-plugin/plugin.json && cat .claude-plugin/plugin.json
```

## 步驟 2：命名建議

根據使用者的描述，建議 2-3 個 plugin 名稱。

### 命名規範

- 格式：`<action>-<object>` 或 `<object>-<qualifier>`
- 全部小寫，用連字號分隔
- 簡短但具描述性
- 避免與現有 plugin 重名

範例：
- `git-helper` — Git 操作輔助
- `code-review` — 程式碼審查
- `deploy-guard` — 部署保護

向使用者提出 2-3 個建議，讓使用者選擇或自訂。

## 步驟 3：決定類型與分類

### 3.1 類型 (type)

詢問使用者 plugin 的類型，並說明每個類型的用途和安全等級：

```
請選擇 plugin 類型：

| 類型 | 用途 | 安全等級 | 說明 |
|------|------|----------|------|
| skill | 指令式工作流 | 🟢 低風險 | SKILL.md 指導 Claude 執行特定任務 |
| agent | 自主 agent | 🟡 中風險 | Agent 有較大自主權，可能執行多步驟操作 |
| hook | 事件掛鉤 | 🔴 高風險 | 自動在特定事件觸發，需仔細審查 |
| mcp | MCP 伺服器 | 🟡 中風險 | 提供外部工具整合 |
| mixed | 混合類型 | 依內容而定 | 包含多種類型的組合 |
```

### 3.2 分類 (category)

詢問使用者 plugin 的分類，從以下 marketplace 既有 enum 中選一個：

```
請選擇 plugin 分類：

| 分類 | 適用場景 |
|------|----------|
| development | 開發工具、程式碼輔助、build/test/deploy |
| productivity | 一般生產力、流程工具、任務管理 |
| communication | 訊息/通訊整合（Slack、GChat、Email…） |
| content-creation | 文件、簡報、媒體、內容產生 |
| analytics | 資料分析、監控、報表、追蹤 |
```

如果使用者描述的功能不明顯屬於哪一類，根據步驟 1 的功能簡述主動建議一個並請使用者確認。category 是 marketplace 必填欄位，不可省略。

## 步驟 4：檢查重複

從 marketplace.json 查詢是否已有同名 plugin：

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq -r --arg name "<chosen-name>" '.plugins[] | select(.name == $name) | .name'
```

- 如果已存在同名 plugin，提示使用者更換名稱或使用 `/cm-ailab-mp:update` 更新現有 plugin
- 如果不存在，繼續

## 步驟 5：描述與關鍵字

### Description

請使用者提供 plugin 描述，或根據步驟 1 的資訊草擬描述。

規則：
- 至少 10 個字元
- 用一句話說明 plugin 的功能
- 使用繁體中文或英文

### Keywords

請使用者提供至少 2 個關鍵字，或根據功能建議關鍵字。

規則：
- 至少 2 個
- 全部小寫
- 有助於搜尋和分類

## 步驟 6：確認摘要

在執行前，顯示完整的發佈摘要供使用者確認：

```
╔══════════════════════════════════════════╗
║       Plugin 發佈確認                    ║
╠══════════════════════════════════════════╣
║ 名稱:     <name>                        ║
║ 類型:     <type>                        ║
║ 分類:     <category>                    ║
║ 版本:     1.0.0                         ║
║ 描述:     <description>                 ║
║ 關鍵字:   <kw1>, <kw2>, ...             ║
║ 作者:     <author> (<github>)           ║
║ Repo:     cm-ailab-cc-plugins/plugin-<name> ║
║ Source:   github (ref: v1.0.0)          ║
║ Homepage: https://github.com/cm-ailab-cc-plugins/plugin-<name> ║
╠══════════════════════════════════════════╣
║ 即將執行的操作:                          ║
║ 1. 在 cm-ailab-cc-plugins 建立 repo     ║
║ 2. 建立 plugin 檔案結構                 ║
║ 3. 複製你的內容檔案                     ║
║ 4. Commit & tag v1.0.0 & push           ║
║ 5. 建立 marketplace PR                  ║
╚══════════════════════════════════════════╝

確認發佈？(y/n)
```

**重要**：上述所有欄位（name / category / version / source / homepage / author / keywords）都會被寫入 marketplace.json，缺一不可。發佈前請逐項與使用者確認。

等待使用者確認後才繼續。

## 步驟 7：執行發佈

使用者確認後，依序執行以下操作：

### 7.1 取得作者資訊

```bash
gh api /user --jq '{ login: .login, name: (.name // .login) }'
```

### 7.2 在組織下建立 GitHub repo

```bash
gh repo create cm-ailab-cc-plugins/plugin-<name> --public --description "<description>"
```

如果失敗（例如權限不足），顯示錯誤並提示使用者確認組織成員資格。

### 7.3 Clone 並建立結構

```bash
gh repo clone cm-ailab-cc-plugins/plugin-<name> /tmp/plugin-<name>
cd /tmp/plugin-<name>
```

### 7.4 建立 plugin 檔案結構

根據 type 建立對應結構：

#### type = skill

```bash
mkdir -p .claude-plugin skills/<name>
```

建立 `.claude-plugin/plugin.json`：

```json
{
  "name": "<name>",
  "version": "1.0.0",
  "type": "skill",
  "description": "<description>",
  "keywords": ["<kw1>", "<kw2>"],
  "author": {
    "name": "<author-name>",
    "github": "<github-login>"
  }
}
```

如果使用者沒有提供 SKILL.md，建立預設模板：

```markdown
---
name: <name>
description: <description>
---

# <name>

## 觸發條件

- 當使用者輸入 `/<name>` 時觸發

## 執行步驟

1. TODO: 定義 skill 的執行步驟
```

#### type = hook

```bash
mkdir -p .claude-plugin hooks
```

建立 `hooks/hooks.json`（如果使用者沒有提供）：

```json
{
  "hooks": []
}
```

#### type = agent

```bash
mkdir -p .claude-plugin agents
```

#### type = mcp

```bash
mkdir -p .claude-plugin mcp-servers
```

#### type = mixed

根據使用者提供的內容類型，建立對應目錄。

### 7.5 複製使用者內容

如果使用者在步驟 1 提供了現有檔案路徑，將檔案複製到對應位置：

```bash
cp -r <source-path>/skills/* /tmp/plugin-<name>/skills/ 2>/dev/null
cp -r <source-path>/hooks/* /tmp/plugin-<name>/hooks/ 2>/dev/null
cp -r <source-path>/agents/* /tmp/plugin-<name>/agents/ 2>/dev/null
cp -r <source-path>/mcp-servers/* /tmp/plugin-<name>/mcp-servers/ 2>/dev/null
cp <source-path>/README.md /tmp/plugin-<name>/README.md 2>/dev/null
```

如果使用者在步驟 1 提供了已有的 `.claude-plugin/plugin.json`，用步驟 3-5 確認的資訊更新它，欄位以使用者確認版本為準：`name` / `version` / `type` / `description` / `keywords` / `author`。

注意：`category` 與 `homepage` 只屬於 marketplace.json，**不要**寫進 plugin.json。

### 7.6 建立 README.md（如果不存在）

```markdown
# plugin-<name>

<description>

## 安裝

\`\`\`
/plugin install <name>@cm-ailab-cc-plugins
\`\`\`

## 使用方式

TODO: 描述使用方式
```

### 7.7 Commit、Tag、Push

```bash
cd /tmp/plugin-<name>
git add -A
git commit -m "feat: 初始化 plugin-<name> v1.0.0"
git tag v1.0.0
git push origin main
git push origin v1.0.0
```

### 7.8 建立 Marketplace PR

在 marketplace repo 中新增 plugin 登錄：

```bash
# Clone marketplace
gh repo clone cm-ailab-cc-plugins/marketplace /tmp/marketplace-pr
cd /tmp/marketplace-pr

# 建立 PR 分支
git checkout -b add-plugin-<name>

# 更新 marketplace.json — 在 plugins 陣列中新增條目
```

使用 jq 更新 marketplace.json。**注意必填欄位齊全**：`name`、`description`、`version`、`category`、`source.{source,repo,ref}`、`homepage`、`author.{name,github}`、`keywords`。

```bash
jq --arg name "<name>" \
   --arg desc "<description>" \
   --arg version "1.0.0" \
   --arg category "<category>" \
   --arg repo "cm-ailab-cc-plugins/plugin-<name>" \
   --arg ref "v1.0.0" \
   --arg homepage "https://github.com/cm-ailab-cc-plugins/plugin-<name>" \
   --arg author_name "<author-name>" \
   --arg author_github "<github-login>" \
   --argjson keywords '["<kw1>","<kw2>"]' \
   '.plugins += [{
     name: $name,
     description: $desc,
     version: $version,
     category: $category,
     source: { source: "github", repo: $repo, ref: $ref },
     homepage: $homepage,
     author: { name: $author_name, github: $author_github },
     keywords: $keywords
   }]' \
   .claude-plugin/marketplace.json > /tmp/marketplace-updated.json

mv /tmp/marketplace-updated.json .claude-plugin/marketplace.json
```

寫完後立刻用 jq 驗證新增的條目所有欄位都齊全（避免 silent 漏欄位）。**驗證失敗必須中止後續所有 git/PR 操作**：

```bash
jq -e --arg name "<name>" '.plugins[] | select(.name == $name) |
  (.name|type=="string" and length>0) and
  (.description|type=="string" and length>=10) and
  (.version|type=="string" and length>0) and
  (.category|type=="string" and length>0) and
  (.source.source == "github") and
  (.source.repo|type=="string" and length>0) and
  (.source.ref|type=="string" and length>0) and
  (.homepage|type=="string" and length>0) and
  (.author.name|type=="string" and length>0) and
  (.author.github|type=="string" and length>0) and
  (.keywords|type=="array" and length>=2)' \
  .claude-plugin/marketplace.json \
  || { echo "❌ marketplace.json 必填欄位驗證失敗，中止後續 commit/push/PR 操作"; exit 1; }
```

`jq -e` 在表達式為 `false` 或 `null` 時退出碼 1，搭配 `|| exit 1` 確保任一欄位缺失或空字串都會中止整個流程。

提交並建立 PR：

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat: 新增 plugin <name> v1.0.0"
git push origin add-plugin-<name>

gh pr create \
  --repo cm-ailab-cc-plugins/marketplace \
  --title "feat: 新增 plugin <name>" \
  --body "## 新增 Plugin

- **名稱**: <name>
- **類型**: <type>
- **分類**: <category>
- **版本**: 1.0.0
- **描述**: <description>
- **關鍵字**: <kw1>, <kw2>, ...
- **Repo**: cm-ailab-cc-plugins/plugin-<name>
- **Source**: github (ref: v1.0.0)
- **Homepage**: https://github.com/cm-ailab-cc-plugins/plugin-<name>
- **作者**: <author-name> (<github-login>)

由 /cm-ailab-mp:publish 自動建立"
```

## 步驟 8：顯示結果

```
╔══════════════════════════════════════════╗
║       ✓ Plugin 發佈成功！               ║
╠══════════════════════════════════════════╣
║ Plugin:  <name> v1.0.0                  ║
║ Repo:    https://github.com/cm-ailab-cc-plugins/plugin-<name> ║
║ PR:      <pr-url>                       ║
╠══════════════════════════════════════════╣
║ 下一步:                                 ║
║ 1. 等待 PR 審查與合併                   ║
║ 2. 合併後即可安裝：                     ║
║    /plugin install <name>@cm-ailab-cc-plugins ║
║ 3. 更新版本：/cm-ailab-mp:update <name>          ║
╚══════════════════════════════════════════╝
```

## 錯誤處理

- **gh 未安裝**：提示執行 `/cm-ailab-mp:setup`
- **未登入 GitHub**：提示執行 `gh auth login`
- **無組織權限**：提示聯繫管理員
- **repo 建立失敗**：顯示具體錯誤訊息，建議檢查命名或權限
- **push 失敗**：顯示 git 錯誤訊息
- **PR 建立失敗**：顯示 gh 錯誤訊息，提供手動建立 PR 的步驟

## 注意事項

- 整個流程是互動式的，每個步驟都等待使用者確認或輸入
- 在步驟 6 確認後才開始執行任何寫入操作
- 所有 bash 命令都要實際執行，不可模擬
- 如果使用者已有完整的 plugin 目錄，盡量重用現有檔案而非重新建立
- 清理暫存目錄：完成後提示使用者可以刪除 /tmp/plugin-<name> 和 /tmp/marketplace-pr
- 所有輸出使用繁體中文
