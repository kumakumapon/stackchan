# StackChan Architecture

## System Context Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                         Internet                              │
└──────────────────────────────────────────────────────────────┘
           │                          │                  │
           │                          │                  │
    ┌──────v──────┐         ┌─────────v────┐    ┌──────v──────┐
    │   App Users  │         │   M5Stack    │    │ XiaoZhi     │
    │  (Flutter)   │         │ Auth Service │    │ AI Service  │
    │ iOS/Android  │         │    (Remote)  │    │             │
    └──────┬──────┘         └─────────┬────┘    └──────┬──────┘
           │                          │                 │
           │                    HTTP  │                 │
           │    HTTP/WS              │            HTTP  │
           │    JWT Auth             │            API   │
           │                          │                 │
           └──────────────┬───────────┴─────────────────┘
                          │
                    ┌─────v──────────┐
                    │  StackChan     │
                    │  Server        │
                    │  (Go, Port     │
                    │   12800)       │
                    └─────┬──────────┘
                          │
                    ┌─────v──────────┐
                    │  MySQL 8.0+    │
                    │  (Database)    │
                    └────────────────┘

    ┌──────────────────────────┐
    │  StackChan Hardware      │
    │  (ESP32-S3 + CoreS3)     │
    │                          │
    │  ├─ Dual Servo           │
    │  ├─ 12 RGB LEDs          │
    │  ├─ 2.0" Touch Display   │
    │  ├─ Microphone x2        │
    │  ├─ Speaker (1W)         │
    │  ├─ Camera (0.3MP)       │
    │  ├─ Wi-Fi / BLE          │
    │  └─ NFC Module           │
    │                          │
    │  Firmware: ESP-IDF 5.5.4 │
    │  (C/C++)                 │
    └────────┬─────────────────┘
             │
             │ HTTP REST
             │ RSA Auth
             │ WebSocket
             │
    ┌────────v─────────────┐
    │ StackChan Server     │
    │ /stackChan/* routes  │
    └──────────────────────┘

    ┌──────────────────────────┐
    │  ESPNow Remote Control   │
    │  (ESP32)                 │
    │                          │
    │  Wireless Protocol:      │
    │  - ESPNow (Direct comm)  │
    │  - Low Latency           │
    │  - No Wi-Fi required     │
    └──────────────────────────┘
```

## Server Components (Internal Architecture)

### 1. Layered Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   HTTP API Layer                        │
│  (/stackChan/v2, /stackChan/*, /admin/stackChan/*)    │
├─────────────────────────────────────────────────────────┤
│                Middleware Layer                         │
│  ├─ CORS                                                │
│  ├─ TokenAuthMiddleware (Device RSA)                   │
│  ├─ V2TokenAuthMiddleware (App JWT)                    │
│  └─ AdminTokenAuthMiddleware (Admin JWT)               │
├─────────────────────────────────────────────────────────┤
│             Controller Layer (HTTP Handler)             │
│  ├─ user_controller        (Login/Register)            │
│  ├─ device_controller      (Bind/Unbind)               │
│  ├─ dance_controller       (Create/Update/Delete)      │
│  ├─ post_controller        (Post/Comment)              │
│  ├─ pano_controller        (Panorama)                  │
│  ├─ file_controller        (Upload/Serve)              │
│  ├─ admin_controller       (App Store)                 │
│  └─ xiaozhi_controller     (AI Agent)                  │
├─────────────────────────────────────────────────────────┤
│             Service Layer (Business Logic)              │
│  ├─ service/user.go        (Auth, JWT generation)      │
│  ├─ service/device.go      (Device lifecycle)          │
│  ├─ service/dance.go       (Dance CRUD)                │
│  ├─ service/file.go        (File handling)             │
│  ├─ service/agent.go       (XiaoZhi integration)       │
│  └─ service/admin_user.go  (Admin auth)                │
├─────────────────────────────────────────────────────────┤
│            Data Access Layer (DAO, GoFrame)             │
│  ├─ dao.User               (Generated ORM)             │
│  ├─ dao.Device                                         │
│  ├─ dao.DeviceDance                                    │
│  ├─ dao.DevicePost                                     │
│  ├─ dao.DevicePostComment                              │
│  ├─ dao.DevicePano                                     │
│  ├─ dao.DeviceFriend                                   │
│  └─ dao.AppStore                                       │
├─────────────────────────────────────────────────────────┤
│                  Database (MySQL)                       │
│  ├─ user                   (PK: uid)                    │
│  ├─ device                 (PK: mac)                    │
│  ├─ device_dance           (FK: mac)                    │
│  ├─ device_post            (FK: mac)                    │
│  ├─ device_post_comment    (FK: post_id, mac)          │
│  ├─ device_pano            (FK: mac)                    │
│  ├─ device_friend          (PK: mac_a, mac_b)          │
│  └─ app_store              (Soft delete)               │
└─────────────────────────────────────────────────────────┘

Special Components:
├─ WebSocket Layer (internal/web_socket/)
│  └─ Connection pool management, binary message forwarding
├─ Boot Layer (internal/boot/)
│  └─ Heartbeat scheduler (5s), Connection cleanup (15s)
├─ Utility Layer (utility/)
│  └─ RSA encryption/decryption
├─ XiaoZhi Integration (internal/xiaozhi/)
│  └─ API client for AI agent operations
└─ Packed Assets (internal/packed/)
   └─ Embedded Flutter Web management console
```

### 2. Request Flow Diagram

#### 2.1 App User Login

```
Request
  ├─ POST /stackChan/v2/user/login
  ├─ Body: { username, password }
  └─ Header: Content-Type: application/json

↓ (No auth middleware bypass for /login)

Controller: user_controller.go::V2Login()
  │
  ├─ Validation: username/password not empty
  │
  └─→ service.Login(ctx, req)
      │
      ├─→ callRemoteLogin()
      │   │ HTTP POST to m5stack.loginUrl
      │   │ (external M5Stack Auth Service)
      │   │
      │   ├─ Parse RemoteLoginResp
      │   ├─ Check status.code == "ok"
      │   └─ Extract uid, username, etc.
      │
      ├─→ saveUserToLocal(ctx, resp)
      │   │ entity.User from remote response
      │   └─ dao.User.Save() → MySQL INSERT/UPDATE
      │
      ├─→ generateToken(ctx, remoteResp.Response.Uid)
      │   │ JWT payload:
      │   │ ├─ id: uid
      │   │ ├─ iss: m5stack.issuer
      │   │ ├─ aud: m5stack.audience
      │   │ ├─ exp: now + 365days
      │   │ ├─ iat: now
      │   │ └─ jti: unique token ID
      │   │
      │   └─ SignedString(jwt_secret) → token string
      │
      └─ Return LoginRes { token }

↓

Response
  ├─ Status: 200
  ├─ Body:
  │  {
  │    "code": 0,
  │    "message": "",
  │    "data": {
  │      "token": "eyJhbGc..."
  │    }
  │  }
  └─ (Client stores token in local storage)
```

#### 2.2 Device Bind

```
Request
  ├─ POST /stackChan/v2/device/bind
  ├─ Header: token: Bearer <user-jwt>
  ├─ Body: { mac: "AA:BB:CC:DD:EE:FF" }
  └─ Content-Type: application/json

↓

Middleware: V2TokenAuthMiddleware()
  │
  ├─ Extract "token" header, strip "Bearer "
  │
  ├─→ jwt.Parse(tokenString, jwtSecret)
  │   │ Verify HMAC signature
  │   │ Check expiration
  │   └─ Extract claims
  │
  ├─ Extract uid from claims["id"]
  │
  └─ ctx.SetCtxVar(model.Uid, uid)

↓

Controller: device_controller.go::V2Bind()
  │
  ├─ Extract uid from ctx
  │
  ├─ Validation: mac format
  │
  └─→ service.BindDevice(ctx, uid, mac)
      │
      ├─→ dao.Device.Ctx(ctx).Where("mac = ?", mac).One()
      │   │ (Check if device exists)
      │   └─ If exists, update uid
      │   │ If not exists, insert new device record
      │
      └─ Return Device { mac, name, uid, bind_time }

↓

Response
  ├─ Status: 200
  ├─ Body:
  │  {
  │    "code": 0,
  │    "message": "",
  │    "data": {
  │      "mac": "AA:BB:CC:DD:EE:FF",
  │      "name": "...",
  │      "uid": 12345,
  │      "bind_time": "2026-09-12 ..."
  │    }
  │  }
  └─ (App can now control this device)
```

#### 2.3 Device REST API (RSA-Authenticated)

```
Request
  ├─ POST /stackChan/device
  ├─ Header: Authorization: <rsa-mac-token>
  │  (RSA-encrypted: "MAC|nonce|unix_timestamp")
  │
  └─ Body: { name: "My StackChan" }

↓

Middleware: TokenAuthMiddleware()
  │
  ├─→ web_socket.GetMac(r)
  │   │
  │   ├─ Extract Authorization header
  │   ├─ Base64 decode
  │   │
  │   ├─→ utility.RSADecrypt(decodedToken)
  │   │   │ RSA-OAEP-SHA256 decrypt
  │   │   │ using rsa.server.private
  │   │   │
  │   │   └─ Plaintext: "MAC|nonce|timestamp"
  │   │
  │   ├─ Split by "|"
  │   ├─ Extract mac (parts[0])
  │   ├─ Extract timestamp (parts[2])
  │   │
  │   ├─ Validate timestamp within ±10 seconds
  │   └─ Return mac
  │
  ├─ ctx.SetCtxVar(model.Mac, mac)
  │
  └─ Next middleware

↓

Controller: device_controller.go::Create()
  │
  ├─ Extract mac from ctx
  │
  ├─ Validation: device name
  │
  └─→ service.CreateDevice(ctx, mac, name)
      │
      ├─→ dao.Device.Ctx(ctx).Save({ mac, name })
      │   │ MySQL INSERT (device table)
      │   │ If exists, soft-update
      │   │
      │   └─ Return device record
      │
      └─ Return response

↓

Response
  └─ Device info
```

### 3. WebSocket Architecture

```
┌─ WebSocket Connection Pool ─────────────────────┐
│                                                  │
│  stackChanClientPool (Device clients)            │
│  ├─ mac1 → *websocket.Conn                       │
│  ├─ mac2 → *websocket.Conn                       │
│  └─ ...                                          │
│                                                  │
│  appClientPool (App clients)                     │
│  ├─ app_device_id_1 → *websocket.Conn            │
│  ├─ app_device_id_2 → *websocket.Conn            │
│  └─ ...                                          │
│                                                  │
│  sync.Map for lock-free concurrent access       │
└──────────────────────────────────────────────────┘

Message Protocol (Binary):
┌────────────────────────────────┐
│ 1 Byte: Message Type (msgType) │
├────────────────────────────────┤
│ 4 Bytes: Payload Length (BE)   │
├────────────────────────────────┤
│ N Bytes: Payload Data          │
└────────────────────────────────┘

Message Types:
├─ 0x01: Opus (Audio frame from/to device)
├─ 0x02: Jpeg (Image frame from device)
├─ 0x03: ControlAvatar (Eye/mouth control)
├─ 0x04: ControlMotion (Servo motion)
├─ 0x05: OnCamera (Turn on camera)
├─ 0x06: OffCamera (Turn off camera)
├─ 0x07: TextMessage (Text chat)
├─ 0x09: RequestCall (Initiate call)
├─ 0x0A: RefuseCall (Decline call)
├─ 0x0B: AgreeCall (Accept call)
├─ 0x0C: HangupCall (End call)
├─ 0x0D: UpdateDeviceName (Device rename)
├─ 0x0E: GetDeviceName (Query device name)
├─ 0x10: ping (Server heartbeat)
├─ 0x11: pong (Client response)
├─ 0x12: OnPhoneScreen (Start phone projection)
├─ 0x13: OffPhoneScreen (Stop phone projection)
├─ 0x14: Dance (Dance playback)
├─ 0x15: GetAvatarPosture (Query posture state)
├─ 0x16: DeviceOffline (Notification)
├─ 0x17: DeviceOnline (Notification)
├─ 0x18: OnAudio (Start audio stream)
├─ 0x19: OffAudio (Stop audio stream)
└─ 0x1A: AimedTakePhoto (Direct photo capture)

Handler Flow:
┌────────────────────────────────────┐
│ /stackChan/ws upgrade              │
├────────────────────────────────────┤
│ Handler() entry point              │
│ ├─ Upgrade to WebSocket (Gorilla)  │
│ ├─ Extract deviceType query param  │
│ ├─ Extract Authorization header    │
│ ├─→ GetMac() for auth              │
│ ├─ Register to connection pool     │
│ └─ Launch readClientMessage loop   │
├────────────────────────────────────┤
│ readClientMessage loop             │
│ ├─ Read binary frame               │
│ ├─ Parse msgType + payload         │
│ ├─→ handleMessage(msgType, data)   │
│ │   ├─ Route to handler by type    │
│ │   └─ Some broadcast, some 1-to-1 │
│ └─ Continue until connection close │
└────────────────────────────────────┘

Broadcast Examples:
├─ 0x01 (Opus): Device → All connected apps for that device
├─ 0x02 (Jpeg): Device → All connected apps for that device
├─ 0x17 (DeviceOnline): Device → Broadcast to certain pool
└─ 0x16 (DeviceOffline): Notify other devices

Heartbeat Mechanism (boot/socket_task.go):
├─ Every 5 seconds: Send 0x10 (ping) to all connections
├─ Connection sends 0x11 (pong) response
├─ Every 15 seconds: Remove stale (non-responsive) connections
└─ Keep connection alive and detect dead clients
```

### 4. Authentication & Authorization

```
Three-tier Authentication Model:

┌─ Device Authentication (RSA) ──────────────────┐
│                                                │
│  Client generates:                             │
│  plaintext = "MAC|nonce|unix_timestamp"        │
│  cipher = RSA-OAEP-SHA256(plaintext,           │
│            rsa.server.public)                  │
│  Authorization = Base64(cipher)                │
│                                                │
│  Server verifies:                              │
│  1. Base64 decode                              │
│  2. RSA-OAEP-SHA256 decrypt                    │
│     using rsa.server.private                   │
│  3. Extract MAC (parts[0])                     │
│  4. Validate timestamp ±10 seconds             │
│  5. Store MAC in context                       │
│                                                │
│  ✓ No user context needed                      │
│  ✓ MAC-based resource isolation                │
└────────────────────────────────────────────────┘

┌─ App User Authentication (JWT) ────────────────┐
│                                                │
│  Client sends:                                 │
│  token: Bearer <jwt>                           │
│                                                │
│  JWT Payload:                                  │
│  {                                             │
│    "id": uid (user ID),                        │
│    "iss": "stackchan-server",                  │
│    "aud": "stackchan-app",                     │
│    "exp": timestamp,                           │
│    "iat": timestamp,                           │
│    "jti": unique_token_id                      │
│  }                                             │
│                                                │
│  Server verifies:                              │
│  1. Strip "Bearer " prefix                     │
│  2. jwt.Parse() with jwt_secret                │
│  3. Verify HMAC signature                      │
│  4. Check expiration                           │
│  5. Extract uid from claims["id"]              │
│  6. Store uid in context                       │
│                                                │
│  ✓ Stateless token (no server-side store)      │
│  ✓ User-based resource isolation               │
│  ✓ 365-day expiration                          │
└────────────────────────────────────────────────┘

┌─ Admin Authentication (JWT) ───────────────────┐
│                                                │
│  Client sends:                                 │
│  Authorization: <jwt>  (no "Bearer " prefix)   │
│                                                │
│  JWT Payload:                                  │
│  {                                             │
│    "username": admin_username,                 │
│    ...other claims                             │
│  }                                             │
│                                                │
│  Server verifies:                              │
│  1. jwt.Parse() with jwt_secret                │
│  2. Extract username from claims               │
│  3. Store username in context                  │
│                                                │
│  ✓ 24-hour expiration (shorter than app)       │
│  ✓ Username-based access                       │
└────────────────────────────────────────────────┘

Resource Access Control:
├─ Devices: Controlled by MAC or uid
│  ├─ Device-side API: uses MAC from token
│  ├─ App-side API: can only access own devices
│  │  (device.uid == user_uid from token)
│  └─ Admin API: no resource-level check (implicit)
│
├─ Dances: Child of device
│  ├─ Create/Update/Delete: Verify device ownership
│  ├─ Read: Public (any authenticated client)
│  └─ Device-side: Create on own MAC only
│
├─ Posts/Comments: Created by device
│  ├─ Posted by: device.mac
│  ├─ Read: Public (paginated)
│  ├─ Delete: Device owner or app owner of device
│  └─ Comment: Any device can add
│
└─ Files: Role-based upload
   ├─ Device upload: /stackChan/uploadFile
   ├─ App upload: via device bind
   ├─ Admin upload: /admin/stackChan/uploadFile
   └─ All users can read /file/*
```

## Database Schema

### Entity Relationship Diagram

```
┌──────────────┐
│     user     │
├──────────────┤
│ uid (PK)     │ ← Foreign Key
│ username     │
│ display_name │
│ ...          │
└──────┬───────┘
       │
       │ 1:N
       │
       v
┌──────────────────────────────┐
│         device               │
├──────────────────────────────┤
│ mac (PK)                     │
│ name                         │
│ uid (FK → user.uid, NULL)    │
│ bind_time                    │
└───────┬──────────┬──────┬────┘
        │          │      │
    1:N │          │ 1:N  │ 1:N
        │          │      │
  ┌─────v──┐  ┌────v──┐  ┌v──────────────┐
  │  dance  │  │ post  │  │  pano         │
  ├────────┤  ├───────┤  ├───────────────┤
  │mac (FK)│  │mac(FK)│  │mac (FK)       │
  │name    │  │text   │  │url            │
  │data(J) │  │image  │  │created_at     │
  │musicUrl│  └───┬───┘  └───────────────┘
  └────────┘      │
                  │ 1:N
                  │
                  v
           ┌─────────────┐
           │ comment     │
           ├─────────────┤
           │post_id (FK) │
           │mac (FK)     │
           │content      │
           └─────────────┘

┌──────────────────┐
│   device_friend  │
├──────────────────┤
│ mac_a (PK, FK)   │
│ mac_b (PK, FK)   │
└──────────────────┘
(Symmetric friendship: both directions stored)

┌──────────────────┐
│   app_store      │
├──────────────────┤
│ id (PK)          │
│ app_name         │
│ app_icon_url     │
│ description      │
│ firmware_url     │
│ is_deleted (soft)│
└──────────────────┘
```

### Key Constraints

```
Device as Central Entity:
├─ device.mac: PK (17-char string, e.g., "AA:BB:CC:DD:EE:FF")
├─ device.uid: FK → user.uid (nullable, can be unbound)
├─ device_dance.mac: FK → device.mac
├─ device_post.mac: FK → device.mac
├─ device_pano.mac: FK → device.mac
├─ device_friend.mac_a/b: FK → device.mac (symmetric)
└─ device_post_comment.mac: FK → device.mac

User as Identity:
├─ user.uid: PK (bigint, remote UID)
├─ user.username: UNIQUE
├─ device.uid: FK → user.uid
└─ App users accessed via uid

Cascade Rules:
├─ device deleted → cascade to dance, post, pano, comments
├─ user deleted → device.uid → NULL (cascade set null)
└─ post deleted → comments deleted (cascade delete)
```

## Data Flow Across System

### Login & Token Flow

```
App              Server                          M5Stack Auth Service
 │                │                                        │
 ├─ User input ──>│ POST /stackChan/v2/user/login         │
 │                │                                        │
 │                ├────────────── POST ─────────────────>│
 │                │        (username, password)           │
 │                │                                        │
 │                │<────────────── Response ──────────────┤
 │                │      (uid, username, ...)             │
 │                │                                        │
 │                ├─[Save to user table]                  │
 │                │                                        │
 │                ├─[Generate JWT]                        │
 │                │  Claims: id=uid, iss, aud, exp, iat  │
 │                │                                        │
 │<─ JWT token ───┤                                        │
 │                │                                        │
 ├─[Store in app]─┤                                        │
 │                │                                        │
```

### Device Control Flow (WebSocket)

```
App (UI)              App (Service)          Server            Device (Firmware)
  │                        │                    │                      │
  ├─[Eye animation]─>      │                    │                      │
  │                        ├─ 0x03 frame ─────>│                       │
  │                        │  (msgType=3)       │                       │
  │                        │                    ├─[Lookup device pool]  │
  │                        │                    ├─ Forward to device ──>│
  │                        │                    │                       │
  │                        │                    │                  [Servo]
  │                        │                    │                  [Motor]
  │                        │                    │                  
  │                        │<─ 0x02 (Jpeg) ────┤ (Camera image)    [Camera]
  │                        │   (msgType=2)      │                       │
  │<─────────────────────────── Image display  │                       │
  │                        │                    │                       │
  │<─[Realtime update]─────┤                    │                       │
```

### File Upload Flow

```
Client (App/Device)    Server                  Disk
       │                 │                       │
       ├─ POST /uploadFile ─>│                   │
       │  Form: file, name, directory           │
       │                 │                       │
       │                 ├─[Receive multipart]  │
       │                 ├─[Save to disk] ─────>│
       │                 │  file/[directory]/   │
       │                 │   [name]              │
       │                 │                       │
       │<─ Response ─────┤                       │
       │  { path: "file/posts/photo.jpg" }      │
       │                 │                       │
       └─ URL in DB ────>│ (store path in DB)   │
         (e.g., content_image in device_post)

[Later]
       │                 │                       │
       ├─ GET /file/posts/photo.jpg ────>│      │
       │                 │                       │
       │                 ├─[Check exists] ─────>│
       │                 │                       │
       │<─ File content ──────────────────────────<─
       │                 │                       │
```

## Deployment Architecture

```
┌─ Production Deployment ──────────────────────────────┐
│                                                      │
│  ┌─ Kubernetes Pod ───────────────────────────┐    │
│  │                                             │    │
│  │  stackchan-server container                │    │
│  │  ├─ Binary: stackChan                       │    │
│  │  ├─ Volume: config.yaml (ConfigMap)        │    │
│  │  ├─ Port: 12800 (TCP)                      │    │
│  │  ├─ Environment: DB_LINK, JWT_SECRET, ...  │    │
│  │  └─ Liveness/Readiness probes              │    │
│  │                                             │    │
│  │  Resources:                                 │    │
│  │  ├─ Requests: CPU, Memory                  │    │
│  │  └─ Limits: CPU, Memory                    │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│           │                                         │
│      Service (12800)                                │
│           │                                         │
│      Ingress / LoadBalancer                         │
│           │                                         │
│  ┌────────v──────────────────────────────────┐    │
│  │  External Clients                          │    │
│  │  ├─ App users (HTTPS)                     │    │
│  │  ├─ Devices (HTTPS + WSS)                 │    │
│  │  └─ Admin users (HTTPS)                   │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
│  ┌─ Persistent Storage ─────────────────────┐      │
│  │  ├─ MySQL Pod / RDS                      │      │
│  │  ├─ PVC for /file (uploads)              │      │
│  │  └─ PVC for /logs                        │      │
│  └──────────────────────────────────────────┘      │
│                                                      │
└──────────────────────────────────────────────────────┘

Docker Build:
├─ Binary compile: go build -o stackChan
├─ Dockerfile: manifest/docker/Dockerfile
│  ├─ Base image: Alpine/Ubuntu
│  ├─ Copy binary
│  ├─ Copy config.yaml
│  ├─ Copy web/management (Flutter Web assets)
│  └─ Expose port 12800
└─ Build: docker build -t stackchan-server:v1.0 .
```

## Async & Scheduling

### Boot Layer (internal/boot/)

```
Main server startup:
  │
  ├─→ boot.InitCron()
  │   │
  │   ├─ Cron Job 1: Heartbeat
  │   │  ├─ Interval: Every 5 seconds
  │   │  ├─ Action: Send WebSocket ping to all connections
  │   │  └─ File: internal/boot/socket_task.go
  │   │
  │   └─ Cron Job 2: Cleanup
  │      ├─ Interval: Every 15 seconds
  │      ├─ Action: Remove stale (non-responsive) connections
  │      └─ File: internal/boot/socket_task.go
  │
  └─ Continue server operation
```

### Token Refresh Logic

```
XiaoZhi Token Caching (internal/xiaozhi/):
├─ Token fetched from XiaoZhi API
├─ Cached in memory (not persisted)
├─ Refresh scheduled every 24 hours
├─ Manual refresh via: GET /stackChan/xiaozhi/token/refresh
└─ If cache expires, re-fetch on next request

No background job for token refresh yet
(推測: Manual refresh endpoint likely called from app)
```

## Error Handling

### Standard Response Format

```
GoFrame Unified Response:
{
  "code": 0,           // 0 = success, non-zero = error
  "message": "",       // Error description
  "data": {...}        // Result payload (null on error)
}

Error Codes (from gcode package):
├─ 0: Success
├─ 400: CodeMissingParameter
├─ 400: CodeInvalidParameter
├─ 401: CodeNotAuthorized
├─ 403: CodeForbidden
├─ 405: CodeNotFound
├─ 500: CodeInternalError
├─ 500: CodeDbOperationError
├─ 500: CodeBusinessValidationFailed
└─ (GoFrame predefined codes)
```

### Common Error Scenarios

```
1. Invalid RSA token (Device)
   ├─ Base64 decode error
   ├─ RSA decrypt failure
   ├─ Timestamp validation failure (>±10s)
   └─ Response: Middleware returns 401

2. Invalid JWT (App/Admin)
   ├─ Missing token
   ├─ Expired token
   ├─ Invalid signature
   └─ Response: Middleware returns 401

3. Resource not found
   ├─ Device not in pool
   ├─ Device not bound to user
   ├─ Dance not found
   └─ Response: Service returns error

4. Database error
   ├─ Connection failed
   ├─ Query failed
   └─ Response: Wrapped with CodeDbOperationError

5. External service error (M5Stack Auth, XiaoZhi)
   ├─ Network timeout
   ├─ Response parsing failure
   ├─ Business error from remote
   └─ Response: Wrapped with CodeInternalError or CodeBusinessValidationFailed
```

## Summary: Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **Three-tier auth** | Device (RSA MAC), App (JWT uid), Admin (JWT username) |
| **WebSocket forwarding** | Real-time app↔device messaging without app knowing device IP |
| **DAO auto-generation** | GoFrame reduces boilerplate, consistency |
| **Soft delete (app_store)** | Preserve audit trail for admin operations |
| **Connection pool (sync.Map)** | Lock-free, concurrent WS handling |
| **5s heartbeat + 15s cleanup** | Keep-alive + stale removal balance |
| **JWT 365-day expiration (app)** | Long-lived tokens for user convenience (app-side refresh responsibility) |
| **JWT 24-hour expiration (admin)** | Shorter window for sensitive admin operations |
| **External M5Stack auth** | Centralized user identity, no duplicate auth |
| **XiaoZhi integration** | Vendor-specific AI agent, abstracted via internal/xiaozhi |
| **No server-side session** | Stateless (JWT) for horizontal scalability |
