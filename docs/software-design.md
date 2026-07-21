# ソフトウェア設計

## 1. 目的と優先順位

山笠に搭載した4個のBLEタグを周囲の主催者スマートフォン群が検出し、端末間で位置取得と送信を分散する。個々の端末のバッテリー消費を抑えながら、観客向けWebへ現在位置、開催状態、進行方向を公開する。

優先順位：

1. 位置精度
2. 個々の端末と端末群全体の省電力
3. 位置更新の継続性
4. 開催・非開催判定の即時性

開催・非開催判定は数分単位の遅延を許容する。判定のための頻繁な通信を行わず、少数の選出端末による高品質な測位へ電力予算を使う。

## 2. 識別子

追跡対象の識別子は、BLE広告、アプリ、API、DB、ログの全層で`resource_id`に統一する。

- 論理型：unsigned 64-bit
- BLE広告：8-byte big-endian binary
- API・JSON・ログ：16桁の小文字16進文字列
- DB：`BIGINT`相当または8-byte binary

```text
BLE:  01 af 34 c9 81 2d e7 04
API:  "01af34c9812de704"
```

`yamakasa_id`、`beacon_id`、`resource_ref`などの別名や変換レイヤーは設けない。

`node_id`は同じ`resource_id`に属する物理タグを識別する。

- `front`
- `rear`
- `left`
- `right`

## 3. システム構成

```text
BLEタグ群
    ↓ BLE広告(resource_id, node_id, auth_tag...)
主催者モバイルアプリ群
    ├── BLE検出
    ├── ローカル候補判定
    ├── 選出端末だけ単発測位
    └── 少数重複を許容して位置送信
            ↓ HTTPS
バックエンド
    ├── presence集約
    ├── 開催状態判定
    ├── 位置検証・統合
    ├── Google Roads APIによる道路補正
    ├── 進行方向推定
    └── 公開状態生成
            ↓ SSE / polling
観客向けWeb
    ├── Google Maps
    ├── 推定位置
    ├── 進行方向
    └── 開催状態・更新時刻
```

## 4. モバイルアプリ

```text
RelayApp
├── BleScanner
├── TagPresenceTracker
├── CandidateSelector
├── LocationProvider
├── ReportUploader
├── LocalEnergyPolicy
└── Diagnostics
```

### 4.1 タグ検出

4タグは同じ`resource_id`と異なる`node_id`を持つ。アプリは4タグを1つの追跡対象として統合する。

```text
ABSENT
  ↓ 10秒以内に正規タグを2回以上検出
ELIGIBLE
  ↓ 90秒間検出なし
ABSENT
```

送信直前には、いずれかの正規タグを直近10秒以内に複数回検出していることを必須とする。RSSIは品質評価には使うが、単独で距離や参加状態を決めない。

### 4.2 匿名端末ID

- `relay_session_id`はイベントまたは数時間単位で生成
- 氏名、電話番号、広告IDなどの恒久識別子は送信しない
- タグ非検出中は位置を取得・送信しない
- 個人の移動履歴を保持しない

## 5. 送信端末の決定

### 5.1 方針

サーバーは毎回送信端末を指名しない。全候補端末による事前GETや厳密な排他制御も行わない。

各端末がローカルで自分が送信候補かを判定し、1スロットあたり期待1〜3台の重複を許容する。

### 5.2 時間スロット

初期値は30秒。

```text
slot_id = floor(current_time / 30 seconds)
```

位置更新目標は30秒前後とし、実地試験後に20〜60秒で調整する。

### 5.3 適応型候補確率

固定確率は使わない。低頻度に配布された推定参加端末数に応じて候補率を変える。

```text
candidate_probability = clamp(
  target_candidates / max(estimated_relay_count, minimum_count),
  minimum_probability,
  maximum_probability
)
```

初期値：

- `target_candidates = 2`
- `minimum_count = 5`
- `minimum_probability = 0.03`
- `maximum_probability = 0.40`

候補判定：

```text
u = Uniform01(Hash(resource_id, slot_id, relay_session_id))
is_candidate = u < adjusted_probability
```

### 5.4 端末状態による補正

候補率を下げる条件：

- 直近5〜10分に送信済み
- 低電力モード
- バッテリー残量が少ない
- BLE検出が弱い・古い
- ネットワーク状態が悪い
- 必要な権限がない

候補率を上げる条件：

- 複数タグを安定検出
- 最近送信していない
- 充電中
- 高品質な最終既知位置がある
- 通信状態が良い

### 5.5 候補内バックオフ

候補端末は0〜10秒の決定論的待機時間を持つ。

```text
delay = Hash(resource_id, slot_id, relay_session_id, "delay") mod 10 seconds
```

待機後にBLE検出条件を再確認し、事前GETなしで測位とPOSTへ進む。

## 6. 位置取得と省電力

### 6.1 最終既知位置

OSが直前に取得して保持している位置を再利用できるが、古さと精度を厳しく評価する。

採用条件：

- 取得から20秒以内
- 水平精度20m以内
- BLEタグを直近10秒以内に複数回検出
- 前回公開位置から物理的に不自然でない

### 6.2 単発測位

条件を満たす最終既知位置がない場合、候補端末だけが単発の高精度測位を行う。

1. 最大8〜12秒待つ
2. 30m以内：通常採用
3. 30〜50m：複数レポートとの統合候補
4. 50m超：原則不採用
5. 取得後は位置更新を直ちに停止

常時高精度測位、常時ナビゲーション、全端末の定期測位は採用しない。

## 7. presenceと開催判定

開催判定の即時性は要求しない。presenceは位置取得を伴わず、以下のタイミングに限定する。

- BLEタグを新たに検出したとき
- 位置レポート送信時
- 前回通信から約5分経過したとき
- BLE検出状態や権限状態が大きく変化したとき

退出通知は必須とせず、リース期限切れで処理する。

初期値：

- presence更新：5分
- presenceリース：10分
- `INACTIVE → ACTIVE`：推定有効端末10台以上を約5分
- `ACTIVE → INACTIVE`：10台未満を10〜15分

開催状態と位置公開処理は分離し、開催判定の遅延が位置更新を妨げないようにする。

## 8. API

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

`resourceId`はBLE広告の`resource_id`と同じ64-bit値を16進文字列で表現したものとする。

### 8.1 位置レポート

```json
{
  "reportId": "uuid",
  "relaySessionId": "rotating-id",
  "resourceId": "01af34c9812de704",
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
  ]
}
```

## 9. バックエンド位置統合

レポートごとに以下を検証する。

- 正規アプリと正規タグ
- `resource_id`と`node_id`の整合性
- BLE検出の鮮度
- 位置の取得時刻と精度
- 想定最高速度を超える不自然なジャンプ
- 同一端末からの異常頻度
- 同一スロット内の他レポートとの整合性

同一スロットに複数レポートがある場合：

1. 50m超を除外
2. 物理的に不可能な点を除外
3. 他レポート群から大きく外れる点を除外
4. 残った位置を精度と鮮度で重み付け
5. 最良点または重み付き中心を生推定位置とする

## 10. Google Maps道路補正

直近の生推定位置3〜10点を時刻順にGoogle Roads APIのSnap to Roadsへ渡す。

採用条件：

- 元位置から道路までの距離が位置精度と整合する
- 前回道路と連続している、または合理的に接続できる
- 最高速度を超える道路ジャンプがない
- 平行道路では過去経路との連続性を優先
- 道路外を通る正当な区間では生位置または平滑化位置へ戻す

Google Roads API障害時も生位置で公開を継続する。

## 11. 進行方向推定

スマートフォン個別のheadingやbearingは使わない。道路補正済み位置の直近30〜90秒から推定する。

1. 外れ値除去
2. 移動距離が位置誤差を十分上回るか確認
3. 道路上の最古点から最新点への接線方向を算出
4. 過去方向と平滑化
5. `HIGH`、`MEDIUM`、`LOW`、`UNKNOWN`の信頼度を付与

低信頼時は誤った矢印を表示せず、方向表示を省略する。

## 12. 初期パラメータ

| 項目 | 初期値 |
|---|---:|
| 位置スロット | 30秒 |
| 目標候補数 | 2台／スロット |
| 候補確率範囲 | 3〜40% |
| 同一端末クールダウン | 5〜10分 |
| 最終既知位置採用 | 20秒以内・精度20m以内 |
| 新規測位通常採用 | 30m以内 |
| 新規測位統合候補 | 30〜50m |
| 新規測位タイムアウト | 8〜12秒 |
| presence更新 | 5分 |
| presenceリース | 10分 |
| 開催開始 | 10台以上を約5分 |
| 開催終了 | 10台未満を10〜15分 |
| 方向推定窓 | 30〜90秒 |

## 13. 成功条件

- 参加人数が10〜100台程度で変動しても、1スロットの送信レポート中央値が1〜3件
- 参加人数増加に比例して重複送信が増え続けない
- 送信前のサーバーGETなしで動作する
- 非候補端末は位置取得・位置送信を行わない
- 位置精度を維持しながら端末ごとの測位・通信・wake回数を最小化する
- BLE、API、DB、ログの識別子が`resource_id`で一貫する
