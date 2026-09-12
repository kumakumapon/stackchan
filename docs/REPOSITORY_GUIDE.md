# StackChan Repository Guide

## 1. 一言でいうと

StackChanは、M5Stack CoreS3（ESP32）をベースにした「かわいいAI対応デスクトップロボット」です。
このリポジトリは、ロボット本体のファームウェア、モバイルアプリ（iOS/Android）、リモコン、バックエンドサーバーの4つの主要コンポーネントを一つのモノレポで管理しています。
ユーザーはモバイルアプリからロボットを制御・操作でき、サーバーがそれらの実行時データを管理します。

## 2. このシステムが解決する問題

- **AI対応ロボットコンパニオン**: M5StackハードウェアをベースにしたAIロボットの実装・運用
- **デバイス・アプリケーション管理**: 複数のロボット、複数ユーザー、複数アプリの一元管理
- **リアルタイム双方向通信**: ロボットとアプリ間のリアルタイムメッセージング（WebSocket）
- **モーション・ダンス制御**: ロボットのサーボ・RGB LED・表情を細かく制御
- **オフライン・オンライン連携**: ESPNowおよびWi-Fi両対応

## 3. 主な利用者

- **エンドユーザー**: モバイルアプリ経由でロボットを操作・カスタマイズ
- **ロボット本体（ファームウェア）**: Wi-Fi接続してサーバーと通信
- **管理者**: 管理画面（Flutter Web）からアプリストア・ファイルを管理

## 4. 主な機能

| 機能 | 主要実装 | 対象 |
|------|--------|------|
| **ユーザー認証** | 外部M5Stackサービス + 本地JWT | app, admin |
| **デバイス管理** | device テーブル | app (bind/unbind) |
| **ダンス制御** | device_dance テーブル + WebSocket | app/device |
| **モーション制御** | WebSocket 0x04 フレーム | app -> device |
| **パノラマ撮影** | device_pano テーブル | device |
| **コミュニティ機能** | device_post, device_post_comment | device |
| **ファイルアップロード** | /stackChan/uploadFile, /file/* | app/device/admin |
| **リアルタイム音声・映像** | WebSocket Opus(0x01), Jpeg(0x02) | app <-> device |
| **AI会話** | XiaoZhi API統合 | app (via device) |
| **アプリストア** | app_store テーブル | app/device |
| **RGBLEDアニメーション** | color.leftRgbColor/rightRgbColor | ダンスデータ |

## 5. システム全体像

```
┌─────────────────────────────────────────────────────────────┐
│                    StackChan Ecosystem                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐                  ┌──────────────┐         │
│  │   モバイル    │                  │   ロボット本体  │         │
│  │   アプリ     │<--─WebSocket─────>│  (CoreS3)   │         │
│  │ (Flutter)   │                  │ (ESP32-S3)  │         │
│  └──────────────┘                  └──────────────┘         │
│         │                                  │                 │
│         │ HTTP REST                        │ HTTP REST       │
│         │                                  │ RSA Auth        │
│         v                                  v                 │
│  ┌──────────────────────────────────────────────────┐       │
│  │        StackChan Server (Go + GoFrame)          │       │
│  │                 Port: 12800                      │       │
│  │  - Device/User Management                       │       │
│  │  - WebSocket Forwarding                         │       │
│  │  - File Upload/Serve                            │       │
│  │  - Admin Console (Flutter Web)                  │       │
│  └──────────────────────────────────────────────────┘       │
│         │                                                     │
│         └─── MySQL Database (8 tables)                       │
│         │                                                     │
│         └─── XiaoZhi API (AI Agent)                          │
│                                                               │
│  ┌──────────────────────┐      ┌──────────────┐             │
│  │ ESPNow リモコン       │      │  管理者       │             │
│  │ (別のESP32)          │      │  (Web)      │             │
│  └──────────────────────┘      └──────────────┘             │
│                                        │                     │
│                                   HTTP REST                  │
│                            (Admin APIs)                      │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## 6. 使用技術

| 分類 | 技術 | このプロジェクトでの役割 |
|------|------|----------------------|
| **Backend** | Go 1.26.3 | サーバーロジック実装 |
| **Framework** | GoFrame v2 | HTTPルーティング, DAO, ORMキット |
| **Database** | MySQL 8.0+ | ユーザー・デバイス・ダンス・投稿・コメント管理 |
| **Mobile App** | Flutter 3.0+ | iOS/Androidアプリケーション |
| **Mobile Language** | Dart 3.0+ | モバイルアプリ開発言語 |
| **Firmware** | ESP-IDF 5.5.4 | ロボット本体ファームウェア(C/C++) |
| **Microcontroller** | ESP32-S3 | ロボット本体SoC (Dual-core 240MHz, 8MB PSRAM) |
| **Communication** | WebSocket (Gorilla) | リアルタイム双方向通信 |
| **Auth** | JWT + RSA-OAEP | トークン認証・MACトークン暗号化 |
| **AI Service** | XiaoZhi API | 音声会話・エージェント管理 |

## 7. ディレクトリ構成

```
stackchan/
├── server/                          # Go バックエンド
│   ├── api/                         # GoFrame API定義（自動生成）
│   │   ├── admin/
│   │   ├── user/                    # ユーザーログイン・登録
│   │   ├── device/
│   │   ├── dance/
│   │   ├── appstore/
│   │   └── ...
│   ├── internal/
│   │   ├── cmd/cmd.go               # HTTP起動・ルート登録
│   │   ├── controller/              # HTTPハンドラー層
│   │   ├── service/                 # ビジネスロジック層
│   │   ├── dao/                     # DBアクセス層（GoFrame自動生成）
│   │   ├── model/                   # リクエスト・レスポンス・Entity
│   │   ├── middleware/              # JWT・RSA認証
│   │   ├── web_socket/              # WebSocket接続管理・フォワード
│   │   ├── xiaozhi/                 # XiaoZhi AI API
│   │   ├── boot/                    # 定期タスク（heartbeat, cleanup）
│   │   └── packed/                  # embedded assets
│   ├── check_list/
│   │   └── create_mysql_database.sql # DBスキーマ
│   ├── file/                        # アップロード済みファイル
│   ├── hack/                        # GoFrame CLI設定
│   ├── manifest/
│   │   ├── config/config.yaml       # ★重要: 設定ファイル
│   │   ├── docker/Dockerfile        # Docker イメージ定義
│   │   └── deploy/kustomize/        # K8s デプロイメント（テンプレート）
│   ├── utility/                     # ユーティリティ（RSA復号など）
│   ├── web/management/              # Flutter Web 管理画面（ビルド済み）
│   ├── main.go                      # エントリーポイント
│   ├── go.mod / go.sum              # Go依存関係
│   ├── Makefile                     # ビルドコマンド
│   └── README.MD                    # サーバー詳細ドキュメント
│
├── app/                             # Flutter モバイルアプリ
│   ├── lib/
│   │   ├── main.dart                # アプリエントリーポイント
│   │   ├── model/                   # データモデル
│   │   ├── network/                 # HTTP・WebSocket
│   │   ├── util/                    # ユーティリティ（RSA, BLE など）
│   │   ├── view/                    # UI層（画面）
│   │   └── app_state.dart           # グローバルState
│   ├── android/                     # Android設定
│   ├── ios/                         # iOS設定
│   ├── pubspec.yaml                 # Dart依存関係
│   └── README.md                    # アプリ詳細ドキュメント
│
├── firmware/                        # ESP32-S3 ファームウェア
│   ├── main/                        # メイン処理
│   ├── tests/                       # ホスト側テスト
│   ├── fetch_repos.py               # 依存リポジトリ取得
│   └── README.md                    # ファームウェア詳細ドキュメント
│
├── remote/                          # ESPNow リモコン
│   ├── code/                        # リモコンファームウェア
│   └── README.md                    # リモコン詳細ドキュメント
│
├── docs/                            # ★このドキュメント
│   ├── REPOSITORY_GUIDE.md          # このファイル
│   ├── ARCHITECTURE.md              # システムアーキテクチャ
│   ├── CODE_READING_GUIDE.md        # コード読み順序
│   └── DEVELOPMENT_GUIDE.md         # 開発セットアップ
│
└── README.md                        # プロジェクト概要
```

## 8. 起動から動作開始まで

### 8.1 サーバー側

```
1. config.yaml 設定
   ├── server.address: ":12800"
   ├── database.default.link: "mysql://..."
   ├── jwt.secret: "generated-secret"
   ├── m5stack.loginUrl, registrationUrl: 外部認証エンドポイント
   ├── rsa.server.private/public: RSA鍵
   └── xiaozhi.secret_key: XiaoZhi API鍵

2. MySQL DB初期化
   └── mysql < check_list/create_mysql_database.sql

3. サーバー起動
   └── go run . または go build -o stackChan && ./stackChan

4. 内部処理
   ├── internal/cmd/cmd.go::Main.Run()
   │   ├── g.Server() (port 12800)
   │   ├── ルート登録:
   │   │   ├── /file/* (静的ファイル)
   │   │   ├── /stackChan/v2/* (app, JWT認証)
   │   │   ├── /stackChan/* (device, RSA認証)
   │   │   ├── /admin/stackChan/* (admin, JWT認証)
   │   │   └── /stackChan/ws (WebSocket)
   │   └── boot.InitCron() (heartbeat/cleanup)
   └── s.Run() で起動待機

5. クライアント接続
   └── app/device が REST/WebSocket で通信開始
```

### 8.2 アプリ側

```
1. 起動
   └── lib/main.dart

2. 設定読み込み
   └── lib/network/urls.dart (server URL)
   └── lib/util/value_constant.dart (RSA public key)

3. Bluetooth デバイス検索

4. サーバーログイン
   └── POST /stackChan/v2/user/login (JWT取得)

5. デバイス操作（WebSocket接続）
   └── WS /stackChan/ws?deviceType=App&deviceId=...
```

## 9. 主要ユースケース

### 9.1 ユーザーログイン・登録

**エントリーポイント:**
- `POST /stackChan/v2/user/registration` → `internal/controller/user/user_v2_register.go`
- `POST /stackChan/v2/user/login` → `internal/controller/user/user_v2_login.go`

**フロー:**
```
app (Flutter)
  │ POST /stackChan/v2/user/login
  ├─ username, password
  │
  v server (internal/controller/user)
  │ → internal/service/user.go::Login()
  │   ├─ callRemoteLogin() (外部M5Stackサービスに問い合わせ)
  │   ├─ saveUserToLocal() (userテーブルに保存)
  │   └─ generateToken() (JWT発行)
  │
  v database (user テーブル)
  │
  v response (JWT token)
  │
  v app (token を保存)
```

### 9.2 デバイスバインド

**フロー:**
```
app (Flutter)
  │ POST /stackChan/v2/device/bind
  ├─ mac (デバイスMAC)
  ├─ token (Bearer <jwt>)
  │
  v server (internal/middleware)
  │ → V2TokenAuthMiddleware
  │   └─ JWT検証・uid抽出
  │
  v server (internal/controller/device)
  │ → device_v2_bind.go
  │
  v internal/service/device.go::Bind()
  │ ├─ デバイス存在確認
  │ ├─ デバイス作成または既存レコード取得
  │ └─ device テーブルを uid で更新
  │
  v database (device テーブル)
  │ ├─ mac (PK)
  │ ├─ name
  │ ├─ uid (ユーザーID) ← updated
  │ └─ bind_time
  │
  v response (デバイス情報)
```

### 9.3 ダンス作成・再生

**フロー:**
```
app (Flutter)
  │ POST /stackChan/v2/dance
  ├─ mac
  ├─ danceName
  ├─ danceData (JSON: 各フレームの目・口・サーボ位置・RGB色)
  ├─ musicUrl
  │
  v server (internal/controller/dance)
  │ → dance_v2_add.go
  │
  v internal/service/dance.go::AddDeviceDance()
  │ └─ device_dance テーブルに JSON保存
  │
  v database (device_dance テーブル)
  │
  v response (ダンスID)

[再生時]
app (or device)
  │ WebSocket 0x14 (Dance) フレーム送信
  │ ├─ ペイロード: ダンスID + タイミング情報
  │
  v server (internal/web_socket)
  │ → handleDanceMessage()
  │
  v device (ESP32)
  │ → サーボ動作・LED点灯・音声再生
```

### 9.4 デバイス間リアルタイム通信（WebSocket）

**フロー:**
```
デバイスA (ロボット)
  │ WS接続
  │ /stackChan/ws?deviceType=StackChan
  │ Authorization: (RSA-encrypted MAC token)
  │
  v server (internal/web_socket)
  │ → Handler() (gorilla/websocket upgrade)
  │ → GetMac() (RSA復号, MAC抽出)
  │ → connection pool に登録
  │ → readClientMessage loop
  │
  v (connection active)

アプリ (モバイル)
  │ WS接続
  │ /stackChan/ws?deviceType=App&deviceId=...
  │ Authorization: (RSA-encrypted MAC token)
  │
  v server (connection pool に登録)

[通信例: アプリ→デバイス]
アプリ
  │ 0x03 (ControlAvatar) フレーム送信
  │ ├─ 左目 X/Y/回転
  │ ├─ 右目 X/Y/回転
  │ └─ 口 X/Y/回転
  │
  v server (web_socket.go::handleMessage())
  │ → 対応デバイスの connection を検索
  │ → フレーム転送
  │
  v デバイス (ロボット)
  │ → サーボ・LEDアニメーション実行
```

## 10. データの流れ

### 10.1 ユーザー認証フロー

```
Request (app)
  │ username, password (plaintext)
  │
  v server::callRemoteLogin()
  │ → 外部M5StackサービスにPOST
  │
  v RemoteLoginResp (JSON)
  │ ├─ status.code: "ok"
  │ ├─ response.uid: ユーザーID
  │ ├─ response.username: ユーザー名
  │ └─ ...
  │
  v internal/model/entity.User
  │ ├─ uid (PK)
  │ ├─ username
  │ ├─ display_name
  │ └─ ...
  │
  v database (user テーブル)
  │
  v JWT generation
  │ ├─ iss (issuer): m5stack.issuer
  │ ├─ aud (audience): m5stack.audience
  │ ├─ id: uid
  │ ├─ exp: 365 days
  │ └─ jti: unique token ID
  │
  v Response (app)
  │ { token: "..." }
```

### 10.2 ダンスデータ構造

```
danceData (JSON配列)
  │ 各フレーム:
  │ ├─ leftEye
  │ │  ├─ x, y (座標)
  │ │  ├─ rotation (回転角)
  │ │  ├─ weight (不透明度)
  │ │  └─ size (大きさ)
  │ ├─ rightEye (同様)
  │ ├─ mouth (同様)
  │ ├─ pitchServo
  │ │  ├─ angle (サーボ角度)
  │ │  └─ speed (動作速度)
  │ ├─ yawServo (同様)
  │ ├─ leftRgbColor (16進数色)
  │ ├─ rightRgbColor
  │ └─ durationMs (フレーム期間)
  │
  v database (device_dance.dance_data)
  │ → JSON型で保存
  │
  v WebSocket (0x14 Dance)
  │ → ペイロード: danceName + durationMs
```

## 11. 外部サービス

### 11.1 M5Stack ユーザー認証サービス

- **用途**: ユーザー登録・ログイン
- **呼び出し元**: `internal/service/user.go`
- **エンドポイント**: `m5stack.loginUrl`, `m5stack.registrationUrl`
- **認証**: `m5stack.registrationToken` (Bearer token)
- **送受信**: username/password ⇄ uid/username/displayname/...

### 11.2 XiaoZhi AI Service

- **用途**: 音声対話・エージェント管理・ライセンス
- **呼び出し元**: `internal/xiaozhi/`, `internal/controller/xiaozhi/`
- **エンドポイント**: `https://xiaozhi.me/`
- **認証**: `xiaozhi.secret_key` (secret key)
- **操作**: トークン取得・リフレッシュ・ライセンストークン生成・エージェント設定

### 11.3 外部ファイルストレージ

- **用途**: ダンス背景音楽URL、写真保存（将来）
- **形式**: 任意のHTTP(S) URL
- **管理**: `/file/*` で本地提供

## 12. 設定・環境変数

### 12.1 config.yaml (manifest/config/config.yaml)

```yaml
server:
  address: ":12800"              # リッスンポート
  openapiPath: "/api.json"       # OpenAPI JSON出力

database:
  default:
    link: "mysql://user:pass@host:3306/stackChan"  # DBコネクション

jwt:
  secret: "generated-long-secret"  # JWT署名秘密鍵

admin:
  users:
    - username: "admin"
      password: "password"       # 管理者認証情報

m5stack:
  loginUrl: "https://..."        # 外部ログインエンドポイント
  registrationUrl: "https://..."
  registrationToken: "Bearer ..."
  issuer: "stackchan-server"     # JWT issuer
  audience: "stackchan-app"      # JWT audience

xiaozhi:
  secret_key: "..."              # XiaoZhi API秘密鍵
  generate_license_token: "..."  # ライセンストークン

rsa:
  server:
    private: "-----BEGIN RSA PRIVATE KEY-----\n..."
    public: "-----BEGIN PUBLIC KEY-----\n..."
```

### 12.2 アプリ側設定

- **lib/network/urls.dart**: サーバーURL
- **lib/util/value_constant.dart**: RSA公開鍵・RSA秘密鍵

## 13. テスト

### 13.1 Server側

- **ユニットテスト**: `internal/service/device_test.go`
- **実行**: `go test ./...`

### 13.2 App側

- **ウィジェットテスト**: `test/widget_test.dart`
- **実行**: `flutter test`
- **カバレッジ**: `flutter test --coverage`
- **リント**: `flutter analyze`

### 13.3 Firmware側

- **ホスト側テスト** (モーション座標ヘルパー)
  ```bash
  cmake -S firmware/tests -B build-host-tests
  cmake --build build-host-tests
  ctest --test-dir build-host-tests --output-on-failure
  ```

## 14. 変更時の注意点

### 14.1 既知の矛盾・不完全な実装

1. **device テーブル: longitude/latitude**
   - 外部公開API（PUT /stackChan/v2/device/update）では対応していますが
   - スキーマでは列が存在しません
   - 実装予定の場合は事前に migration を実行

2. **device.bind_time の型**
   - varchar(32) として保存（推測: ISO8601文字列）
   - datetime 型への変更を検討

3. **hard-coded music URL**
   - `internal/controller/dance/dance_v2_get_list.go`
   - 本番環境では自身のファイルサーバーURL に置き換える

4. **Flutter Web 管理画面**
   - ソースコード非公開（ビルド済み資産のみ）
   - 更新には別のリポジトリが必要

### 14.2 セキュリティ注意事項

1. **環境変数・秘密鍵**
   - config.yaml の secret 関連は環境変数から注入
   - git にコミットしない

2. **RSA鍵ペア**
   - server.private, server.public, client.private (app), client.public は秘密文書
   - 本番環境ごとに異なる鍵を使用

3. **JWT secret**
   - openssl rand -base64 32 で生成
   - 最低 16 bytes 以上

### 14.3 ビジネスロジック上の注意

1. **デバイスバインド・アンバインド**
   - アンバインド時に XiaoZhi agent reset を試行
   - 失敗してもアンバインドは成功する設計

2. **WebSocket タイムアウト**
   - 15秒ごとに stale connection をクリーンアップ
   - heartbeat: 5秒ごと ping 送信

3. **トークン有効期限**
   - app JWT: 365日
   - admin JWT: 24時間
   - XiaoZhi token: 24時間キャッシュ+リフレッシュ

4. **デバイス MAC アドレス**
   - device テーブルの PK
   - 複数ユーザーで共有可能（uid で所有権分離）
   - ESPNow リモコンも同じ MAC スキーム

## 15. 用語集

| 用語 | 意味 |
|------|------|
| **StackChan** | M5Stack CoreS3 ベースのAI対応ロボット |
| **CoreS3** | M5Stack の最新フラグシップ IoT開発キット（ESP32-S3搭載） |
| **MAC** | デバイスの物理アドレス (Media Access Control) 例: AA:BB:CC:DD:EE:FF |
| **WebSocket** | 双方向通信プロトコル、リアルタイムメッセージ用 |
| **DAO** | Data Access Object, GoFrame の ORM 自動生成層 |
| **GoFrame** | Go 高性能 Web フレームワーク |
| **RSA** | 非対称暗号、デバイス認証 MAC トークン暗号化に使用 |
| **JWT** | JSON Web Token, ユーザー・管理者認証トークン |
| **XiaoZhi** | 中国の AI エージェント・音声認識 API |
| **ESPNow** | Espressif 独自の低遅延無線通信プロトコル |
| **CORS** | Cross-Origin Resource Sharing, ブラウザ間通信許可 |
| **OTA** | Over-The-Air, ファームウェア無線更新 |

## 16. 不明点・確認が必要な点

1. **Flutter Web 管理画面のソースコード**
   - web/management 配下はビルド済み資産のみ
   - 管理画面の変更が必要な場合は別リポジトリが必要（未公開）

2. **XiaoZhi API の詳細**
   - internal/xiaozhi 配下の実装から推測
   - 公開ドキュメントが必要な場合は確認

3. **ロボット本体のモーション座標系**
   - danceData の x/y/rotation が何を基準にしているかは firmware/main を参照必要
   - 3D座標系の正確な定義が確認が必要

4. **デバイス online/offline の管理**
   - WebSocket 接続状態から推測
   - 明確な timeout 定義が docs に不足

5. **パフォーマンス・スケーラビリティ**
   - WebSocket connection pool の max size
   - MySQL query 最適化状況
   - 同時接続デバイス数の上限
