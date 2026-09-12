# StackChan Code Reading Guide

最初にこれらのファイルを読むことで、システム全体を段階的に理解できます。

## 最初に読む5ファイル (必須)

### 1. **server/README.MD** [重要度: Critical]

**理由:**
- サーバーの全体像・目的・アーキテクチャが最も詳しく記述
- API仕様・データベーススキーマ・設定方法が一通り揃っている
- このドキュメントから全体像を把握

**注目する関数・クラス:**
- Section 1: Overview (責務)
- Section 2: Architecture and Modules (アクター・ルート)
- Section 12: Database Overview (テーブル一覧)

---

### 2. **server/main.go** [重要度: Critical]

**理由:**
- エントリーポイント
- Go プログラムの開始地点を把握

**コード:**
```go
package main

import (
    "stackChan/internal/cmd"
    _ "stackChan/internal/packed"
    _ "github.com/gogf/gf/contrib/drivers/mysql/v2"
    "github.com/gogf/gf/v2/os/gctx"
)

func main() {
    cmd.Main.Run(gctx.GetInitCtx())
}
```

**重要:**
- `cmd.Main.Run()` が実際の起動処理を行う
- mysql driver とファイル資産の初期化が自動

---

### 3. **server/internal/cmd/cmd.go** [重要度: Critical]

**理由:**
- HTTP サーバー設定
- ルート登録・ミドルウェア設定
- アプリケーション起動シーケンス全体

**コード構造:**
```go
Main = gcmd.Command{
    Func: func(ctx context.Context, parser *gcmd.Parser) (err error) {
        s := g.Server()
        s.SetClientMaxBodySize(100 * 1024 * 1024)
        
        // Middleware
        s.Use(middleware.CORS)
        
        // WebSocket
        s.BindHandler("/stackChan/ws", web_socket.Handler)
        
        // Routes
        s.Group("/file", ...)
        s.Group("/stackChan/v2", ...)  // App API
        s.Group("/stackChan", ...)      // Device API
        s.Group("/admin/stackChan", ...) // Admin API
        
        s.SetPort(12800)
        s.Run()
    }
}
```

**注目点:**
- ルート分割: `/stackChan/v2` (app) vs `/stackChan` (device)
- 各ルートグループのミドルウェア差異
- WebSocket エントリーポイント

---

### 4. **server/internal/middleware/middleware.go** [重要度: Critical]

**理由:**
- 認証・認可の全ロジック
- 3種類の認証方式の実装

**重要関数:**
- `TokenAuthMiddleware()` - RSA MAC トークン (device)
- `V2TokenAuthMiddleware()` - JWT ユーザートークン (app)
- `AdminTokenAuthMiddleware()` - JWT 管理者トークン

**認証フロー:**
```go
TokenAuthMiddleware()
  → web_socket.GetMac() [RSA復号]
  → ctx.SetCtxVar(model.Mac, mac)

V2TokenAuthMiddleware()
  → jwt.Parse() [JWT検証]
  → claims["id"] をuidとして抽出
  → ctx.SetCtxVar(model.Uid, uid)

AdminTokenAuthMiddleware()
  → jwt.Parse() [JWT検証]
  → claims["username"] を抽出
  → ctx.SetCtxVar(model.Username, username)
```

**キー実装:**
- RSA-OAEP-SHA256 復号
- JWT HMAC署名検証
- Timestamp 有効期限チェック (±10秒)

---

### 5. **server/check_list/create_mysql_database.sql** [重要度: Critical]

**理由:**
- データベーススキーマ全体
- テーブル関係・制約
- カラム定義・コメント

**主要テーブル:**
1. `user` - リモート UID キャッシュ
2. `device` - デバイス (MAC 基点)
3. `device_dance` - ダンス JSON データ
4. `device_post` - デバイス投稿
5. `device_post_comment` - 投稿コメント
6. `device_pano` - パノラマ画像
7. `device_friend` - デバイス友情
8. `app_store` - App Store アイテム

**重要な制約:**
- `device.mac` = PK (MAC アドレス)
- `device.uid` = FK → user.uid (nullable)
- Cascade delete: device 削除 → dance/post/pano/comments 削除
- Cascade set null: user 削除 → device.uid = NULL

---

## 次に読むファイル (段階別)

### Phase 1: ユーザー認証フロー理解

```
server/internal/controller/user/user_v2_login.go
  → server/internal/service/user.go::Login()
  → server/internal/model/entity/user.go (Entity)
  → server/internal/dao/user.go (DAO, auto-generated)
```

**読む順序:**
1. `internal/controller/user/user_v2_login.go`
   - HTTP ハンドラーの形式確認
   - リクエスト・レスポンス型

2. `internal/service/user.go`
   - `Login()`, `callRemoteLogin()`, `generateToken()`
   - 外部M5Stackサービスとの連携
   - JWT生成ロジック

3. `internal/model/entity/user.go`
   - Entity 構造体定義

4. `internal/dao/user.go`
   - GoFrame自動生成の ORM レイヤー
   - Save/Delete/Query メソッド

---

### Phase 2: デバイスバインドフロー理解

```
server/internal/controller/device/device_v2_bind.go (app側)
  → server/internal/service/device.go::BindDevice()
  → server/internal/dao/device.go

server/internal/controller/device/device_v1_create.go (device側)
  → server/internal/service/device.go::CreateDevice()
  → RSA認証・MAC抽出
```

**読む順序:**
1. `internal/middleware/middleware.go::V2TokenAuthMiddleware()`
   - JWT認証（app側）

2. `internal/controller/device/device_v2_bind.go`
   - app ユーザーがデバイスをバインド

3. `internal/service/device.go`
   - `BindDevice()`, `CreateDevice()`

4. `internal/dao/device.go`
   - Device テーブルアクセス

---

### Phase 3: WebSocket リアルタイム通信理解

```
server/internal/web_socket/web_socket.go (Main handler)
  → server/internal/web_socket/socket_task.go (Message handling)
  → Message routing by type
  → Connection pool management
```

**読む順序:**
1. `internal/web_socket/web_socket.go`
   - `Handler()` - WebSocket upgrade
   - `GetMac()` - RSA トークン復号
   - Message type constants (0x01-0x1A)
   - `readClientMessage()` loop

2. `internal/web_socket/socket_task.go`
   - Message type別ハンドラー
   - `handleMessage()` dispatch
   - Connection pool: `stackChanClientPool`, `appClientPool`

3. `internal/boot/socket_task.go`
   - Heartbeat scheduler (5秒ping)
   - Stale connection cleanup (15秒)

---

### Phase 4: ダンス制御フロー理解

```
server/internal/controller/dance/dance_v2_add.go (Create)
  → server/internal/service/dance.go::AddDeviceDance()
  → server/internal/dao/device_dance.go

[再生]
WebSocket 0x14 (Dance) message
  → socket_task.go::handleDanceMessage()
  → Device connection に転送
```

**読む順序:**
1. `internal/controller/dance/dance_v2_add.go`
   - ダンス作成 API ハンドラー

2. `internal/service/dance.go`
   - `AddDeviceDance()` - DAO 呼び出し

3. `internal/dao/device_dance.go`
   - Dance テーブルアクセス

4. `internal/web_socket/socket_task.go::handleDanceMessage()`
   - ダンス再生時の WebSocket メッセージ処理

---

### Phase 5: ファイルアップロード・提供

```
server/internal/controller/file/file_v1_upload.go
  → server/internal/service/file.go
  → server/internal/cmd/cmd.go (/file/* static route)
```

**読む順序:**
1. `internal/controller/file/file_v1_upload.go`
   - multipart/form-data ハンドリング
   - ファイル保存

2. `internal/service/file.go`
   - ファイル操作の詳細

3. `internal/cmd/cmd.go` の `/file` グループ
   - 静的ファイル提供

---

### Phase 6: Admin コンソール

```
server/internal/controller/admin/admin_v1_login.go
  → server/internal/service/admin_user.go

server/internal/controller/appstore/appstore_v1_add.go
  → App Store CRUD
```

**読む順序:**
1. `internal/controller/admin/admin_v1_login.go`
   - Admin ログイン

2. `internal/service/admin_user.go`
   - Admin 認証・JWT生成

3. `internal/controller/appstore/appstore_v1_add.go`
   - App Store アイテム管理

---

## エントリーポイント別コード追跡

### HTTP Request エントリー

```
HTTP Request → GoFrame routing
  ↓
Middleware stack (CORS, Auth, etc.)
  ↓
Controller method
  (internal/controller/*/*)
  ↓
Service method (ビジネスロジック)
  (internal/service/*.go)
  ↓
DAO method (DB操作)
  (internal/dao/*.go, auto-generated)
  ↓
MySQL Database

Response (逆順):
  DAO result → Service → Controller → GoFrame → HTTP Response
```

### WebSocket エントリー

```
WS /stackChan/ws
  ↓
web_socket.Handler()
  ├─ Upgrade to WebSocket
  ├─ GetMac() [RSA復号・認証]
  ├─ Connection pool に登録
  └─ readClientMessage() loop
      ↓
      Binary frame parse
      ├─ msgType (1 byte)
      ├─ payload length (4 bytes BE)
      └─ payload (N bytes)
      ↓
      socket_task.handleMessage()
      ├─ msgType別ハンドラー dispatch
      ├─ Connection pool を検索
      └─ Forward or Broadcast

例:
0x03 (ControlAvatar) from App
  → Find device connection in stackChanClientPool
  → Forward to device
  → Device firmware executes servo/eye control
```

### Scheduled Task エントリー

```
boot.InitCron()
  ├─ Task 1: Heartbeat (every 5s)
  │  → Send 0x10 (ping) to all connections
  │  → Location: internal/boot/socket_task.go
  │
  └─ Task 2: Cleanup (every 15s)
     → Remove stale connections (no pong received)
     → Location: internal/boot/socket_task.go
```

---

## 主要ユースケース別コードマップ

### UC1: ユーザーログイン

```
Entry: POST /stackChan/v2/user/login

Flow:
  app/lib/network/http.dart (Dart)
    │ HTTP POST with username/password
    ↓
  server/internal/controller/user/user_v2_login.go
    │ - Extract req body
    │ - No auth (public endpoint)
    ↓
  server/internal/service/user.go::Login()
    │ - Validate inputs
    │ - Call callRemoteLogin()
    │   └─ HTTP POST to m5stack.loginUrl
    │   └─ Parse RemoteLoginResp
    │ - Call saveUserToLocal()
    │   └─ dao.User.Save(entity) → INSERT/UPDATE
    │ - Call generateToken()
    │   └─ JWT HS256 sign
    ↓
  Response: { token: "eyJhbGc..." }

Database:
  user table: INSERT/UPDATE uid, username, displayname, ...

Client:
  Store token in localStorage
  Use in subsequent requests: token: Bearer <jwt>
```

### UC2: デバイスバインド (App側)

```
Entry: POST /stackChan/v2/device/bind

Flow:
  app/lib/network/http.dart
    │ HTTP POST with mac
    │ Header: token: Bearer <user-jwt>
    ↓
  server/internal/middleware/middleware.go::V2TokenAuthMiddleware()
    │ - Parse JWT
    │ - Extract uid from claims["id"]
    │ - ctx.SetCtxVar(model.Uid, uid)
    ↓
  server/internal/controller/device/device_v2_bind.go
    │ - Get uid from ctx
    │ - Get mac from request
    ↓
  server/internal/service/device.go::BindDevice()
    │ - Check if device exists
    │   └─ dao.Device.One() where mac
    │ - If not exists: INSERT new device
    │ - If exists: UPDATE device.uid = uid
    │ - Set bind_time
    ↓
  server/internal/dao/device.go::Save()
    │ SQL: INSERT/UPDATE device WHERE mac
    ↓
  Response: { mac, name, uid, bind_time }

Database:
  device table:
  ├─ mac (PK): AA:BB:CC:DD:EE:FF
  ├─ uid: 12345
  └─ bind_time: "2026-09-12..."
```

### UC3: デバイス登録 (Device側, REST)

```
Entry: POST /stackChan/device

Flow:
  device/firmware/... (C/C++)
    │ HTTP POST
    │ Header: Authorization: <rsa-mac-token>
    │         (RSA encrypted: MAC|nonce|timestamp)
    ↓
  server/internal/middleware/middleware.go::TokenAuthMiddleware()
    │ - Get Authorization header
    │ - Base64 decode
    │ - web_socket.GetMac()
    │   └─ RSA-OAEP-SHA256 decrypt using rsa.server.private
    │   └─ Extract MAC from "MAC|nonce|timestamp"
    │   └─ Validate timestamp ±10 seconds
    │ - ctx.SetCtxVar(model.Mac, mac)
    ↓
  server/internal/controller/device/device_v1_create.go
    │ - Get mac from ctx
    │ - Get name from request
    ↓
  server/internal/service/device.go::CreateDevice()
    │ - dao.Device.Save() INSERT/UPDATE
    ↓
  Response: { mac, name, uid (if bound) }

Database:
  device table: INSERT mac, name (uid = NULL initially)
```

### UC4: ダンス作成 (App側)

```
Entry: POST /stackChan/v2/dance

Flow:
  app/lib/view/... (Dart)
    │ User creates dance in UI
    │ → HTTP POST with:
    │   - mac
    │   - danceName
    │   - danceData (JSON array of frames)
    │   - musicUrl
    │ Header: token: Bearer <user-jwt>
    ↓
  server/internal/middleware/middleware.go::V2TokenAuthMiddleware()
    │ - Parse JWT, extract uid
    ↓
  server/internal/controller/dance/dance_v2_add.go
    │ - Validate device ownership
    │   └─ device.uid == uid
    │ - Get dance details from request
    ↓
  server/internal/service/dance.go::AddDeviceDance()
    │ - Prepare entity.DeviceDance
    │ - dao.DeviceDance.Save() INSERT
    ↓
  server/internal/dao/device_dance.go
    │ SQL: INSERT INTO device_dance (mac, dance_name, dance_data, music_url)
    │      VALUES (?, ?, ?, ?)
    ↓
  Response: { id, mac, danceName, ... }

Database:
  device_dance table:
  ├─ id: 1 (auto-increment)
  ├─ mac: AA:BB:CC:DD:EE:FF
  ├─ dance_name: "wave-demo"
  ├─ dance_data: [{"leftEye": {...}, "rightEye": {...}, ...}]
  └─ music_url: "http://server/file/music/..."

[Later] ダンス再生:
  app/lib/network/web_socket_util.dart
    │ WebSocket 0x14 frame: danceName + duration
    ↓
  server/internal/web_socket/web_socket.go::readClientMessage()
    │ - Receive binary frame
    │ - msgType == 0x14
    ↓
  server/internal/web_socket/socket_task.go::handleMessage()
    │ - handleDanceMessage()
    │ - Find device connection in pool
    │ - Forward to device
    ↓
  device/firmware/... (C/C++)
    │ Receive dance message
    │ Execute servo motions + LED animations
    │ Play music
```

### UC5: リアルタイム音声・画像 (WebSocket)

```
Entry: WS /stackChan/ws

Flow:
  device/firmware/...
    │ Open WebSocket
    │ Authorization: <rsa-mac-token>
    │ GET /stackChan/ws?deviceType=StackChan
    ↓
  server/internal/web_socket/web_socket.go::Handler()
    │ - websocket.Upgrader.Upgrade()
    │ - web_socket.GetMac() [RSA auth]
    │ - stackChanClientPool[mac] = conn
    ↓
  device/firmware/...
    │ Send binary frames:
    │ - 0x01 (Opus): Audio chunks
    │ - 0x02 (Jpeg): Camera images
    │ - 0x03 (ControlAvatar): Response to eye control
    ↓
  server/internal/web_socket/web_socket.go::readClientMessage()
    │ - Read binary frames
    │ - For 0x01, 0x02: Broadcast to all app clients for this device
    │ - For 0x03: Keep locally (avatar state)
    ↓
  app/lib/network/web_socket_util.dart
    │ Receive 0x01 (Opus)
    │ - Play audio
    │
    │ Receive 0x02 (Jpeg)
    │ - Display camera image

  app → device (reverse):
    │ Send 0x03 (ControlAvatar):
    │ - Eye position/rotation/size
    │ - Mouth position
    ↓
  server/internal/web_socket/socket_task.go::handleMessage()
    │ - msgType == 0x03
    │ - Find device connection
    │ - Forward to device
    ↓
  device/firmware/...
    │ Receive eye control
    │ Update servo/LED/expression
```

### UC6: ファイルアップロード

```
Entry: POST /stackChan/uploadFile (Device side)
       OR
       POST /admin/stackChan/uploadFile (Admin side)

Flow:
  device/firmware/... OR admin/web/...
    │ HTTP POST (multipart/form-data)
    │ - file: [binary]
    │ - name: "photo.jpg"
    │ - directory: "posts" (or "apps" for admin)
    ↓
  server/internal/middleware/middleware.go::TokenAuthMiddleware()
    │ (Device auth with RSA)
    ↓
  server/internal/controller/file/file_v1_upload.go
    │ - Parse multipart request
    │ - Validate directory/filename
    ↓
  server/internal/service/file.go::SaveFile()
    │ - Create file/[directory] if not exists
    │ - Write file to disk
    ↓
  Disk: file/[directory]/[name]

Response:
  { path: "file/posts/photo.jpg" }

[Later] ファイル提供:
  Entry: GET /file/posts/photo.jpg
    ↓
  server/internal/cmd/cmd.go (/file group)
    │ - Check if file exists
    │ - Serve file with correct MIME type
    ↓
  File content
```

---

## 重要なパターン・メソッド

### DAO Pattern (GoFrame自動生成)

```go
// internal/dao/user.go (auto-generated)
var User = &userDao{}

func (d *userDao) Ctx(ctx context.Context) *UserDao {
    return d.ctx(ctx)
}

func (d *UserDao) Save(entity interface{}) (result sql.Result, err error) {
    // INSERT or UPDATE
}

func (d *UserDao) One(ctx context.Context, where ...interface{}) (*entity.User, error) {
    // SELECT * LIMIT 1
}

func (d *UserDao) All(ctx context.Context, where ...interface{}) ([]*entity.User, error) {
    // SELECT *
}

func (d *UserDao) Delete(ctx context.Context, where ...interface{}) (sql.Result, error) {
    // DELETE
}

// Usage:
user, err := dao.User.Ctx(ctx).One(ctx, "uid = ?", uid)
```

### Service Pattern (ビジネスロジック)

```go
// internal/service/device.go
func BindDevice(ctx context.Context, uid int64, mac string) (*entity.Device, error) {
    // 1. Validate
    if mac == "" {
        return nil, errors.New("mac required")
    }
    
    // 2. Check ownership / authorization
    // (app middleware already validated uid)
    
    // 3. Access database
    device, err := dao.Device.Ctx(ctx).One(ctx, "mac = ?", mac)
    if err != nil && !db.IsNotFound(err) {
        return nil, err
    }
    
    // 4. Create or update
    if device == nil {
        device = &entity.Device{Mac: mac}
    }
    device.Uid = uid
    device.BindTime = time.Now().Format("2006-01-02 15:04:05")
    
    _, err = dao.Device.Ctx(ctx).Save(device)
    if err != nil {
        return nil, err
    }
    
    return device, nil
}
```

### Controller Pattern (HTTP Handler)

```go
// internal/controller/device/device_v2_bind.go
type V2 struct{}

func NewV2() *V2 { return &V2{} }

func (c *V2) Bind(ctx context.Context, req *device.V2BindReq) (res *device.V2BindRes, err error) {
    // 1. Get context variables set by middleware
    uid := ctx.Value(model.Uid).(int64)
    
    // 2. Call service
    result, err := service.BindDevice(ctx, uid, req.Mac)
    if err != nil {
        return nil, err
    }
    
    // 3. Return response
    return &device.V2BindRes{
        Mac:      result.Mac,
        Name:     result.Name,
        Uid:      result.Uid,
        BindTime: result.BindTime,
    }, nil
}
```

### RSA Authentication Pattern

```go
// internal/web_socket/web_socket.go
func GetMac(r *ghttp.Request) (string, error) {
    token := r.Header.Get(model.Authorization)
    if token == "" {
        return "", errors.New("authorization header missing")
    }
    
    // 1. Base64 decode
    decodedToken, err := base64.StdEncoding.DecodeString(token)
    if err != nil {
        return "", err
    }
    
    // 2. RSA decrypt
    decrypted, err := utility.RSADecrypt(decodedToken)
    if err != nil {
        return "", err
    }
    
    // 3. Parse plaintext: "MAC|nonce|timestamp"
    tokenStr := string(decrypted)
    parts := strings.Split(tokenStr, "|")
    if len(parts) < 3 {
        return "", errors.New("invalid token format")
    }
    
    mac := parts[0]
    timestamp, err := strconv.ParseInt(parts[2], 10, 64)
    if err != nil {
        return "", err
    }
    
    // 4. Validate timestamp (±10 seconds)
    now := time.Now().Unix()
    if math.Abs(float64(now-timestamp)) > 10 {
        return "", errors.New("timestamp too old")
    }
    
    return mac, nil
}
```

### JWT Generation Pattern

```go
// internal/service/user.go
func generateToken(ctx context.Context, uid int64) (string, error) {
    now := time.Now()
    
    claims := jwt.MapClaims{
        "jti": guid.S(),                    // unique token ID
        "id":  uid,                         // user UID
        "iss": g.Cfg().MustGet(ctx, "m5stack.issuer").String(),
        "aud": g.Cfg().MustGet(ctx, "m5stack.audience").String(),
        "iat": now.Unix(),
        "exp": now.Add(TokenExpire).Unix(), // 365 days
    }
    
    tokenObj := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    jwtSecret := GetJwtSecret()
    token, err := tokenObj.SignedString([]byte(jwtSecret))
    if err != nil {
        return "", err
    }
    
    return token, nil
}
```

---

## デバッグ・トレース方法

### Log確認

```bash
# サーバーログ
tail -f server/logs/*.log

# JSON format example:
# [2026-09-12 12:00:00.123] [DBG] [user.go:55] "Login attempt" uid=12345
```

### Database確認

```bash
mysql -u user -p stackChan

# ユーザー確認
SELECT * FROM user WHERE uid = 12345;

# デバイス確認
SELECT * FROM device WHERE mac = 'AA:BB:CC:DD:EE:FF';

# ダンス確認
SELECT id, mac, dance_name FROM device_dance WHERE mac = 'AA:BB:CC:DD:EE:FF';
```

### WebSocket Debug

```bash
# wscat で WebSocket テスト
wscat -c ws://localhost:12800/stackChan/ws?deviceType=StackChan \
  -H "Authorization: <rsa-mac-token>"

# または
curl --include \
  --no-buffer \
  --header "Connection: Upgrade" \
  --header "Sec-WebSocket-Key: ..." \
  --header "Authorization: <rsa-mac-token>" \
  ws://localhost:12800/stackChan/ws?deviceType=StackChan
```

### API テスト

```bash
# Login
curl -X POST http://localhost:12800/stackChan/v2/user/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"test","password":"pass"}'

# Device bind
curl -X POST http://localhost:12800/stackChan/v2/device/bind \
  -H 'Content-Type: application/json' \
  -H 'token: Bearer <jwt>' \
  -d '{"mac":"AA:BB:CC:DD:EE:FF"}'

# List dances
curl "http://localhost:12800/stackChan/v2/dance?mac=AA:BB:CC:DD:EE:FF" \
  -H 'token: Bearer <jwt>'
```

---

## コード品質・テスト

### Unit Test例

```go
// internal/service/device_test.go
func TestBindDevice(t *testing.T) {
    // Setup
    ctx := context.Background()
    uid := int64(123)
    mac := "AA:BB:CC:DD:EE:FF"
    
    // Execute
    device, err := BindDevice(ctx, uid, mac)
    
    // Assert
    if err != nil {
        t.Fatal(err)
    }
    if device.Mac != mac || device.Uid != uid {
        t.Errorf("unexpected device: %+v", device)
    }
}
```

### テスト実行

```bash
cd server
go test ./...  # All tests
go test ./internal/service -v  # Specific package
```

---

## まとめ

| 段階 | 学習内容 | ファイル |
|------|--------|--------|
| 1 | 全体像 | README.MD, main.go, cmd.go |
| 2 | 認証・ルーティング | middleware.go |
| 3 | ユーザー流れ | controller/user, service/user |
| 4 | デバイス管理 | controller/device, service/device |
| 5 | リアルタイム通信 | web_socket/web_socket.go, socket_task.go |
| 6 | ビジネスロジック詳細 | 各 service/\*.go |
| 7 | データベース | create_mysql_database.sql, dao/\*.go |

このガイドの流れで読むことで、コード内で迷いにくくなります。
