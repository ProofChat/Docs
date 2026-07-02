# ProofChat — Integration Guide

> Phiên bản: 2026-07-02  
> Dành cho: team bên ngoài tích hợp ProofChat vào app (SuperApp, AladinWork, hoặc platform khác).

---

## Tổng quan

ProofChat là hệ thống nhắn tin bảo mật đầu cuối (E2EE) dùng giao thức MLS.  
Server **không bao giờ thấy nội dung plaintext** — chỉ lưu ciphertext theo từng thiết bị.

| Thành phần | URL |
|---|---|
| REST API | `https://api.proofchat.me/api/v1` |
| WebSocket | `wss://api.proofchat.me/ws/` |

> **Lưu ý WebSocket path:** xác nhận path `/ws/` với team ProofChat trước khi deploy lên production. Có thể thay đổi.

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

**Response:**
```json
{
  "accessToken": "...",
  "refreshToken": "...",
  "userDid": "did:phoenix:<slot>:<hash>"
}
```

| Lỗi | Nguyên nhân |
|---|---|
| 401 | Token sai hoặc hết hạn |
| 429 | Vượt rate limit (5 lần/60 giây) |

### 1.2 Refresh token

```
POST /auth/refresh
Content-Type: application/json

{ "refreshToken": "..." }
```

**Response:** `{ "accessToken": "...", "refreshToken": "..." }` (token rotation — refreshToken cũ hết hiệu lực).

### 1.3 Đăng xuất

```
POST /auth/logout
Authorization: Bearer <accessToken>
```

---

## 2. Hội thoại (Conversations)

Mọi request REST cần header `Authorization: Bearer <accessToken>`.

### 2.1 Danh sách hội thoại

```
GET /conversations?type=DIRECT&skip=0&take=50
```

**Response:** mảng conversation object:
```json
[
  {
    "id": "conv_...",
    "title": "Tên nhóm",   // null nếu là hội thoại 1-1
    "type": "DIRECT" | "GROUP" | "JOB_NEGOTIATION",
    "ownerId": "did:phoenix:...",
    "memberCount": 2,
    "createdAt": "2026-07-01T00:00:00Z"
  }
]
```

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

Response là mảng tin nhắn — mỗi tin có `variants[]` (ciphertext theo thiết bị).

### 3.2 Gửi tin nhắn (qua REST)

```
POST /messages
{
  "id": "<uuid>",
  "conversationId": "conv_...",
  "senderId": "did:phoenix:...",
  "createdAt": <unix ms — do client tạo, server không ghi đè>,
  "variants": [
    { "deviceId": "...", "ciphertext": "..." }
  ]
}
```

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

Namespace `/chat` — xác thực bằng JWT.

### 4.1 Kết nối

```javascript
import { io } from 'socket.io-client'

const socket = io("wss://api.proofchat.me/ws/", {
  auth: { token: accessToken },
  transports: ["websocket"],
  reconnection: true,
})
```

Hoặc dùng query param: `?token=<accessToken>`.

### 4.2 Events — emit từ client

| Event | Payload | Mô tả |
|---|---|---|
| `chat.room.join` | `{ roomId: conversationId }` | Vào phòng chat |
| `contract:message.send` | `{ id, conversationId, timestamp, mimeType, variants[] }` | Gửi tin nhắn |
| `chat:typing` | `{ conversationId, isTyping: true\|false }` | Trạng thái đang gõ |
| `chat:message.read` | `{ messageId, conversationId }` | Đánh dấu đã đọc |

### 4.3 Events — nhận từ server

| Event | Payload | Mô tả |
|---|---|---|
| `contract:message.new` | message object | Tin nhắn mới (ciphertext) |
| `chat:typing` | `{ userId, isTyping }` | Người khác đang gõ |
| `chat:message.read` | `{ messageId, userId }` | Tin nhắn đã được đọc |
| `chat:message.delivered` | `{ messageId }` | Tin nhắn đã giao |
| `chat:message.reaction.added` | `{ messageId, userId, emoji }` | Có reaction mới |
| `chat:message.pinned` | `{ messageId, conversationId }` | Tin nhắn được ghim |
| `mls:sync.epoch` | `{ epoch }` | MLS key rotation event |
| `mls:device.joined` | `{ deviceId, userDid }` | Thiết bị mới tham gia |
| `presence:update` | `{ userId, status }` | Trạng thái online |
| `error:auth` | `{ message }` | Token hết hạn — refresh và reconnect |

---

## 5. Feature Flag (Vận hành độc lập)

ProofChat hỗ trợ chạy song song mode mock và mode thật qua biến môi trường.  
Khi flag tắt, toàn bộ module chạy với mock data — không cần backend online.

```
PROOFCHAT_BACKEND_ENABLED=true    # false = chạy mock
PROOFCHAT_API_URL=https://api.proofchat.me/api/v1
PROOFCHAT_WS_URL=wss://api.proofchat.me/ws/
```

**Nguyên tắc bắt buộc:** mọi call tới ProofChat backend phải được guard bởi flag này.  
App phải chạy được khi flag tắt hoặc backend down.

---

## 6. E2EE — MLS Crypto Stack

Gửi và nhận nội dung thật yêu cầu:
- Thư viện MLS (Messaging Layer Security, RFC 9420)
- Mỗi thiết bị có KeyPackage đã đăng ký qua `POST /mls/key-packages`
- Mã hóa payload → `variants[]` trước khi gửi
- Giải mã `variants[]` sau khi nhận

> Nếu chưa có crypto stack trên platform của mình, xem §7 để test tích hợp mà không cần E2EE.

---

## 7. Scope v2.1 — Testable mà không cần E2EE

Đây là mức tối thiểu để test tích hợp ProofChat mà **không cần MLS crypto stack**:

| # | Việc | Cách |
|---|---|---|
| 1 | Auth bridge | `POST /auth/phoenixkey/login` với session_token |
| 2 | Danh sách hội thoại thật | `GET /conversations` → thay thế mock data |
| 3 | Metadata phòng chat | `GET /conversations/:id` → tên, thành viên, loại |
| 4 | Nhận tin realtime (ciphertext) | WS `contract:message.new` → hiển thị "Tin nhắn mã hóa 🔒" |
| 5 | Typing indicator | WS `chat:typing` emit + listen |
| 6 | Đánh dấu đã đọc | WS `chat:message.read` emit |

**Chưa làm được ở v2.1** (chờ crypto stack):
- Gửi tin nhắn thật (cần encode `variants[]`)
- Giải mã nội dung tin nhắn nhận được
- Khởi tạo KeyPackage

---

## 8. Error Codes

| HTTP | Ý nghĩa |
|---|---|
| 400 | Thiếu hoặc sai format payload |
| 401 | Token hết hạn hoặc không hợp lệ → refresh rồi retry |
| 403 | Không có quyền (không phải thành viên conversation) |
| 404 | Resource không tồn tại |
| 429 | Rate limit — thử lại sau `Retry-After` header |

---

## 9. Lưu ý khi tích hợp

- `createdAt` trong tin nhắn phải do **client tạo** (Unix ms) — server không ghi đè (cần thiết cho ZKP MerkleLeaf).
- `variants[]` chứa ciphertext riêng cho từng thiết bị — không chia sẻ chéo.
- Khi nhận `error:auth` qua WS → gọi `POST /auth/refresh` → reconnect với token mới.
- PhoenixKey base URL (để verify token): `https://api.phoenixkey.me`
- PhoenixKey JWKS: `https://api.phoenixkey.me/.well-known/jwks.json` (không có prefix `/api/v1`)

---

## Liên hệ

Gặp lỗi hoặc cần thêm endpoint → mở issue tại [github.com/ProofChat/Docs](https://github.com/ProofChat/Docs).
