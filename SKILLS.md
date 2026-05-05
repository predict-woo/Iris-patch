# Iris API

KakaoTalk bot HTTP/WS bridge. Default `http://<device-ip>:3000` (configurable via `botHttpPort`). All JSON.

## Endpoints

### `GET /config`
Returns current bot config.
```json
{ "bot_name", "bot_http_port", "web_server_endpoint", "db_polling_rate", "message_send_rate", "bot_id" }
```

### `POST /config/{name}`
Update one config field. `{name}` ∈ `endpoint | botname | dbrate | sendrate | botport`.

Body (only relevant key required):
```json
{ "endpoint": "...", "botname": "...", "rate": "1000", "port": "3000" }
```
Note: `rate` and `port` are sent as **strings** (custom serializer parses to Long/Int). Returns `{ "success": true, "message": "success" }`.

### `POST /reply`
Send message to a room.
```json
{
  "type": "text" | "image" | "image_multiple",
  "room": "<chat_id>",
  "data": "hello" | "<base64-png>" | ["<b64>", "<b64>"],
  "threadId": "<optional>"
}
```
- `room` is the **numeric `chat_id`** (from `json.chat_id` in the WS payload, or the `chat_id` column in `chat_logs`). It is **not** the top-level `room` field of a WS event — that is the human-readable room name and will not work.
- `text` → `data` is a string.
- `image` → `data` is base64 PNG string.
- `image_multiple` → `data` is JSON array of base64 strings.

### `POST /query`
Run raw SQL on KakaoTalk DB. Rows returned with encrypted columns auto-decrypted.
```json
{ "query": "SELECT * FROM chat_logs WHERE id = ?", "bind": ["123"] }
```
Response: `{ "data": [ { "col": "val", ... } ] }` (all values stringified).

### `POST /decrypt`
Manually decrypt a Kakao-encrypted blob.
```json
{ "enc": <int>, "b64_ciphertext": "<base64>", "user_id": <long|null> }
```
`user_id` falls back to `botId` when null. Returns `{ "plain_text": "..." }`.

### `GET /aot`
Returns current AOT auth token JSON: `{ "success": true, "aot": { ... } }`.

### `GET /dashboard`
HTML status page.

### `GET /dashboard/status`
JSON status: `{ "isObserving", "statusMessage", "lastLogs": [...] }`.

### `WS /ws`
Streams new chat events as JSON strings. Subscribe and read each text frame.

Example frame (shape, not contract — see `ObserverHelper.kt` for authoritative fields):
```json
{
  "msg": "decrypted message text",
  "room": "Room Display Name",
  "sender": "Sender Display Name",
  "json": {
    "_id": "12345",
    "chat_id": "987654321",
    "user_id": "111222333",
    "message": "decrypted message text",
    "attachment": "{...}",
    "type": "1",
    "created_at": "1700000000",
    "v": "{...}"
  }
}
```
To reply to the originating room, use `json.chat_id` as `room` in `POST /reply`.

## Errors
Any exception → `500` with `{ "message": "<reason>" }`.

## Examples

Send text:
```bash
curl -X POST http://device:3000/reply \
  -H 'Content-Type: application/json' \
  -d '{"type":"text","room":"123456789","data":"hi"}'
```

Query last 10 messages:
```bash
curl -X POST http://device:3000/query \
  -H 'Content-Type: application/json' \
  -d '{"query":"SELECT * FROM chat_logs ORDER BY _id DESC LIMIT 10"}'
```

Listen for new chats:
```bash
websocat ws://device:3000/ws
```

## For AI agents
This file is a high-level overview. For specifics (exact field types, edge cases, serializer behavior, error paths), read the source — primarily [app/src/main/java/party/qwer/iris/IrisServer.kt](app/src/main/java/party/qwer/iris/IrisServer.kt) and the model classes under [app/src/main/java/party/qwer/iris/model/](app/src/main/java/party/qwer/iris/model/).
