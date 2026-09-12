# StackChan Development Guide

開発環境のセットアップと実装パターンのガイドです。

## Requirements

### サーバー開発

- **Go**: 1.26.3 以上 ([golang.org](https://golang.org))
- **MySQL**: 8.0 以上
- **GoFrame CLI**: `make build` / `make dao` 用（オプション）
- **OS**: macOS, Linux, Windows

確認:
```bash
go version
mysql --version
```

### モバイルアプリ開発

- **Flutter**: 3.0 以上 ([flutter.dev](https://flutter.dev))
- **Dart**: 3.0 以上
- **iOS**: Xcode 12.0+, iOS 14.0+ SDK
- **Android**: Android Studio, Android API 21+

確認:
```bash
flutter doctor
dart --version
```

### ファームウェア開発

- **ESP-IDF**: v5.5.4 ([docs.espressif.com](https://docs.espressif.com/projects/esp-idf/en/v5.5.4/esp32s3/))
- **Python**: 3.6 以上（idf.py 用）

確認:
```bash
idf.py --version
python3 --version
```

---

## Setup

### 1. リポジトリクローン

```bash
git clone https://github.com/kumakumapon/stackchan.git
cd stackchan
```

### 2. サーバーセットアップ

#### 2.1 Go 依存関係のインストール

```bash
cd server
go mod download  # ダウンロード
go mod tidy      # 最適化
```

#### 2.2 MySQL データベース初期化

```bash
# 1. Database を作成
mysql -u root -p <<EOF
CREATE DATABASE IF NOT EXISTS stackChan
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
EOF

# 2. Schema をインポート
mysql -u root -p stackChan < check_list/create_mysql_database.sql

# 3. 確認
mysql -u root -p stackChan -e "SHOW TABLES;"
```

#### 2.3 設定ファイル作成

```bash
# Template からコピー
cp manifest/config/config.yaml manifest/config/config.yaml.local

# 編集
vi manifest/config/config.yaml
```

**最小限の設定:**

```yaml
server:
  address: ":12800"
  openapiPath: "/api.json"

logger:
  path: "./logs"
  file: "{Y-m-d}.log"
  level: "all"
  stdout: true

database:
  default:
    link: "mysql://root:password@127.0.0.1:3306/stackChan?charset=utf8mb4&collation=utf8mb4_0900_ai_ci"

jwt:
  secret: "your-long-secret-key-min-16-chars-generated-by-openssl-rand"

admin:
  users:
    - username: "admin"
      password: "your-admin-password"

m5stack:
  loginUrl: "https://m5stack-auth-service/login"
  registrationUrl: "https://m5stack-auth-service/register"
  registrationToken: "Bearer your-registration-token"
  issuer: "stackchan-server"
  audience: "stackchan-app"

rsa:
  server:
    private: |
      -----BEGIN RSA PRIVATE KEY-----
      your-private-key-here
      -----END RSA PRIVATE KEY-----
    public: |
      -----BEGIN PUBLIC KEY-----
      your-public-key-here
      -----END PUBLIC KEY-----

xiaozhi:
  secret_key: "your-xiaozhi-secret"
  generate_license_token: "your-license-token"
```

JWT Secret 生成:
```bash
openssl rand -base64 32
```

#### 2.4 設定検証

```bash
# サーバーが設定を読み込めるか確認
cd server
go run . -help
```

### 3. モバイルアプリセットアップ

#### 3.1 Flutter 依存関係

```bash
cd app
flutter pub get
```

#### 3.2 設定ファイル編集

```bash
# server URL 設定
vi lib/network/urls.dart
```

**編集内容:**
```dart
class Urls {
  static const String url = "http://your-server-ip:12800/stackChan/";
  // ...
}
```

#### 3.3 RSA 鍵設定

```bash
vi lib/util/value_constant.dart
```

**編集内容:**
```dart
class ValueConstant {
  static const String serverPublicKey = """
-----BEGIN PUBLIC KEY-----
your-server-public-key-here
-----END PUBLIC KEY-----
""";
  
  static const String clientPrivateKey = """
-----BEGIN RSA PRIVATE KEY-----
your-client-private-key-here
-----END RSA PRIVATE KEY-----
""";
}
```

#### 3.4 Lint / Format 確認

```bash
flutter analyze
dart format lib/
```

### 4. ファームウェアセットアップ

#### 4.1 依存リポジトリの取得

```bash
cd firmware
python3 ./fetch_repos.py
```

#### 4.2 ESP-IDF 環境設定

```bash
# ESP-IDF インストール（初回のみ）
# $IDF_INSTALL_PATH に インストール後
export IDF_PATH=/path/to/esp-idf
source $IDF_PATH/export.sh
```

#### 4.3 設定確認

```bash
idf.py --version
```

---

## Environment Variables

### サーバー側

```bash
# 実行時環境変数の注入例
export DB_LINK="mysql://root:pass@localhost:3306/stackChan"
export JWT_SECRET="your-secret"
export ADMIN_USERNAME="admin"
export ADMIN_PASSWORD="pass"

go run .
```

または `manifest/config/config.yaml` で定義。

### アプリ側

```bash
# Dart environment variables
flutter run --dart-define=SERVER_URL=http://192.168.1.100:12800
```

### ファームウェア側

```bash
# ESP-IDF config
idf.py menuconfig
# Component config → StackChan Configuration で設定
```

---

## Database Setup

### Initial Schema

```bash
mysql -u root -p stackChan < check_list/create_mysql_database.sql
```

### Schema 変更時

1. SQL ファイルを直接編集
2. `go.mod` の `gf` バージョン確認
3. DAO 再生成:
   ```bash
   make dao
   ```
4. エンティティが自動生成される:
   - `internal/dao/*.go`
   - `internal/model/entity/*.go`

### Migration (将来)

Currently, migrations are not tracked in a formal way.  
When schema changes are needed:

1. Update `check_list/create_mysql_database.sql`
2. Manually run ALTER TABLE statements
3. Regenerate DAO: `make dao`

## Migration

Database schema を変更する場合:

```bash
# 1. SQL 実行（例: 列追加）
mysql -u root -p stackChan -e \
  "ALTER TABLE device ADD COLUMN longitude DECIMAL(10,8);"

# 2. DAO 再生成
cd server
make dao

# 3. Model/Controller を更新
# internal/model/entity/device.go が自動更新される
```

---

## Run

### サーバー実行

#### 開発モード

```bash
cd server

# 直接実行（ホットリロード非対応）
go run .

# ビルド して実行
go build -o stackChan .
./stackChan
```

起動ログ:
```
2026-09-12 12:00:00.000 [INF] [] http server started listening on ":12800"
2026-09-12 12:00:00.100 [INF] [] Boot: Heartbeat started
2026-09-12 12:00:00.110 [INF] [] Boot: Connection cleanup started
```

#### ホットリロード (Unofficial)

```bash
go install github.com/cosmtrek/air@latest
air  # config: .air.toml
```

#### Docker 実行

```bash
# Build
cd server
go build -o stackChan .
docker build -f manifest/docker/Dockerfile -t stackchan-server:dev .

# Run
docker run -d \
  -p 12800:12800 \
  -v $(pwd)/manifest/config/config.yaml:/app/config.yaml \
  -e DB_LINK="mysql://root:pass@host.docker.internal:3306/stackChan" \
  stackchan-server:dev
```

### モバイルアプリ実行

#### iOS

```bash
cd app

# Install Pods
cd ios
pod install
cd ..

# Run on simulator
flutter run -d ios

# Run on device
flutter run -d <device-id>

# Build for release
flutter build ios --release
```

#### Android

```bash
cd app

# Run on emulator
flutter run -d emulator

# Run on device
flutter run -d <device-id>

# Build APK
flutter build apk --release

# Build App Bundle
flutter build appbundle --release
```

### ファームウェア

```bash
cd firmware

# Build
idf.py build

# Flash
idf.py flash

# Monitor (serial output)
idf.py monitor
```

---

## Test

### Server Tests

#### Unit Tests

```bash
cd server

# All tests
go test ./...

# Specific package
go test ./internal/service -v

# With coverage
go test ./... -cover
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
```

#### Integration Tests (Manual)

```bash
# Start server
cd server
go run .

# In another terminal
cd ../server

# Test login
curl -X POST http://localhost:12800/stackChan/v2/user/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"test","password":"pass"}'

# Response should be:
# { "code": 0, "message": "", "data": { "token": "..." } }
```

### App Tests

```bash
cd app

# Widget tests
flutter test

# Specific test
flutter test test/widget_test.dart

# With coverage
flutter test --coverage

# Lint
flutter analyze

# Format
dart format lib/
```

### Firmware Tests (Host-side)

```bash
cd firmware

# Build and run host tests
cmake -S tests -B build-host-tests
cmake --build build-host-tests
ctest --test-dir build-host-tests --output-on-failure
```

---

## Lint / Format

### Server (Go)

```bash
cd server

# Format (in-place)
gofmt -w .

# Or with more features
go fmt ./...

# Lint
golangci-lint run ./...

# Or basic
go vet ./...
```

### App (Dart)

```bash
cd app

# Format (in-place)
dart format lib/

# Lint
flutter analyze
```

### Firmware (C/C++)

```bash
cd firmware

# Using clang-format (if available)
find . -name "*.c" -o -name "*.h" | xargs clang-format -i

# Or manual formatting guidelines in ESP-IDF docs
```

---

## Build

### Server Binary

```bash
cd server

# Simple build
go build -o stackChan .

# With version info
go build -ldflags="-X main.Version=v1.0.0" -o stackChan .

# Build for different OS
GOOS=linux GOARCH=amd64 go build -o stackChan-linux .
GOOS=darwin GOARCH=amd64 go build -o stackChan-mac .
```

### Server (GoFrame CLI)

```bash
cd server

# Requires: go install github.com/gogf/gf/cmd/gf@latest
make build

# Which runs:
# gf build
```

### Docker Image

```bash
cd server

# Build binary
go build -o stackChan .

# Build image
docker build -f manifest/docker/Dockerfile -t stackchan-server:v1.0 .

# Tag for registry
docker tag stackchan-server:v1.0 your-registry/stackchan-server:v1.0

# Push
docker push your-registry/stackchan-server:v1.0
```

### Mobile App

```bash
cd app

# iOS
flutter build ios --release  # Output: build/ios/iphoneos/Runner.app

# Android APK
flutter build apk --release  # Output: build/app/outputs/apk/release/app-release.apk

# Android AAB (for Play Store)
flutter build appbundle --release  # Output: build/app/outputs/bundle/release/app-release.aab
```

### Firmware

```bash
cd firmware

# Build
idf.py build  # Output: build/stackChan.bin

# Build for OTA
idf.py build  # Same command, with OTA settings in menuconfig
```

---

## Debug

### Server Debug

#### Print Debug Logs

Edit `manifest/config/config.yaml`:
```yaml
logger:
  level: "all"  # "debug" or "all" for verbose
```

#### Breakpoint Debug (VSCode)

`.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "StackChan Server",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/server",
      "env": {},
      "args": [],
      "showLog": true
    }
  ]
}
```

Run: `Ctrl+F5` in VSCode

#### Database Query Debug

```bash
# Monitor queries in real-time
mysql -u root -p stackChan
> SET GLOBAL log_queries_not_using_indexes = 'ON';
> SHOW ENGINE INNODB STATUS;
```

### App Debug

#### Flutter DevTools

```bash
flutter pub global activate devtools
flutter pub global run devtools

# Or automatically
flutter run --devtools
```

#### Breakpoint Debug (VSCode)

`.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Flutter",
      "type": "dart",
      "request": "launch",
      "program": "lib/main.dart"
    }
  ]
}
```

#### Print Debug

```dart
// In lib/**/*.dart
print('Debug: $variable');
debugPrint('Debug: $variable');  // Flutter preferred
```

### Firmware Debug

#### Serial Monitor

```bash
cd firmware
idf.py monitor

# Verbose output
idf.py monitor -v

# Filter by tag
idf.py monitor --tag TAG_NAME
```

#### UART Logging

Firmware code:
```c
ESP_LOGI("STACKCHAN", "Debug message: %d", value);
ESP_LOGE("STACKCHAN", "Error: %s", error_msg);
```

---

## よくある変更

### 1. APIを追加する

#### a. API定義を追加

`server/api/device/v2/device_v2_custom.go`:
```go
package v2

type CustomReq struct {
    g.Meta `path:"device/custom" method:"POST" summary:"Custom operation"`
    Mac    string `json:"mac" dc:"Device MAC"`
    Data   string `json:"data" dc:"Custom data"`
}

type CustomRes struct {
    g.Meta  struct{}
    Result  string `json:"result" dc:"Operation result"`
}
```

#### b. Controllerを追加

`server/internal/controller/device/device_v2_custom.go`:
```go
package device

func (c *V2) Custom(ctx context.Context, req *v2.CustomReq) (res *v2.CustomRes, err error) {
    uid := ctx.Value(model.Uid).(int64)
    
    // Validate
    if req.Mac == "" {
        return nil, gerror.New("mac required")
    }
    
    // Business logic
    result, err := service.CustomOperation(ctx, uid, req.Mac, req.Data)
    if err != nil {
        return nil, err
    }
    
    return &v2.CustomRes{Result: result}, nil
}
```

#### c. Serviceを追加

`server/internal/service/device.go` に追記:
```go
func CustomOperation(ctx context.Context, uid int64, mac string, data string) (string, error) {
    // Implement business logic
    device, err := dao.Device.Ctx(ctx).One(ctx, "mac = ?", mac)
    if err != nil {
        return "", err
    }
    
    // Verify ownership
    if device.Uid != uid && device.Uid != nil {
        return "", gerror.New("device not owned by user")
    }
    
    // Do something with data
    result := "success: " + data
    
    return result, nil
}
```

#### d. ルート登録確認

`server/internal/cmd/cmd.go` では `group.Bind(device.NewV2(), ...)` で自動登録される。

#### e. テスト

```bash
curl -X POST http://localhost:12800/stackChan/v2/device/custom \
  -H 'token: Bearer <jwt>' \
  -H 'Content-Type: application/json' \
  -d '{"mac":"AA:BB:CC:DD:EE:FF","data":"test"}'
```

### 2. DB項目を追加する

#### a. Schemaを更新

```sql
ALTER TABLE device ADD COLUMN custom_field VARCHAR(255) AFTER bind_time;
```

#### b. DAO再生成

```bash
cd server
make dao
```

#### c. Model/Controller更新

`internal/model/entity/device.go` が自動更新される。  
Controller でフィールドを使用可能:
```go
device.CustomField = req.CustomField
_, err := dao.Device.Ctx(ctx).Save(device)
```

#### d. API定義を更新

`server/api/device/v2/*.go` の Req/Res struct に追加。

### 3. ビジネスロジックを変更する

#### Example: Device bind時の追加検証

`server/internal/service/device.go` の `BindDevice()` を編集:
```go
func BindDevice(ctx context.Context, uid int64, mac string) (*entity.Device, error) {
    // 既存の検証
    if mac == "" {
        return nil, gerror.New("mac required")
    }
    
    // 追加: MAC形式チェック
    if !isMacValid(mac) {
        return nil, gerror.New("invalid MAC format")
    }
    
    // 追加: ユーザーが既に5台以上のデバイスをバインドしていないか確認
    count, err := dao.Device.Ctx(ctx).Count("uid = ?", uid)
    if err != nil {
        return nil, err
    }
    if count >= 5 {
        return nil, gerror.New("maximum 5 devices per user")
    }
    
    // 既存ロジック続行
    // ...
}

func isMacValid(mac string) bool {
    re := regexp.MustCompile(`^([0-9A-F]{2}[:]){5}([0-9A-F]{2})$`)
    return re.MatchString(mac)
}
```

### 4. 外部サービスを追加する

#### Example: 新しいAI APIを統合

`server/internal/xiaozhi/new_service.go`:
```go
package xiaozhi

import (
    "stackChan/internal/model"
    "github.com/gogf/gf/v2/frame/g"
)

type NewServiceClient struct {
    ApiKey string
    BaseUrl string
}

func NewClient() *NewServiceClient {
    return &NewServiceClient{
        ApiKey: g.Cfg().MustGet(g.Ctx(), "newservice.apikey").String(),
        BaseUrl: g.Cfg().MustGet(g.Ctx(), "newservice.baseurl").String(),
    }
}

func (c *NewServiceClient) GetData(ctx context.Context, param string) (string, error) {
    resp := g.Client().GetVar(ctx, 
        c.BaseUrl + "/api/data",
        g.Map{"param": param})
    
    if resp == nil {
        return "", gerror.New("service unavailable")
    }
    
    var result model.NewServiceResp
    err := resp.Scan(&result)
    if err != nil {
        return "", err
    }
    
    return result.Data, nil
}
```

設定を追加:
```yaml
# manifest/config/config.yaml
newservice:
  apikey: "your-api-key"
  baseurl: "https://api.newservice.com"
```

使用:
```go
// internal/controller でも service でも使用可能
client := xiaozhi.NewClient()
data, err := client.GetData(ctx, "parameter")
```

### 5. テストを追加する

```go
// internal/service/device_test.go に追記
func TestCustomOperation(t *testing.T) {
    ctx := context.Background()
    uid := int64(123)
    mac := "AA:BB:CC:DD:EE:FF"
    data := "test-data"
    
    // Setup: device をセットアップ
    device := &entity.Device{Mac: mac, Uid: uid}
    _, err := dao.Device.Ctx(ctx).Save(device)
    if err != nil {
        t.Fatal(err)
    }
    
    // Execute
    result, err := CustomOperation(ctx, uid, mac, data)
    
    // Assert
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result == "" {
        t.Error("result should not be empty")
    }
    
    // Cleanup
    dao.Device.Ctx(ctx).Delete("mac = ?", mac)
}
```

実行:
```bash
cd server
go test ./internal/service -v -run TestCustomOperation
```

---

## 開発時の注意事項

### 1. Authentication Context

ミドルウェアが context に値を設定しているので、controller でアクセス:
```go
// App route: uid が設定されている
uid := ctx.Value(model.Uid).(int64)

// Device route: mac が設定されている
mac := ctx.Value(model.Mac).(string)

// Admin route: username が設定されている
username := ctx.Value(model.Username).(string)
```

### 2. Database Transaction

複数の操作をトランザクション処理する場合:
```go
tx := dao.Device.Ctx(ctx).TX
err := tx.Begin()
if err != nil {
    return err
}

_, err = tx.Device.Ctx(ctx).Save(device1)
if err != nil {
    tx.Rollback()
    return err
}

_, err = tx.Device.Ctx(ctx).Save(device2)
if err != nil {
    tx.Rollback()
    return err
}

tx.Commit()
```

### 3. WebSocket Connection State

WebSocket を使用する場合、接続プールの状態に注意:
```go
// device connection を検索
connInterface, exists := web_socket.stackChanClientPool.Load(mac)
if !exists {
    return gerror.New("device not connected")
}
conn := connInterface.(*websocket.Conn)
```

### 4. Error Handling

GoFrame 標準エラーを使用:
```go
import "github.com/gogf/gf/v2/errors/gerror"
import "github.com/gogf/gf/v2/errors/gcode"

// エラー生成
err := gerror.NewCode(gcode.CodeNotAuthorized, "invalid token")

// エラーラップ（スタックトレース保持）
err := gerror.WrapCode(gcode.CodeDbOperationError, dbErr, "failed to save device")

// チェック
if err != nil && errors.Is(err, context.Canceled) {
    // handle context cancellation
}
```

### 5. Logging

```go
import "github.com/gogf/gf/v2/frame/g"

// 情報
g.Log().Infof(ctx, "User logged in: uid=%d", uid)

// デバッグ
g.Log().Debugf(ctx, "Device state: %+v", device)

// エラー
g.Log().Errorf(ctx, "Failed to bind device: %v", err)
```

### 6. Configuration Access

```go
import "github.com/gogf/gf/v2/frame/g"

// String
url := g.Cfg().MustGet(ctx, "m5stack.loginUrl").String()

// Int
port := g.Cfg().MustGet(ctx, "server.port").Int()

// Bool
enabled := g.Cfg().MustGet(ctx, "feature.enabled").Bool()

// Struct
type Config struct {
    LoginUrl string
    Issuer   string
}
var cfg Config
g.Cfg().MustGet(ctx, "m5stack").Scan(&cfg)
```

### 7. Concurrency

Connection pool は `sync.Map` で thread-safe:
```go
// Safe for concurrent access
web_socket.stackChanClientPool.Store(mac, conn)
web_socket.stackChanClientPool.Load(mac)
web_socket.stackChanClientPool.Delete(mac)
web_socket.stackChanClientPool.Range(func(key, value interface{}) bool {
    // Process each connection
    return true
})
```

### 8. File Upload Security

```go
// ファイルサイズチェック
if req.File.Size > 100*1024*1024 { // 100MB
    return gerror.New("file too large")
}

// ファイルタイプチェック
allowedTypes := map[string]bool{
    "image/jpeg": true,
    "image/png": true,
}
if !allowedTypes[req.File.Header.Get("Content-Type")] {
    return gerror.New("unsupported file type")
}

// パストラバーサル防止
filename := filepath.Base(req.Name)
if filename != req.Name {
    return gerror.New("invalid filename")
}
```

### 9. Rate Limiting (将来対応が必要な場合)

```go
// 推奨: github.com/didip/tollbooth
// または redis ベースレート制限
```

### 10. Documentation

コメントはコードの「なぜ」を説明:
```go
// GOOD: 理由を説明
// Device MAC validation:必须使用标准格式 AA:BB:CC:DD:EE:FF
// because firmware expects exact format, no lowercase support
if !isMacValid(mac) {
    return err
}

// BAD: 何をするか は読めば分かる
// Check if MAC is valid
if !isMacValid(mac) {
    return err
}
```

---

## 本番環境へのデプロイ

### 事前チェックリスト

- [ ] 秘密鍵を環境変数から読み込み（config.yaml にハードコードしない）
- [ ] JWT secret は最低16文字、ランダム生成
- [ ] Admin 認証情報を変更
- [ ] M5Stack 外部サービス URL を設定
- [ ] XiaoZhi API key を設定
- [ ] SSL/TLS 証明書を設定
- [ ] Database バックアップ戦略を立案
- [ ] Log rotation を設定
- [ ] Monitoring/Alert を設定

### Docker Deploy

```bash
# Build
go build -o stackChan .
docker build -f manifest/docker/Dockerfile -t stackchan-server:latest .

# Push to registry
docker tag stackchan-server:latest your-registry/stackchan-server:latest
docker push your-registry/stackchan-server:latest

# Run with env vars
docker run -d \
  --name stackchan-server \
  -p 12800:12800 \
  -e DB_LINK="mysql://..." \
  -e JWT_SECRET="..." \
  -v /path/to/logs:/app/logs \
  -v /path/to/files:/app/file \
  --restart=always \
  your-registry/stackchan-server:latest
```

### Kubernetes Deploy

```bash
# Apply manifests
kubectl apply -f manifest/deploy/kustomize/base

# Check status
kubectl get pods
kubectl logs -f stackchan-server-xyz

# Scale
kubectl scale deployment stackchan-server --replicas=3
```

### Health Check

```bash
# After deployment
curl http://localhost:12800/api.json  # Should return OpenAPI spec

curl -X POST http://localhost:12800/admin/stackChan/login \
  -H 'Content-Type: application/json' \
  -d '{"user_name":"admin","pass_word":"password"}'
  # Should return token
```
