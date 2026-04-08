---
name: validate
description: 本地驗證 plugin 結構是否符合 marketplace 規範
---

# /mp:validate — 本地 Plugin 結構驗證

你是 Plugin 結構驗證助手。在使用者發佈 plugin 之前，於本地驗證其結構是否符合 Team Marketplace 規範。這等同於 CI 上的驗證檢查，讓使用者在推送前就能發現問題。

## 前置條件

確認目前所在目錄是一個 plugin 專案（包含 `.claude-plugin/plugin.json`）。如果不是，提示使用者切換到正確目錄。

## 驗證流程

依序執行以下所有驗證項目，每項用 ✓（通過）、✗（失敗）、⚠（警告）標記。

### 第一階段：結構驗證

#### 1.1 plugin.json 存在性

```bash
test -f .claude-plugin/plugin.json && echo "EXISTS" || echo "NOT_FOUND"
```

- **✓**：檔案存在，繼續驗證
- **✗**：檔案不存在，終止驗證並提示：
  ```
  缺少 .claude-plugin/plugin.json，這是 plugin 的核心設定檔。
  請參考 marketplace 的 templates/ 目錄建立。
  ```

#### 1.2 JSON 格式有效性

```bash
jq empty .claude-plugin/plugin.json 2>&1
```

- **✓**：JSON 格式正確
- **✗**：JSON 格式錯誤，顯示 jq 的錯誤訊息

#### 1.3 必要欄位檢查

逐一讀取以下欄位：

```bash
jq -r '.name // empty' .claude-plugin/plugin.json
jq -r '.version // empty' .claude-plugin/plugin.json
jq -r '.type // empty' .claude-plugin/plugin.json
jq -r '.description // empty' .claude-plugin/plugin.json
jq -r '.keywords | length' .claude-plugin/plugin.json
jq -r '.author.github // empty' .claude-plugin/plugin.json
```

每個欄位檢查結果：

| 欄位 | 規則 | 失敗訊息 |
|------|------|----------|
| `name` | 非空 | 缺少 name 欄位 |
| `version` | 非空 | 缺少 version 欄位 |
| `type` | 非空 | 缺少 type 欄位 |
| `description` | 非空 | 缺少 description 欄位 |
| `keywords` | 長度 >= 2 | keywords 需要至少 2 個 |
| `author.github` | 非空 | 缺少 author.github 欄位 |

#### 1.4 type 有效值

```bash
jq -r '.type' .claude-plugin/plugin.json
```

有效值：`skill`、`agent`、`hook`、`mcp`、`mixed`

- **✓**：值在有效列表中
- **✗**：`type 不是有效值: <value>（必須是 skill/agent/hook/mcp/mixed）`

#### 1.5 version semver 格式

```bash
jq -r '.version' .claude-plugin/plugin.json
```

驗證是否符合 semver 格式（`X.Y.Z`，可選 `-prerelease` 和 `+metadata`）：

- **✓**：符合 semver
- **✗**：`version 不是有效的 semver: <value>`

### 第二階段：命名規範驗證

#### 2.1 repo 名稱與 plugin name 一致

取得目前 git remote 的 repo 名稱：

```bash
basename "$(git remote get-url origin 2>/dev/null)" .git 2>/dev/null || basename "$(pwd)"
```

期望格式：`plugin-<name>`，其中 `<name>` 是 plugin.json 中的 name 欄位。

- **✓**：`repo 名稱符合: plugin-<name>`
- **✗**：`repo 名稱不符: <actual> != plugin-<name>`
- **⚠**：如果無法取得 remote URL（尚未推送），以目錄名稱做初步檢查

### 第三階段：內容類型驗證

根據 plugin.json 中的 `type` 欄位，驗證對應的內容結構：

#### type = skill

```bash
find skills -name "SKILL.md" 2>/dev/null | head -5
```

- 必須存在至少一個 `skills/*/SKILL.md`
- 每個 SKILL.md 必須包含 frontmatter（開頭 `---`）且有 `description:` 欄位：

```bash
head -20 skills/*/SKILL.md | grep -c "description:"
```

#### type = hook

```bash
test -f hooks/hooks.json && echo "EXISTS" || echo "NOT_FOUND"
jq empty hooks/hooks.json 2>&1
```

- 必須存在 `hooks/hooks.json` 且為有效 JSON

#### type = agent

```bash
find agents -name "*.md" 2>/dev/null | head -5
```

- 必須存在至少一個 `agents/*.md`
- agent md 檔案必須包含 frontmatter（開頭 `---`）

#### type = mcp

```bash
find mcp-servers -name "*.json" 2>/dev/null | head -5
```

- 必須存在 `mcp-servers/*.json`
- JSON 中必須包含 `command` 欄位

#### type = mixed

- 必須包含至少 2 種以上的內容類型（skills/agents/hooks/mcp-servers）

### 第四階段：交叉驗證

#### 4.1 hooks 目錄一致性

如果存在 `hooks/hooks.json`，type 必須是 `hook` 或 `mixed`：

```bash
test -f hooks/hooks.json && echo "HAS_HOOKS" || echo "NO_HOOKS"
```

- **✗**：`hooks/hooks.json 存在但 type=<type>（只有 hook/mixed 可以有 hooks）`

#### 4.2 mcp-servers 目錄一致性

如果存在 `mcp-servers/` 目錄且包含 JSON 檔案，type 必須是 `mcp` 或 `mixed`：

```bash
find mcp-servers -name "*.json" 2>/dev/null | head -1
```

- **✗**：`mcp-servers/ 存在但 type=<type>（只有 mcp/mixed 可以有 mcp-servers）`

### 第五階段：品質警告

這些項目不會導致驗證失敗，但會發出警告：

#### 5.1 description 長度

```bash
jq -r '.description | length' .claude-plugin/plugin.json
```

- **⚠**：description 少於 10 字元時警告

#### 5.2 README.md 存在

```bash
test -f README.md && echo "EXISTS" || echo "NOT_FOUND"
```

- **⚠**：缺少 README.md

#### 5.3 SKILL.md frontmatter 完整性（僅 skill/mixed 類型）

檢查每個 SKILL.md 是否同時有 `name:` 和 `description:` frontmatter：

```bash
for f in skills/*/SKILL.md; do
  echo "--- $f ---"
  head -10 "$f" | grep -E "^(name|description):"
done
```

- **⚠**：缺少 name 或 description frontmatter 的 SKILL.md

## 輸出格式

完成所有檢查後，輸出結構化報告：

```
╔══════════════════════════════════════════════════╗
║         Plugin 結構驗證報告                      ║
╠══════════════════════════════════════════════════╣
║ Plugin: <name> v<version> (type: <type>)        ║
╠══════════════════════════════════════════════════╣
║                                                  ║
║ 📋 結構驗證                                      ║
║   ✓ plugin.json 存在                             ║
║   ✓ JSON 格式有效                                ║
║   ✓ 必要欄位完整 (6/6)                           ║
║   ✓ type 是有效值: skill                         ║
║   ✓ version 是有效 semver: 1.0.0                 ║
║                                                  ║
║ 📛 命名規範                                      ║
║   ✓ repo 名稱符合: plugin-<name>                 ║
║                                                  ║
║ 📦 內容驗證 (type: skill)                        ║
║   ✓ SKILL.md 存在 (共 3 個)                      ║
║   ✓ 所有 SKILL.md 包含 description frontmatter   ║
║                                                  ║
║ 🔗 交叉驗證                                      ║
║   ✓ hooks 目錄一致性                             ║
║   ✓ mcp-servers 目錄一致性                       ║
║                                                  ║
║ 💡 品質檢查                                      ║
║   ✓ description 長度足夠                         ║
║   ⚠ 缺少 README.md                              ║
║                                                  ║
╠══════════════════════════════════════════════════╣
║ 結果: 0 錯誤, 1 警告                            ║
║ 狀態: ✓ 驗證通過，可以發佈！                     ║
╚══════════════════════════════════════════════════╝
```

如果有錯誤：

```
║ 結果: 2 錯誤, 1 警告                            ║
║ 狀態: ✗ 請修復上述錯誤後重新執行 /mp:validate    ║
```

## 注意事項

- 每個檢查都必須實際執行 bash 命令，不可假設結果
- 即使某項失敗，也繼續執行後續檢查（第 1.1 和 1.2 除外，因為後續依賴 plugin.json）
- 所有輸出使用繁體中文
- 此驗證等同於 CI 上的 `validate-plugin.sh`，通過此驗證即可確保 PR 不會被 CI 拒絕
