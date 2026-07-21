# ソフトウェア設計

## 1. 目的と優先順位

山笠に搭載した4個のBLEタグを周囲の主催者スマートフォン群が検出し、端末間で位置取得と送信を分散する。個々の端末のバッテリー消費を抑えながら、観客向けWebへ現在位置、開催状態、進行方向を公開する。

優先順位は次の通り。

1. 位置精度
2. 個々の端末と端末群全体の省電力
3. 位置更新の継続性
4. 開催・非開催判定の即時性

開催・非開催判定は数分単位の遅延を許容する。判定のための頻繁な通信を行わず、少数の選出端末による高品質な測位へ電力予算を使う。

## 2. 用語と識別子

ハードウェア広告では、山笠を表す固定公開IDを `yamakasa_id` と呼ぶ。バックエンドとAPIでは追跡対象を一般化して `resource_id` / `resourceId` と呼ぶ。

```text
yamakasa_id ── server-side mapping ──> resource_id
node_id     ─────────────────────────> tag observation node_id
```

- `yamakasa_id`: BLE広告上の固定公開ID
- `node_id`: `front` / `rear` / `left` / `right`
- `resource_id`: API・保存・公開処理で使う追跡対象ID
- `relay_session_id`: 数時間またはイベント単位の匿名端末ID

## 3. システム構成

```text
4個のBLEタグ
    ↓ BLE広告
主催者モバイルアプリ群
    ├── BLE検出
    ├── ローカル候補判定
    ├── 選出端末だけ単発測位
    └── 少数の重複を許容して位置送信
            ↓ HTTPS
バックエンド
    ├── 低頻度presence集約
    ├── 推定参加端末数・開催状態
    ├── 位置レポート検証・統合
    ├── Google Roads APIによる道路スナップ
    ├── 進行方向推定
    └── 公開状態生成
            ↓ SSE / polling
観客向けWeb
    ├── Google Maps
    ├── 道路補正済み推定位置
    ├── 進行方向と信頼度
    └── 開催状態・更新時刻
```

## 4. モバイルアプリ

```text
RelayApp
├── BleScanner
├── TagPresenceTracker
├── AdaptiveCandidateSelector
├── LocationProvider
├── ReportUploader
├── LocalEnergyPolicy
└── Diagnostics
```

### 4.1 BLE近傍判定

4タグを1つの対象として統合する。

```text
ABSENT
  ↓ 10秒以内に正規タグを2回以上検出
ELIGIBLE
  ↓ 90秒間検出なし
ABSENT
```

送信直前には、いずれかの正規タグを直近10秒以内に複数回検出していることを再確認する。RSSIは品質評価には使うが、単独で距離や参加状態を決定しない。

### 4.2 プライバシー

- 恒久的な端末識別子を送信しない
- `relay_session_id`は数時間またはイベント単位で更新する
- タグ非検出中の位置は取得・送信しない
- 個人の移動履歴を保存しない

## 5. 適応型の送信端末決定

詳細な数式と急増・急減時の振る舞いは [適応型分散送信設計](adaptive-relay-coordination.md) を正本とする。

### 5.1 設計原則

- サーバーは毎スロット送信端末を指名しない
- 全端末による送信前GETやロックを行わない
- 各端末がローカルで候補判定する
- 1スロット当たり1〜3台程度の重複を正常動作として許容する
- 参加人数の増減に応じて候補確率を変える

### 5.2 時間スロット

初期値は30秒とする。

```text
slot_id = floor(current_time / 30 seconds)
```

位置更新目標は30秒前後とし、実地試験後に20〜60秒で調整する。

### 5.3 候補確率

```text
candidate_probability = clamp(
  target_candidates / max(estimated_relay_count, minimum_count),
  minimum_probability,
  maximum_probability
)
```

初期値：

```text
target_candidates = 2
minimum_count = 5
minimum_probability = 0.03
maximum_probability = 0.40
```

| 推定参加端末数 | 候補確率 | 期待候補数 |
|---:|---:|---:|
| 5 | 40% | 2.0 |
| 10 | 20% | 2.0 |
| 20 | 10% | 2.0 |
| 50 | 4% | 2.0 |
| 100 | 3% | 3.0 |

候補判定は決定論的疑似乱数で行う。

```text
u = Uniform01(Hash(resource_id, slot_id, relay_session_id))
is_candidate = u < adjusted_probability
```

### 5.4 端末状態による補正

候補確率を下げる条件：

- 直近5〜10分以内に送信済み
- 低電力モード中
- バッテリー残量が少ない
- BLE検出が弱い、古い、または不安定
- ネットワーク状態が悪い
- 必要な権限が不足

候補確率を上げる条件：

- 複数タグを安定検出
- 最近送信していない
- 充電中
- 高品質な最終既知位置を保持
- 通信状態が良い

### 5.5 候補内バックオフ

候補端末は0〜10秒の決定論的待機時間を持つ。

```text
delay = Hash(resource_id, slot_id, relay_session_id, "delay") mod 10 seconds
```

待機後にBLE近傍性を再確認し、事前GETなしで測位とPOSTへ進む。

## 6. 位置取得と省電力

### 6.1 最終既知位置の再利用

OSが既に保持する最終既知位置は、新しくGNSSを起動せず利用できる。ただし次の全条件を満たす場合に限る。

- 取得から20秒以内
- 水平精度20m以内
- BLEタグを直近10秒以内に複数回検出
- 前回採用位置から物理的に不自然でない

### 6.2 単発高精度測位

最終既知位置が条件を満たさない場合、候補端末だけが単発測位する。

1. 高精度の単発位置を要求
2. 最大8〜12秒待つ
3. 水平精度30m以内は通常採用
4. 30〜50mは複数レポートとの統合候補
5. 50m超は原則不採用
6. 取得後は位置更新を直ちに停止

常時測位、常時ナビゲーション、全端末の定期測位は採用しない。

### 6.3 電力削減原則

- 大多数の端末はBLE監視とローカル抽選だけを行う
- 非候補端末は位置を評価しない
- 候補端末だけが最終既知位置を評価する
- 新規高精度測位は必要な候補端末だけが行う
- 送信前の競合確認GETは行わない
- 位置POSTとpresence更新を同一通信へまとめる

## 7. 位置レポート

```json
{
  "reportId": "uuid",
  "relaySessionId": "rotating-id",
  "resourceId": "public-resource-id",
  "slotId": 123456,
  "observedAt": "timestamp",
  "location": {
    "latitude": 0.0,
    "longitude": 0.0,
    "horizontalAccuracyM": 12.0,
    "ageMs": 4500,
    "source": "fresh-single-fix"
  },
  "tagObservations": [
    {
      "nodeId": "front",
      "lastSeenAgeMs": 900,
      "sampleCount": 6,
      "rssiMedian": -65
    }
  ],
  "batteryClass": "normal",
  "lowPowerMode": false
}
```

`location.source`:

- `last-known`
- `fresh-single-fix`
- `passive-system-location`

同一スロットの重複POSTは正常動作として扱う。

## 8. 開催・非開催判定

開催判定は位置更新から分離し、即時性より省電力を優先する。

### 8.1 presence

送信タイミング：

- BLEタグを新たに検出したとき
- 位置レポート送信時
- 前回通信から約5分経過したとき
- BLE検出・権限状態が大きく変わったとき

退出通知は必須とせず、リース期限切れで処理する。

初期値：

- presence更新：5分
- presenceリース：10分
- 人数推定更新：1〜5分
- 設定有効期限：10分

### 8.2 暫定開催判定

```text
estimated_relay_count >= 10 → ACTIVE候補
estimated_relay_count < 10  → INACTIVE候補
```

- `INACTIVE → ACTIVE`: 10台以上を約5分観測
- `ACTIVE → INACTIVE`: 10台未満を10〜15分観測

開始・終了表示の遅延は許容する。位置公開は開催状態に依存せず継続できる。

## 9. バックエンド位置統合

### 9.1 検証

- 正規アプリと認証済みタグ
- BLE検出の鮮度
- 位置取得時刻と水平精度
- 想定最高速度を超える不自然なジャンプ
- 同一端末からの異常頻度
- 同一スロット内の他レポートとの整合性

### 9.2 統合

1. 精度50m超を原則除外
2. 時系列上不可能な位置を除外
3. 他レポート群から大きく外れる点を除外
4. 残った位置を精度と鮮度で重み付け
5. 最良点または重み付き中心を生推定位置とする

単一端末の精度値だけでなく、レポート間の分散も公開信頼度へ反映する。

## 10. 道路スナップ

直近3〜10点の生推定位置列を、時刻順にGoogle Roads APIのSnap to Roadsへ渡す。

採用条件：

- 元位置から道路までの距離が水平精度と整合する
- 前回採用道路と連続、または合理的に接続可能
- 最高速度を超える道路ジャンプがない
- 平行道路では過去経路との連続性を優先
- 道路外を通る正当な区間では生位置または平滑化位置を使う

`interpolate=true`は表示軌跡用途に限定する。Google Roads API障害時も位置公開を継続する。

## 11. 進行方向推定

スマートフォン個別のheadingやbearingは採用しない。道路補正済み位置の時系列から推定する。

1. 直近30〜90秒の採用位置を取得
2. 外れ値を除外
3. 道路上の移動距離が位置誤差を十分上回るか確認
4. 最古点から最新点への道路接線方向を算出
5. 過去方向と平滑化
6. 信頼度を付与

信頼度：

- `HIGH`: 複数点が同一道路上で一貫し、移動距離が誤差を十分上回る
- `MEDIUM`: 方向は推定可能だが位置分散がある
- `LOW`: 交差点、平行道路、移動距離不足、位置誤差大
- `UNKNOWN`: 方向を表示しない

低信頼時は誤った矢印を出さない。

## 12. 一般化API URI

```text
POST /v1/resources/{resourceId}/presence
POST /v1/resources/{resourceId}/location-reports
GET  /v1/resources/{resourceId}/public-state
GET  /v1/resources/{resourceId}/stream
GET  /v1/resources/{resourceId}/relay-config
```

管理API：

```text
POST  /v1/resources
GET   /v1/resources/{resourceId}
PATCH /v1/resources/{resourceId}
GET   /v1/resources/{resourceId}/diagnostics
```

送信候補決定のための状態照会APIは設けない。`relay-config`はアプリ起動時や既存通信の応答に同梱する設定の取得・復旧用途であり、各スロットで呼び出さない。

公開応答例：

```json
{
  "resourceId": "public-resource-id",
  "status": "ACTIVE",
  "position": {
    "latitude": 0.0,
    "longitude": 0.0,
    "accuracyM": 16,
    "source": "road-snapped",
    "updatedAt": "timestamp"
  },
  "heading": {
    "degrees": 82,
    "confidence": "HIGH"
  }
}
```

## 13. 初期パラメータ

| 項目 | 初期値 |
|---|---:|
| 位置スロット | 30秒 |
| 目標候補数 | 2台／スロット |
| 候補確率範囲 | 3〜40% |
| 候補内待機 | 0〜10秒 |
| 同一端末送信クールダウン | 5〜10分 |
| 最終既知位置の採用 | 20秒以内・精度20m以内 |
| 新規測位の通常採用 | 30m以内 |
| 統合候補 | 30〜50m |
| 新規測位タイムアウト | 8〜12秒 |
| presence更新 | 5分 |
| presenceリース | 10分 |
| ACTIVE開始 | 10台以上を約5分 |
| INACTIVE開始 | 10台未満を10〜15分 |
| 方向推定窓 | 30〜90秒 |

## 14. 検証指標

- 参加端末数10〜100台でのスロット当たり送信件数
- 端末1台当たりの測位回数、通信回数、wake回数
- 位置精度50・90・95パーセンタイル
- 道路スナップ誤道路率
- 進行方向誤差と信頼度別精度
- 参加人数急増・急減後の欠落時間と重複数
- iOS／Android、機種、低電力モード別の消費差

最適化対象は送信台数の最小化そのものではない。観客へ表示する位置精度を維持しながら、個々の端末と端末群全体の測位・通信・wake回数を最小化する。