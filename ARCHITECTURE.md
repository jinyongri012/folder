# Azure IoT ダミーアプリケーション アーキテクチャ設計書

## 📋 概要

本アプリケーションは、Azure IoT Hubにテレメトリデータを送信するIoTデバイスシミュレーターです。UWBセンサーなどのIoTデバイスの動作をエミュレートし、開発・テスト・デモ用途に使用できます。

## 🏗️ アーキテクチャ概要

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────┐
│   app.js        │ ───── │ DeviceSimulator  │ ───── │ Azure IoT Hub   │
│ (エントリポイント) │       │   (コーディネータ)  │       │   (クラウド)     │
└─────────────────┘       └──────────────────┘       └─────────────────┘
                                    │
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │ EnvironmentConfig│    │SensorDataGenerator│    │   IoTClient     │
   │  (設定管理)       │    │  (データ生成)      │    │  (通信管理)      │
   └─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📁 ディレクトリ構造

```
azure-iot-dummy-app/
├── app.js                     # アプリケーションエントリポイント
├── package.json               # Node.js依存関係とスクリプト
├── .env                       # 環境変数設定 (Git対象外)
├── env.example               # 環境変数テンプレート
├── setup-azure-iot.sh       # Azure リソース作成スクリプト
├── get-device-connection-string.sh # デバイス接続文字列取得スクリプト
├── README.md                 # プロジェクト説明書
├── SETUP_GUIDE.md           # セットアップガイド
└── src/
    ├── config/
    │   └── environment.js    # 環境設定管理クラス
    ├── iot/
    │   └── iotClient.js      # Azure IoT Hub クライアント
    ├── sensors/
    │   └── sensorDataGenerator.js # センサーデータ生成器
    └── simulator/
        └── deviceSimulator.js # デバイスシミュレーターメインクラス
```

## 🔧 コンポーネント詳細

### 1. app.js (エントリポイント)
**責務**: アプリケーションの起動と終了処理
- DeviceSimulatorのインスタンス化
- シグナルハンドリング (SIGINT, SIGTERM)
- 未捕獲例外処理
- 優雅な終了処理

### 2. DeviceSimulator (コーディネーター)
**場所**: `src/simulator/deviceSimulator.js`
**責務**: 全コンポーネントの調整と管理

#### 主要機能:
- **ライフサイクル管理**: 開始、停止、再起動
- **データ送信制御**: 定期的なテレメトリ送信
- **メッセージ処理**: Cloud-to-Device メッセージ対応
- **ログ管理**: 送信データの記録と表示

#### 主要メソッド:
```javascript
start()           // シミュレーター開始
stop()            // シミュレーター停止  
sendTelemetry()   // テレメトリデータ送信
handleCloudMessage() // クラウドメッセージ処理
restart()         // 再起動
```

### 3. IoTClient (通信管理)
**場所**: `src/iot/iotClient.js`
**責務**: Azure IoT Hubとの通信管理

#### 主要機能:
- **接続管理**: MQTT接続の確立・切断
- **メッセージ送信**: テレメトリデータの送信
- **イベント処理**: エラー、切断、受信メッセージ
- **メッセージプロパティ**: デバイスタイプ、タイムスタンプなど

#### 通信プロトコル:
- **プロトコル**: MQTT over TLS
- **認証**: 共有アクセスキー (SAS)
- **メッセージ形式**: JSON

### 4. SensorDataGenerator (データ生成)
**場所**: `src/sensors/sensorDataGenerator.js`
**責務**: リアルなセンサーデータの生成

#### 生成データタイプ:
- **環境センサー**:
  - 温度 (日変化シミュレーション)
  - 湿度 (日変化シミュレーション) 
  - 気圧
  - 空気質量指数 (AQI)
  - 光照度 (日変化シミュレーション)

- **運動センサー**:
  - 加速度計 (X, Y, Z軸)
  - ジャイロスコープ (X, Y, Z軸)

- **位置データ** (オプション):
  - GPS座標 (緯度、経度)
  - 高度
  - 精度

- **バッテリー** (オプション):
  - バッテリーレベル
  - 充電状態
  - 電圧

### 5. EnvironmentConfig (設定管理)
**場所**: `src/config/environment.js`
**責務**: 環境変数の管理と検証

#### 管理する設定:
- IoT Hub接続文字列 (必須)
- デバイス設定 (ID、タイプ、送信間隔)
- 機能フラグ (位置データ、バッテリーシミュレーション)

## 📊 データフロー

### 1. 送信データフロー
```
SensorDataGenerator → DeviceSimulator → IoTClient → Azure IoT Hub
     ↓                     ↓              ↓            ↓
1. データ生成          2. 送信制御      3. MQTT送信   4. クラウド受信
   - センサー値           - 定期実行       - JSON変換    - ストレージ
   - タイムスタンプ       - エラー処理     - プロパティ   - 分析
   - メッセージID         - ログ出力       - 認証        - 監視
```

### 2. 受信データフロー (Cloud-to-Device)
```
Azure IoT Hub → IoTClient → DeviceSimulator → アクション実行
     ↓             ↓            ↓                ↓
1. メッセージ送信  2. 受信処理   3. コマンド解析    4. 動作変更
   - 設定変更        - JSON解析   - タイプ判定       - 間隔変更
   - コマンド        - エラー処理 - パラメータ抽出   - 再起動
```

## 🔌 外部依存関係

### NPM パッケージ
- **azure-iot-device**: Azure IoT Device SDK
- **azure-iot-device-mqtt**: MQTT トランスポート
- **dotenv**: 環境変数管理
- **uuid**: ユニークID生成

### Azure サービス
- **Azure IoT Hub**: デバイス管理とメッセージング
- **Azure CLI**: リソース管理とデプロイ

## ⚙️ 設定オプション

### 環境変数 (.env)
```bash
# 必須設定
IOT_HUB_CONNECTION_STRING=HostName=xxx.azure-devices.net;DeviceId=xxx;SharedAccessKey=xxx

# デバイス設定
DEVICE_ID=uwb-sensor-001
DEVICE_TYPE=uwb-positioning-sensor  
SEND_INTERVAL=5000                   # 送信間隔 (ミリ秒)

# 機能フラグ
ENABLE_LOCATION_DATA=true            # GPS位置データ
ENABLE_BATTERY_SIMULATION=true       # バッテリーシミュレーション
```

## 🛡️ エラーハンドリング

### 1. 接続エラー
- **自動再接続**: IoTClientで接続状態監視
- **グレースフル処理**: 接続失敗時のデータスキップ
- **ログ出力**: 詳細なエラー情報記録

### 2. データ送信エラー
- **リトライロジック**: 送信失敗時の処理継続
- **ステータス監視**: 送信成功/失敗の追跡
- **アラート**: 連続失敗時の通知

### 3. 設定エラー
- **起動時検証**: 必須環境変数のチェック
- **デフォルト値**: オプション設定の自動補完
- **エラーメッセージ**: 分かりやすいエラー表示

## 🔄 ライフサイクル管理

### 1. 起動プロセス
```
app.js起動 → 設定読み込み → IoT Hub接続 → データ送信開始 → 定期実行
```

### 2. 停止プロセス  
```
シグナル受信 → 送信停止 → IoT Hub切断 → リソース解放 → プロセス終了
```

### 3. エラー復旧
```
エラー検出 → ログ記録 → 接続再試行 → サービス復旧
```

## 🚀 スケーラビリティ考慮事項

### 1. 複数デバイス対応
- 環境変数でのデバイスID制御
- Docker化による複数インスタンス実行
- 負荷分散機能

### 2. データ量制御
- 送信間隔の動的調整
- データサイズの最適化
- バッチ送信機能

### 3. 監視機能
- ヘルスチェック API
- メトリクス収集
- アラート機能

## 🔧 カスタマイズポイント

### 1. センサーデータ拡張
`SensorDataGenerator.js`を編集して新しいセンサータイプを追加:
```javascript
generateCustomSensorData() {
    return {
        customValue: Math.random() * 100,
        customUnit: 'unit'
    };
}
```

### 2. 通信プロトコル変更
`IoTClient.js`でHTTPSやAMQP対応可能:
```javascript
const { Https } = require('azure-iot-device-https');
this.client = Client.fromConnectionString(connectionString, Https);
```

### 3. データ処理ロジック追加
`DeviceSimulator.js`でカスタム処理を追加:
```javascript
processCustomData(data) {
    // カスタムデータ処理ロジック
    return processedData;
}
```

## 📈 パフォーマンス考慮事項

### 1. メモリ使用量
- データバッファリングの制限
- オブジェクトプーリング
- ガベージコレクション最適化

### 2. CPU使用量  
- 非同期処理の活用
- タイマー処理の効率化
- 計算処理の最適化

### 3. ネットワーク効率
- 接続プール管理
- 送信データの圧縮
- バッチ処理による効率化

## 📚 関連ドキュメント

- [Azure IoT Hub ドキュメント](https://docs.microsoft.com/azure/iot-hub/)
- [Azure IoT Device SDK](https://docs.microsoft.com/azure/iot-hub/iot-hub-devguide-sdks)
- [MQTT プロトコル仕様](https://mqtt.org/mqtt-specification/)

---

*このドキュメントは Azure IoT ダミーアプリケーション v1.0.0 用に作成されました。*