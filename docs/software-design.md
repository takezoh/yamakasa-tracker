# ソフトウェア設計

## 1. 目的

山笠に搭載した4個のBLEタグを、周囲の主催者スマートフォン群が検出し、端末間で位置取得と送信を分散する。各端末の電池消費を抑えながら、観客向けWebへ山笠の現在位置、開催状態、進行方向を公開する。

本設計では暫定的に、直近の有効な参加端末数が10台以上なら`ACTIVE`、10台未満なら`INACTIVE`と判定する。

## 2. システム構成

```text
山笠BLEタグ群
    ↓ BLE広告
主催者モバイルアプリ群
    ├── BLE検出
    ├── 動的参加・離脱
    ├── 分散送信担当選出
    └── 必要時のみ位置取得
            ↓ HTTPS
バックエンド
    ├── 端末presence管理
    ├── 開催状態判定
    ├── 位置レポート検証・統合
    ├── Google Roads APIによる道路スナップ
    ├── 進行方向推定
    └── 公開状態生成
            ↓ SSE / polling
観客向けWeb
    ├── Google Maps
    ├── 山笠の推定位置
    ├── 進行方向
    └── 開催状態・更新時刻
```

## 3. コンポーネント

### 3.1 モバイルアプリ

```text
RelayApp
├── BleScanner
├── TagPresenceTracker
├── RelayMembership
├── CoordinationScheduler
├── LocationProvider
├── ReportUploader
├── LocalEnergyPolicy
└── Diagnostics
```

### 3.2 バックエンド

```text
Backend
├── AuthenticationService
├── DevicePresenceService
├── EventStateService
├── RelayCoordinationService
├── LocationIngestService
├── PositionFusionService
├── RoadSnapService
├── HeadingEstimator
├── PublicProjectionService
└── AdminService
```

### 3.3 観客向けWeb

- Google Maps JavaScript APIで地図表示
- 最新の道路補正済み位置を表示
- 進行方向を矢印または向き付きマーカーで表示
- `ACTIVE` / `INACTIVE`を表示
- 最終更新時刻、推定精度、方向信頼度を表示

## 4. モバイルアプリの状態管理

### 4.1 タグ検出状態

4タグは同一の`yamakasa_id`を持ち、異なる`node_id`を持つ。アプリは4タグを1つの山笠として統合する。

```text
ABSENT
  ↓ 10秒以内にいずれかの正規タグを2回以上検出
PRESENT
  ↓ 直近10秒以内にタグ検出あり
ELIGIBLE
  ↓ 90秒間タグを検出しない
ABSENT
```

推奨初期値：

- 参加判定：10秒以内に2回以上検出
- 有効判定：最終検出から30秒以内
- 猶予：30〜90秒
- 離脱判定：90秒以上検出なし

RSSIは品質評価に用いるが、単独で参加・離脱を決めない。

### 4.2 匿名端末ID

端末は恒久的な個人識別子を送信しない。

- `relay_session_id`: イベントまたは数時間単位でローテーション
- 氏名、電話番号、広告IDは送信しない
- タグ非検出中の位置は取得・送信しない

## 5. 分散送信アルゴリズム

### 5.1 目的

全端末が同時にGPSを取得・送信することを避ける。数十台の端末が存在しても、通常は1更新あたり1台、障害時のみ予備端末が送信する。

### 5.2 時間スロット

開催中の位置更新を20秒単位のスロットへ分割する。

```text
slot_id = floor(server_time / 20 seconds)
```

初期値：

- スロット長：20秒
- 位置更新目標：20〜30秒
- 送信候補バックオフ：0〜15秒
- 同一端末クールダウン：5分

### 5.3 決定論的バックオフ

各端末は次の入力から、自分の待機時間を計算する。

```text
base_score = HMAC(
  session_secret,
  yamakasa_id | slot_id | relay_session_id
)

delay = base_score mod 15 seconds
```

同じ端末が連続して担当しないよう、次のペナルティを加える。

```text
effective_score =
  base_score
  + recent_report_penalty
  + low_battery_penalty
  + low_power_mode_penalty
  + stale_location_penalty
  + weak_ble_penalty
```

優先される端末：

- 最近送信していない
- タグを複数回・複数個検出している
- OSキャッシュに新しい位置を持つ
- バッテリー残量に余裕がある
- 低電力モードでない
- 通信可能

### 5.4 送信前の競合抑制

待機時間が到来した端末は次を確認する。

1. タグを直近10秒以内に検出している
2. 直近10秒で2回以上検出している
3. サーバー上の最新位置が更新期限を過ぎている
4. 自端末がクールダウン中でない
5. 位置権限・通信が利用可能

条件を満たす場合のみ位置を取得する。

```text
GET /v1/yamakasa/{id}/relay-state
```

応答例：

```json
{
  "generation": 1284,
  "lastAcceptedAt": "2026-07-21T05:10:20Z",
  "nextDueAt": "2026-07-21T05:10:40Z"
}
```

送信時はgenerationを指定する。

```text
POST /v1/yamakasa/{id}/reports
If-Match: generation=1284
```

別端末が先に更新していれば、サーバーは`409 Conflict`または`412 Precondition Failed`を返し、後続端末は終了する。

### 5.5 フォールバック

第一候補が位置取得または通信に失敗した場合、待機時間の長い第二・第三候補が引き継ぐ。

- 0〜5秒：第一候補群
- 5〜10秒：第二候補群
- 10〜15秒：第三候補群

厳密な単一リーダー選出より、少数の重複を許す方がOSのバックグラウンド制約に強い。重複レポートはバックエンドで統合する。

## 6. 位置取得と省電力

### 6.1 段階的な位置取得

選出された端末だけが位置を評価する。

1. OSが保持する最終位置を確認
2. 30秒以内かつ精度50m以内なら採用
3. 不十分なら低〜中精度の単発位置取得
4. 5〜8秒以内に取得できれば送信
5. 精度100mを超える場合は原則不採用

常時高精度GPS、常時位置監視、全端末による定期測位は採用しない。

### 6.2 想定負荷

20秒更新、30台参加なら、理論上は1端末あたり約10分に1回の担当となる。

実際には参加・離脱やOS制約があるため、目標は1端末あたり平均5〜15分に1回以下とする。

### 6.3 BLEスキャン

- 対象Service UUIDまたはManufacturer Dataでフィルタリング
- 無期限の高頻度アプリ内ループを避ける
- OSのバックグラウンドBLE機構を利用
- 同一タグ広告の処理をデバウンス
- タグ未検出時に位置処理を起動しない

## 7. 開催・非開催判定

### 7.1 有効端末

バックエンドは端末のpresenceを短期リースとして管理する。

端末はタグを検出中、30〜60秒ごとに軽量presenceを送る。

```json
{
  "relaySessionId": "rotating-id",
  "yamakasaId": 1,
  "observedAt": "2026-07-21T05:10:15Z",
  "bleLastSeenAgeMs": 1200,
  "detectedNodeCount": 2,
  "rssiMedian": -67
}
```

presenceには緯度経度を含めない。

有効端末の条件：

- presenceが直近90秒以内
- タグ最終検出がpresence送信時点から10秒以内
- 正規タグ認証に成功
- 同一`relay_session_id`の重複を除外

### 7.2 暫定判定

```text
active_device_count >= 10 → ACTIVE
active_device_count < 10  → INACTIVE
```

チャタリングを防ぐためヒステリシスを入れる。

推奨初期値：

- `INACTIVE → ACTIVE`: 10台以上が60秒継続
- `ACTIVE → INACTIVE`: 10台未満が180秒継続

これにより、一時的な通信断や端末の出入りで開催状態が頻繁に変わることを防ぐ。

### 7.3 開催状態と位置公開

開催状態と位置そのものは分離して管理する。

- `ACTIVE`: 現在地、進行方向、更新時刻を表示
- `INACTIVE`: 最後に確認された位置を表示可能
- 夜間停車中もタグは広告を継続する

## 8. 位置レポート

```json
{
  "reportId": "uuid",
  "relaySessionId": "rotating-id",
  "yamakasaId": 1,
  "slotId": 912345,
  "observedAt": "2026-07-21T05:10:42Z",
  "latitude": 33.5901,
  "longitude": 130.4012,
  "horizontalAccuracyM": 18,
  "locationAgeMs": 6200,
  "bleLastSeenAgeMs": 900,
  "detectedNodes": [
    {"nodeId": "front", "rssiMedian": -69, "sampleCount": 8},
    {"nodeId": "left", "rssiMedian": -64, "sampleCount": 10}
  ],
  "appState": "background",
  "generation": 1284
}
```

## 9. 位置検証・統合

### 9.1 入力検証

- タグ認証
- アプリ認証・端末attestation
- タイムスタンプ範囲
- 水平精度
- BLE検出鮮度
- スロット重複
- 物理的に不可能な移動速度

最高速度は人が全力で走る程度とし、初期のハード上限を12m/sとする。位置精度を考慮し、単純な速度閾値だけで拒否しない。

```text
allowed_distance =
  max_speed * elapsed_time
  + previous_accuracy
  + current_accuracy
  + safety_margin
```

### 9.2 同一スロットの統合

同じ20秒スロットに複数レポートが届いた場合：

1. 精度100m超を除外
2. BLE検出が古いレポートを除外
3. 既存軌跡から極端に外れる位置を除外
4. 残った位置を水平精度で重み付け
5. 中央値または重み付き中心を生位置とする

単一レポートでも受理可能だが、複数レポートが一致する場合は信頼度を上げる。

## 10. Google Mapsの道路へのマッピング

### 10.1 使用API

時系列の位置点にはGoogle Roads APIの`Snap to Roads`を使用する。単独点だけを処理する`Nearest Roads`ではなく、直近の連続軌跡を渡して道路の連続性を利用する。

Google Roads APIは1リクエストあたり最大100点を受け取り、道路上へ補正できる。入力点間は300m以内を目標とする。

### 10.2 入力バッファ

直近2〜5分、最大20点程度の統合済み生位置を保持する。

```text
P1 → P2 → ... → Pn
        ↓
Google Roads API snapToRoads
        ↓
S1 → S2 → ... → Sn
```

推奨：

- `interpolate=false`を初期値とする
- 生位置と道路補正位置の距離を保存
- 最新点に対応する`originalIndex`を優先
- API応答を短時間キャッシュ

### 10.3 誤スナップ防止

祭りでは通常の車両交通とは異なる場所を通る可能性があるため、常に道路補正を正としない。

道路補正を採用する条件：

- 生位置から道路までの距離が許容範囲内
- 直前の道路セグメントから連続している
- 推定速度で到達可能
- 急激な道路ジャンプがない

初期値：

- 生位置と道路補正位置の最大差：50m
- 直前セグメントから不連続なら保留
- 2回連続で同じ道路系列を支持したら切り替え

不採用時は生位置または平滑化位置を表示し、道路補正信頼度を下げる。

### 10.4 API障害時

Google Roads APIの失敗で位置公開全体を止めない。

- 最後の道路セグメントへ局所投影
- 生位置を平滑化して表示
- バックグラウンドで再試行
- 公開データに`snapStatus`を含める

## 11. 進行方向推定

### 11.1 原則

個々のスマートフォンが返す`course`や端末方位は、その参加者の向きを示す可能性があるため、山笠の方向として直接採用しない。

進行方向は、道路補正済み位置の時系列からバックエンドで推定する。

### 11.2 推定方法

直近30〜90秒の道路補正済み位置を使用する。

1. 精度・鮮度の悪い点を除外
2. 連続する道路補正点を取得
3. 直近点群へ時間加重線形回帰または移動ベクトル平均を適用
4. 最新セグメントの道路接線方向を候補とする
5. 過去の進行方向に近い向きを選ぶ
6. 急な反転は複数更新で確認する

道路には双方向の接線があるため、直前の位置から現在位置への移動ベクトルとの内積が正になる方向を採用する。

### 11.3 更新条件

位置誤差に対して移動距離が小さい場合、方向は更新しない。

推奨初期値：

- 方向算出窓：60秒
- 最低移動距離：20m、または位置精度中央値の1.5倍
- 最大1更新あたりの方向変化：45度
- 120度超の反転：2回以上の連続支持で確定

### 11.4 信頼度

```text
heading_confidence =
  position_quality
  × road_snap_quality
  × traveled_distance_quality
  × temporal_consistency
```

公開値：

- `HIGH`: 矢印を明確に表示
- `MEDIUM`: 矢印を薄く表示
- `LOW`: 方向を表示しない

停止または移動不足の場合、最後の方向を現在方向として断定しない。

## 12. 公開状態モデル

```json
{
  "yamakasaId": 1,
  "eventStatus": "ACTIVE",
  "activeDeviceCount": 24,
  "location": {
    "latitude": 33.5900,
    "longitude": 130.4010,
    "accuracyM": 22,
    "source": "ROAD_SNAPPED",
    "updatedAt": "2026-07-21T05:10:44Z"
  },
  "heading": {
    "degrees": 84,
    "confidence": "HIGH"
  },
  "snap": {
    "status": "SNAPPED",
    "roadPlaceId": "...",
    "distanceFromRawM": 8
  }
}
```

観客向けAPIでは個々の端末情報や生位置レポートを公開しない。

## 13. API設計

### モバイル用

```text
POST /v1/presence
GET  /v1/yamakasa/{id}/relay-state
POST /v1/yamakasa/{id}/reports
POST /v1/tag-observations/verify
```

### 公開用

```text
GET /v1/public/yamakasa/{id}
GET /v1/public/yamakasa/{id}/stream
```

### 管理用

```text
GET  /v1/admin/yamakasa/{id}/health
PATCH /v1/admin/yamakasa/{id}/config
GET  /v1/admin/yamakasa/{id}/relay-metrics
```

## 14. データストア

### 短期状態

Redis等：

- 端末presenceリース
- 最新generation
- 最新位置
- スロット状態
- 短期軌跡
- 道路スナップキャッシュ

### 永続化

PostgreSQL等：

- 山笠・タグ設定
- イベント設定
- 統合済み位置履歴
- 道路補正結果
- 開催状態遷移
- 運用メトリクス

個々の端末生位置は短期保持し、統合後に早期削除する。

## 15. 推奨技術構成

- モバイル：iOS Swift / Core Bluetooth / Core Location、Android Kotlin / BLE APIs / Fused Location Provider
- API：GoまたはTypeScript
- 実行基盤：Cloud Run等
- 短期状態：Redis / Memorystore
- 永続化：PostgreSQL / Cloud SQL
- 地図・道路：Google Maps JavaScript API、Google Roads API
- リアルタイム配信：Server-Sent Events

ネイティブのバックグラウンドBLE・位置処理が必要なため、初期段階では完全なWebアプリのみの構成は採用しない。

## 16. 監視指標

- 有効端末数
- 端末1台あたりのpresence回数
- 端末1台あたりの位置取得回数
- 端末1台あたりの位置送信回数
- 位置更新間隔p50 / p95 / p99
- 同一スロット重複数
- 位置取得失敗率
- BLE検出から位置更新までの時間
- 開催判定の状態遷移回数
- Roads API成功率・遅延・費用
- 道路補正距離
- 進行方向信頼度
- iOS / Android別の成功率

## 17. 初期パラメータ

| 項目 | 初期値 |
|---|---:|
| 開催閾値 | 有効端末10台 |
| 開催確定 | 10台以上が60秒継続 |
| 非開催確定 | 10台未満が180秒継続 |
| presence送信 | 30〜60秒 |
| presenceリース | 90秒 |
| 位置更新スロット | 20秒 |
| バックオフ | 0〜15秒 |
| 同一端末クールダウン | 5分 |
| キャッシュ位置許容 | 30秒・50m以内 |
| 単発位置タイムアウト | 5〜8秒 |
| 最大位置精度 | 100m |
| 方向算出窓 | 60秒 |
| 道路補正最大差 | 50m |
| 物理速度上限 | 12m/s |

## 18. 実装フェーズ

### Phase 1: 前景動作による成立性検証

- BLEタグ検出
- 10〜30端末の動的参加・離脱
- 分散バックオフ
- 単発位置取得
- 開催判定
- 生位置のWeb表示

### Phase 2: バックグラウンド動作

- iOSバックグラウンドBLE
- AndroidバックグラウンドBLE
- 画面ロック・低電力モード
- 電池消費測定
- 端末ごとの公平性

### Phase 3: 位置品質

- 複数レポート統合
- Google Roads API連携
- 誤スナップ抑制
- 進行方向推定
- 信頼度表示

### Phase 4: 本番運用

- セキュリティ・attestation
- 監視・管理画面
- API費用最適化
- データ保持・削除方針
- 本番端末数での負荷試験

## 19. 成功条件

- 10台以上の有効端末を安定して検出し、開催状態へ遷移できる
- 端末群全体で20〜30秒間隔の位置更新を維持する
- 個々の端末の位置取得は平均5〜15分に1回以下
- 端末の出入りや一部端末の停止で位置更新が途切れない
- 道路補正が不自然な道路ジャンプを起こさない
- 進行方向を信頼度付きで提供できる
- Google Roads API障害時も生位置でサービスを継続する
- 観客向けAPIへ個々の参加者情報を公開しない
