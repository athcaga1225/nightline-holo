# NIGHTLINE 前端 ↔ 私人後端 API

公開前端只呼叫以下端點；後端只回傳「角色要傳的簡訊」，不回傳人格、Prompt、原作語料、規則或內部狀態。
後端網址與存取碼由使用者在 `#debug` 面板輸入，只存在使用者自己的瀏覽器。

驗證：`Authorization: Bearer <存取碼>`；後端只接受允許清單內的來源（CORS）。

## GET /api/health
`{ ok, characters: ["judy", …], provider, devMode }`

## POST /api/reply
```json
{ "contactId": "judy", "text": "V 的訊息",
  "settings": { "relationship": "friend|lover", "timeline": "working_at_lizzie", "proactive": true },
  "client": { "tz": "Asia/Taipei" },
  "recent": [{ "role": "user|assistant", "text": "…", "t": 0 }] }
```
回應：`{ "messages": ["…", "…"], "choices": null | ["…", "…", "…"] }`

## POST /api/sync
重新開啟網站（`phase: "on_open"`）或網站開著時（`"while_open"`）呼叫，取得可能的主動訊息。
```json
{ "contactId": "judy", "settings": { … }, "client": { "tz": "Asia/Taipei" }, "phase": "on_open" }
```
回應：`{ "messages": [{ "text", "src": "proactive", "trigger", "phase", "genAt", "eventAt", "t" }] }`

## 開發模式（後端 `NIGHTLINE_DEV_MODE=1` 才開放）
- `GET /api/debug/state?contactId=judy&tz=…` → 時鐘、Dynamic Context、近期生活事件、主動訊息紀錄
- `POST /api/debug/clock { "offsetMs": 0 }` → 模擬時間

資料結構見 `schemas/`。
