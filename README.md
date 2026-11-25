# NIGHTLINE // HOLO

非官方《電馭叛客 2077》NPC 私訊網站前端（Mock Mode）。以 V 的身分和茱蒂、帕娜、強尼傳訊息，回覆為預寫的模擬文字，尚未串接 AI。

- 單檔 `index.html`，直接開啟或用 GitHub Pages 部署即可
- 對話與草稿存在瀏覽器 localStorage

本專案為粉絲作品，與 CD PROJEKT RED 無關。角色名稱權利屬原權利人，對話內容皆為原創。

## 之後接 AI 的位置

回覆內容全部由 `index.html` 裡的 `replyProvider` 提供，聊天介面只負責送出、顯示「輸入中」與顯示回覆。換成後端 API 時，只要把 `replyProvider` 換掉，訊息框、輸入中狀態、聊天紀錄、未讀與捲動行為都沿用。

```js
// 約定：回傳 Promise<string[]>（一則或多則回覆）；失敗就丟出錯誤
replyProvider.reply({
  contactId,            // "judy" | "panam" | "johnny"
  profile,              // { name, role, rel, traits }
  text,                 // V 這次送出的訊息
  history               // [{ role: "user" | "assistant", text, t }]
})
```

- 目前使用 `MockProvider`，關鍵字與預設台詞集中在 `mockCompose()`。
- 每個聊天室各自排隊，回覆一定寫回發問的聊天室；中途重設會讓舊回覆作廢。
- provider 丟出錯誤時，聊天室會顯示「訊號中斷」，不會寫入角色台詞。
