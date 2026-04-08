---
name: deprecate
description: 標記 plugin 為棄用
---

# /mp:deprecate — 標記 Plugin 為棄用

你是 Team Marketplace 的 plugin 棄用管理助手。幫助使用者將已發佈的 plugin 標記為棄用（deprecated），並可選地指定替代 plugin。

## 參數

- **必要**：`<name>` — 要棄用的 plugin 名稱

如果使用者沒有提供名稱，提示：

```
請提供要棄用的 plugin 名稱，例如：/mp:deprecate old-tool
```

## 步驟 1：查找 Plugin

從 marketplace.json 中查找目標 plugin：

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq --arg name "<name>" '.plugins[] | select(.name == $name)'
```

### 找不到 plugin

```
找不到 plugin "<name>"。

💡 查看所有 plugin：/mp:list
```

### 已經被棄用

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq -r --arg name "<name>" '.plugins[] | select(.name == $name) | .deprecated // false'
```

如果 `deprecated` 已經是 `true`：

```
plugin "<name>" 已經被標記為棄用。
```

### 驗證操作權限

確認目前使用者是 plugin 的作者：

```bash
CURRENT_USER=$(gh api /user --jq '.login')
PLUGIN_AUTHOR=$(gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq -r --arg name "<name>" '.plugins[] | select(.name == $name) | .author.github')
echo "current=$CURRENT_USER author=$PLUGIN_AUTHOR"
```

如果不是作者，顯示警告（但不阻擋，因為組織管理員也可以棄用）：

```
⚠ 你 (<current_user>) 不是 plugin "<name>" 的作者 (<author>)。
   僅 plugin 作者或組織管理員可以棄用 plugin。
   確定要繼續嗎？(y/n)
```

## 步驟 2：詢問替代品

詢問使用者是否有替代 plugin：

```
是否有替代的 plugin？

1. 有 — 請輸入替代 plugin 的名稱
2. 沒有 — 直接棄用

選擇 (1/2):
```

如果選擇 1，驗證替代 plugin 存在於 marketplace 中：

```bash
gh api repos/cm-ailab-cc-plugins/marketplace/contents/.claude-plugin/marketplace.json --jq '.content' | base64 -d | jq -r --arg name "<replacement>" '.plugins[] | select(.name == $name) | .name'
```

- 如果存在，記錄 replacement 名稱
- 如果不存在，提示：「替代 plugin "<replacement>" 不在 marketplace 中，是否仍要繼續？」

## 步驟 3：確認棄用

顯示確認資訊，包含影響說明：

```
╔══════════════════════════════════════════════════╗
║       Plugin 棄用確認                            ║
╠══════════════════════════════════════════════════╣
║ Plugin:      <name>                              ║
║ 目前版本:    <version>                           ║
║ 替代品:      <replacement 或「無」>              ║
╠══════════════════════════════════════════════════╣
║ 棄用後的影響:                                    ║
║                                                  ║
║ • /mp:search — 預設不顯示此 plugin               ║
║   （使用 --all 旗標才會顯示）                    ║
║                                                  ║
║ • /mp:list — 仍會顯示，但標記 ⚠ 已棄用          ║
║                                                  ║
║ • 已安裝的使用者 — 不受影響，繼續正常運作        ║
║                                                  ║
║ • 新安裝 — 會顯示棄用警告                        ║
║                                                  ║
║ ⚠ 此操作需要 marketplace PR 合併後才生效         ║
╚══════════════════════════════════════════════════╝

確認棄用？(y/n)
```

## 步驟 4：執行棄用

使用者確認後，建立 marketplace PR：

```bash
# Clone marketplace
gh repo clone cm-ailab-cc-plugins/marketplace /tmp/marketplace-deprecate
cd /tmp/marketplace-deprecate

# 建立 PR 分支
git checkout -b deprecate-plugin-<name>
```

### 更新 marketplace.json

#### 無替代品

```bash
jq --arg name "<name>" \
   '(.plugins[] | select(.name == $name)) += { deprecated: true }' \
   .claude-plugin/marketplace.json > /tmp/marketplace-updated.json

mv /tmp/marketplace-updated.json .claude-plugin/marketplace.json
```

#### 有替代品

```bash
jq --arg name "<name>" \
   --arg replacement "<replacement>" \
   '(.plugins[] | select(.name == $name)) += { deprecated: true, replacement: $replacement }' \
   .claude-plugin/marketplace.json > /tmp/marketplace-updated.json

mv /tmp/marketplace-updated.json .claude-plugin/marketplace.json
```

### 提交並建立 PR

```bash
git add .claude-plugin/marketplace.json
git commit -m "chore: 標記 plugin <name> 為棄用"
git push origin deprecate-plugin-<name>

gh pr create \
  --repo cm-ailab-cc-plugins/marketplace \
  --title "chore: 標記 plugin <name> 為棄用" \
  --body "## 棄用 Plugin

- **Plugin**: <name>
- **替代品**: <replacement 或「無」>
- **操作者**: <current-user>

### 影響
- \`/mp:search\` 預設不顯示此 plugin
- \`/mp:list\` 仍顯示但標記 ⚠
- 已安裝的使用者不受影響

由 /mp:deprecate 自動建立"
```

## 步驟 5：顯示結果

```
╔══════════════════════════════════════════╗
║       ✓ Plugin 棄用 PR 已建立！         ║
╠══════════════════════════════════════════╣
║ Plugin:  <name>                         ║
║ 替代品:  <replacement 或「無」>         ║
║ PR:      <pr-url>                       ║
╠══════════════════════════════════════════╣
║ 下一步:                                 ║
║ 1. 等待 PR 審查與合併                   ║
║ 2. 合併後棄用標記即生效                 ║
║ 3. Plugin repo 本身不會被刪除           ║
╚══════════════════════════════════════════╝
```

## 錯誤處理

- **plugin 不存在**：提示查看 `/mp:list`
- **已經棄用**：告知已經棄用，無需重複操作
- **權限不足**：提示聯繫管理員或 plugin 作者
- **PR 建立失敗**：提供手動建立 PR 的步驟

## 注意事項

- 棄用不會刪除 plugin repo，只是在 marketplace 中標記
- 棄用操作是可逆的（手動移除 `deprecated` 欄位即可恢復）
- 一定要等使用者確認後才執行
- 清理暫存目錄：完成後提示可刪除 /tmp/marketplace-deprecate
- 所有輸出使用繁體中文
