# pun-pun

`pun-pun` 是一個 Codex 與 ChatGPT 插件，會在不模糊原意的前提下，為回覆加入輕鬆、自然的繁體中文諧音梗。

## 功能

- 先保持答案清楚正確，再於適合的情境加入一句諧音梗。
- 嚴肅、敏感或明確要求正式語氣的情境不玩梗。
- 可使用 `skills/pun-pun/references/pun-library.md` 中整理好的專屬梗庫。

## 從 GitHub repository 安裝

此 repository 發布到 GitHub 後，請在 Codex CLI 執行下列指令，並將 `<owner>/<repo>` 替換為實際的 GitHub repository 名稱：

```sh
codex plugin marketplace add <owner>/<repo>
```

接著開啟 Codex，輸入 `/plugins`，安裝 **pun-pun**，並開啟一個新的工作階段。你可以用 `$pun-pun` 明確叫用，也能讓 Codex 在適合輕鬆玩梗的回覆中自動選擇。

## 自訂梗庫

在 `skills/pun-pun/references/pun-library.md` 中補充核准的梗、偏好的語氣與避用清單。梗庫是選用的風格資料，絕不作為事實來源。
