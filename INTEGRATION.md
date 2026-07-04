# ProofChat — Integration Guide

> Phiên bản: 2026-07-04  
> Dành cho: team bên ngoài tích hợp ProofChat vào app (SuperApp, AladinWork, hoặc platform khác).  
> Nguồn đối chiếu kỹ thuật: [`docs/INTEGRATION_DEPLOYMENT.md`](https://github.com/ProofChat/BE/blob/docs/issue-11-integration-confirmations/docs/INTEGRATION_DEPLOYMENT.md) (ProofChat/BE#15)

---

## Tổng quan

ProofChat là hệ thống nhắn tin bảo mật đầu cuối (E2EE) dùng giao thức MLS.  
Server **không bao giờ thấy nội dung plaintext** — chỉ lưu ciphertext theo từng thiết bị.

| Thành phần | URL |
|---|---|
| REST API | `https://api.proofchat.me/api/v1` |
| WebSocket | xem §4 |

---

## 1. Xác thực (Authentication)

ProofChat dùng PhoenixKey DID làm identity duy nhất. Không có đăng ký riêng — người dùng đăng nhập bằng PhoenixKey QR, lấy `session_token`, rồi đổi sang ProofChat token.

### 1.1 Đổi PhoenixKey session_token → ProofChat token

```
POST /auth/phoenixkey/login
Content-Type: application/json

{
  "sessionToken": "<session_token từ PhoenixKey QR login>"
}
```

`session_token` là JWT ký bằng Ed25519 (`alg=EdDSA`, `kid=phoenixkey-ed25519-1`), TTL 24 giờ.

**Response (bọc trong envelope `.data`):**
```json
{
  "data": {
    "accessToken": "...",
    "refreshToken": "...",
    "userDid": "did:phoenix:<slot>:<hash>"
  },
  "message": "...",
  "statusCode": 201,
  "timestamp": "..."
}
```

> **Lưu ý:** mọi response REST đều bị `TransformInterceptor` bọc thành `{ data, message, statusCode, timestamp }`. Payload thật nằm dưới `.data`.

| Lỗi | Nguyên nhân |
|---|---|
| 503 | `PHOENIXKEY_ENABLED=false` trên server — tính năng bị tắt server-side |
| 503 | PhoenixKey upstream không truy cập được |
| 401 | Token sai hoặc hết hạn (chỉ khi flag đã bật) |
| 429 | Vượt rate limit (5 lần/60 giây) |

> Khi gặp 503, đây là lỗi **cấu hình server** — không phải do token client. Liên hệ team ProofChat để bật staging.

### 1.2 Refresh token

```
POST /auth/refresh
Content-Type: application/json

{ "refreshToken": "..." }
```

**Response `.data`:** `{ "accessToken": "...", "refreshToken": "..." }` (token rotation — refreshToken cũ hết hiệu lực).

### 1.3 Đăng xuất

```
POST /auth/logout
```

Không cần `Authorization` header — endpoint là `@Public()`.

---

## 2. Hội thoại (Conversations)

Mọi request REST cần header `Authorization: Bearer <accessToken>`.

### 2.1 Danh sách hội thoại

```
GET /conversations?type=DIRECT&skip=0&take=50
```

**Response `.data`:** `{ data: Conversation[], total: number }` — mảng nằm ở `resp.data.data`.

```json
{
  "data": {
    "data": [
      {
        "id": "conv_...",
        "title": "Tên nhóm",
        "type": "DIRECT",
        "proposalId": null,
        "avatar": null,
        "createdAt": "2026-07-01T00:00:00Z",
        "participants": [...],
        "_count": { "messages": 5 }
      }
    ],
    "total": 1
  }
}
```

> **Lưu ý:** conversation object **không có** `ownerId` hoặc `memberCount`. Số tin nhắn ở `_count.messages`, danh sách thành viên ở `participants[]`.

Các endpoint khác:
```
GET  /conversations/ids          → mảng conversation id
GET  /conversations/:id          → chi tiết + participants[]
POST /conversations              → tạo hội thoại mới
```

**Tạo hội thoại:**
```json
POST /conversations
{
  "type": "DIRECT",
  "participantIds": ["did:phoenix:...", "did:phoenix:..."],
  "title": "Tên nhóm (bỏ qua nếu DIRECT)",
  "avatar": "https://... (tùy chọn)"
}
```

---

## 3. Tin nhắn (Messages)

> **Lưu ý quan trọng:** tin nhắn ProofChat là ciphertext E2EE. Gửi và nhận tin thật yêu cầu MLS crypto stack (xem §6). Để test tích hợp mà chưa có crypto stack, xem §7 (Scope v2.1).

### 3.1 Lấy lịch sử tin nhắn

```
GET /conversations/:id/messages?deviceId=<deviceId>&limit=15&offset=0
```

Response `.data` là mảng tin nhắn — mỗi tin có `variants[]` (ciphertext theo thiết bị).

### 3.2 Gửi tin nhắn (qua REST)

```
POST /messages
{
  "id": "<uuid>",
  "conversationId": "conv_...",
  "senderId": "did:phoenix:...",
  "createdAt": <unix ms — do client tạo, server không ghi đè>,
  "variants": [
    {
      "senderDeviceId": "...",
      "targetDeviceId": "...",
      "encryptedContent": {
        "opkId": "...",
        "type": 1,
        "body": "<base64 ciphertext>"
      }
    }
  ]
}
```

> `variants[].encryptedContent` là `SignalTransportPayload` — cần đủ `senderDeviceId`, `targetDeviceId`, `encryptedContent`. Payload thiếu field sẽ bị 400 (server dùng `whitelist: true`).

### 3.3 Tính năng bổ sung

| Tính năng | Endpoint |
|---|---|
| Reaction | `POST /messages/:id/reactions` `{ "emoji": "👍" }` |
| Ghim tin nhắn | `POST /conversations/:id/pins` `{ "messageId": "..." }` |
| Bình chọn (poll) | `POST /polls/vote` `{ "pollId": "...", "optionIndex": 0 }` |
| Tin nhắn hẹn giờ | `POST /conversations/:id/scheduled` |
| Lưu tin nhắn | `POST /messages/:id/saved` |
| Xóa tin nhắn | `POST /messages/:id/delete` `{ "forEveryone": true }` |
| Tìm người dùng | `GET /users/search?q=<did hoặc username>` |

---

## 4. WebSocket (Realtime)

Namespace `/chat`, xác thực bằng ProofChat `accessToken`.

### 4.1 Kết nối

```javascript
import { io } from 'socket.io-client'

const socket = io("wss://api.proofchat.me/chat", {
  path: "/ws/socket.io/",          // bắt buộc — nginx định tuyến /ws/ → WS:8090
  auth: { token: accessToken },
  transports: ["websocket"],
  reconnection: true,
})
```

Hoặc query param: `?token=<accessToken>`.

> **Token WS = ProofChat `accessToken`** — lấy từ `POST /auth/phoenixkey/login`. KHÔNG dùng `session_token` gốc của PhoenixKey (khác secret, sẽ fail verify).

> **Tại sao cần `path`:** nginx route `location /ws/` → WS:8090 và strip prefix `/ws`. Nếu không set `path`, socket.io gọi `/socket.io/` mặc định → nginx route nhầm sang FE Next.js. Cloudflare không cần thay đổi — WS upgrade đã được nginx xử lý qua `api.proofchat.me`.

### 4.2 Events — emit từ client

| Event | Payload | Mô tả |
|---|---|---|
| `chat.room.join` | `{ roomId: conversationId }` | Vào phòng chat |
| `contract:message.send` | `{ id, roomId, conversationId, timestamp, mimeType, type, variants[] }` | Gửi tin nhắn |
| `contract:message.typing` | `{ roomId: conversationId, isTyping: true\|false }` | Trạng thái đang gõ |
| `contract:message.read` | `{ jobId: conversationId, messageId }` | Đánh dấu đã đọc |
| `mls:sync.epoch` | `MLSEpochSyncRecord` | Đồng bộ epoch MLS |

### 4.3 Events — nhận từ server

| Event | Payload | Mô tả |
|---|---|---|
| `contract:message.new` | message object | Tin nhắn mới (ciphertext) |
| `contract:message.typing` | `{ userId, isTyping }` | Người khác đang gõ |
| `contract:message.read` | `{ messageId, readBy }` | Tin nhắn đã được đọc |
| `contract:message.pinned` | `{ roomId, messageId, pinnedBy }` | Tin nhắn được ghim |
| `contract:message.unpinned` | `{ roomId, messageId }` | Bỏ ghim |
| `mls:sync.epoch` | `MLSEpochSyncRecord` | MLS key rotation event |
| `mls:device.joined` | `{ deviceId, userDid }` | Thiết bị mới tham gia |
| `error:auth` | `{ message }` | Token hết hạn — refresh và reconnect |

> **Chưa implement** (không emit trong namespace `/chat`): `chat:message.delivered`, `chat:message.reaction.added`, `presence:update`.

---

## 5. Feature Flag (Vận hành độc lập)

ProofChat hỗ trợ chạy song song mode mock và mode thật qua biến môi trường.  
Khi flag tắt, toàn bộ module chạy với mock data — không cần backend online.

```
PROOFCHAT_BACKEND_ENABLED=true    # false = chạy mock
PROOFCHAT_API_URL=https://api.proofchat.me/api/v1
PROOFCHAT_WS_URL=wss://api.proofchat.me
PROOFCHAT_WS_PATH=/ws/socket.io/
```

**Nguyên tắc bắt buộc:** mọi call tới ProofChat backend phải được guard bởi flag này.  
App phải chạy được khi flag tắt hoặc backend down.

---

## 6. E2EE — MLS Crypto Stack

Gửi và nhận nội dung thật yêu cầu:
- Thư viện MLS (Messaging Layer Security, RFC 9420)
- Ciphersuite: `MLS_128_DHKEMP256_AES128GCM_SHA256_P256`
- Mỗi thiết bị có KeyPackage đã đăng ký qua `POST /mls/keypackage`
- Mã hóa payload → `variants[]` trước khi gửi
- Giải mã `variants[]` sau khi nhận

**MLS endpoints:**

| Endpoint | Mô tả |
|---|---|
| `POST /mls/keypackage` | Đăng ký KeyPackage cho device |
| `GET /mls/keypackage/status` | Kiểm tra trạng thái |
| `POST /mls/keypackages/batch` | Lấy KeyPackage nhiều user |
| `GET /mls/keypackages/room/:conversationId` | KeyPackages theo room |
| `POST /mls/epoch-sync` | Lưu epoch sync record |
| `GET /mls/epoch-sync/:conversationId/current` | Epoch hiện tại |

> Nếu chưa có crypto stack trên platform của mình, xem §7 để test tích hợp mà không cần E2EE.

---

## 7. Scope v2.1 — Testable mà không cần E2EE

Đây là mức tối thiểu để test tích hợp ProofChat mà **không cần MLS crypto stack**:

| # | Việc | Cách |
|---|---|---|
| 1 | Auth bridge | `POST /auth/phoenixkey/login` với session_token |
| 2 | Danh sách hội thoại thật | `GET /conversations` → `resp.data.data[]` |
| 3 | Metadata phòng chat | `GET /conversations/:id` → tên, participants[], loại |
| 4 | Nhận tin realtime (ciphertext) | WS `contract:message.new` → hiển thị "Tin nhắn mã hóa 🔒" |
| 5 | Typing indicator | WS `contract:message.typing` emit + listen |
| 6 | Đánh dấu đã đọc | WS `contract:message.read` emit |

**Chưa làm được ở v2.1** (chờ crypto stack):
- Gửi tin nhắn thật (cần encode `variants[]`)
- Giải mã nội dung tin nhắn nhận được
- Khởi tạo KeyPackage

---

## 8. Error Codes

| HTTP | Ý nghĩa |
|---|---|
| 400 | Thiếu hoặc sai format payload (validation failed) |
| 401 | Token hết hạn hoặc không hợp lệ → refresh rồi retry |
| 403 | Không có quyền (không phải thành viên conversation) |
| 404 | Resource không tồn tại |
| 429 | Rate limit — thử lại sau `Retry-After` header |
| 503 | Feature bị tắt server-side (`PHOENIXKEY_ENABLED=false`) hoặc PhoenixKey upstream không truy cập được |

---

## 9. Lưu ý khi tích hợp

- `createdAt` trong tin nhắn phải do **client tạo** (Unix ms) — server không ghi đè (cần thiết cho ZKP MerkleLeaf).
- `variants[]` chứa ciphertext riêng cho từng thiết bị — không chia sẻ chéo.
- Khi nhận `error:auth` qua WS → gọi `POST /auth/refresh` → reconnect với token mới.
- PhoenixKey base URL (để verify token): `https://api.phoenixkey.me`
- PhoenixKey JWKS: `https://api.phoenixkey.me/.well-known/jwks.json` (không có prefix `/api/v1`)
- Mọi response REST bọc trong `{ data, message, statusCode, timestamp }` — payload thật ở `.data`.

---

## 10. SuperApp — Checklist tích hợp nhanh

Dành riêng cho team SuperApp (Aladin). Các file service đã có sẵn trong repo SuperApp, chỉ cần wire và bật flag.

### Env vars cần thêm vào `.env`

```
PROOFCHAT_BACKEND_ENABLED=true
PROOFCHAT_API_URL=https://api.proofchat.me/api/v1
PROOFCHAT_WS_URL=wss://api.proofchat.me
PROOFCHAT_WS_PATH=/ws/socket.io/
```

> Khi `PROOFCHAT_BACKEND_ENABLED=false` (mặc định) → module chạy mock như cũ — offline-first đảm bảo.

### Files đã có, không cần viết lại

| File | Vai trò |
|---|---|
| `src/services/proofchat-api.ts` | REST client + `isProofChatBackendEnabled()` + token refresh |
| `src/services/proofchatAuthBridge.ts` | Lấy `sessionToken` từ PhoenixKey → login ProofChat |
| `src/types/env.d.ts` | Đã khai báo `PROOFCHAT_BACKEND_ENABLED`, `PROOFCHAT_API_URL` |

### Việc cần làm (SuperApp team)

1. **Thêm env vars** ở trên vào `.env` / CI secrets
2. **Gọi auth bridge khi module mount** — `proofchatAuthBridge.connectProofChat()` trong `ProofChatHomeScreen`
3. **Thay MOCK_ROOMS** → `proofChatApi.conversations.list({ take: 50 })` (nhớ đọc `resp.data.data`)
4. **Tạo `src/services/proofchatSocket.ts`** — kết nối WS namespace `/chat` (xem §4.1), lắng nghe `contract:message.new`
5. **Xóa escrow code** khỏi `ProofChatHomeScreen`, `ChatScreen`, `proofchatSlice` (Chat không có escrow)

### Trạng thái server (cần confirm từ team ProofChat)

- `PHOENIXKEY_ENABLED` trên BE staging: đang chờ bật — [ProofChat/BE#13](https://github.com/ProofChat/BE/issues/13)
- WS Option A (`ws.proofchat.me`) hoặc Option B (`api.proofchat.me` + `path`): xem [ProofChat/BE#15](https://github.com/ProofChat/BE/pull/15)

---

## Liên hệ

Gặp lỗi hoặc cần thêm endpoint → mở issue tại [github.com/ProofChat/Docs](https://github.com/ProofChat/Docs).
