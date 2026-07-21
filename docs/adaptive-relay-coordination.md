# 適応型分散送信設計

## 1. 位置づけ

本書は [ソフトウェア設計](software-design.md) の送信端末選出部分を詳細化する補助設計である。用語、API URI、測位品質基準、開催判定時間が異なる場合は `software-design.md` を優先する。

## 2. 目的

開催中に参加端末数が大きく増減しても、位置更新の精度と可用性を維持しながら、個々の端末と端末群全体の電池消費を抑える。

固定候補確率は採用しない。参加端末が増えるほど候補確率を下げ、減るほど候補確率を上げる。

## 3. 設計原則

- 1スロット当たりの期待候補数を概ね1〜3台に保つ
- 厳密な単一リーダー選出を行わず、少数の重複を許容する
- 候補判定をローカルで行い、送信前のサーバー照会を要求しない
- 非候補端末は位置取得と位置送信を行わない
- 開催判定の遅延を許容し、presence通信を低頻度にする
- 候補端末には必要時の単発高精度測位を許可する

## 4. 参加端末数の推定

各端末は、低頻度に配布される `estimated_relay_count` を保持する。この値は瞬間人数ではなく、直近数分のpresenceと位置レポートから算出した平滑化値とする。

```text
smoothed_count = EWMA(observed_active_relays)
```

初期値：

- presence更新：5分
- presenceリース：10分
- 人数推定更新：1〜5分
- クライアント設定有効期限：10分

設定取得だけの定期GETは行わない。次の既存通信の応答へ設定を同梱する。

- presence送信
- 位置レポート送信
- アプリ起動時の設定取得
- OSが許可した低頻度バックグラウンド同期

設定が期限切れの場合は、最後の値または安全な既定値を使う。

## 5. 適応型候補確率

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

| 推定端末数 | 候補確率 | 期待候補数 |
|---:|---:|---:|
| 5 | 40% | 2.0 |
| 10 | 20% | 2.0 |
| 20 | 10% | 2.0 |
| 50 | 4% | 2.0 |
| 100 | 3% | 3.0 |

候補判定：

```text
u = Uniform01(Hash(resource_id, slot_id, relay_session_id))
is_candidate = u < adjusted_probability
```

同じ入力に対する結果は常に同じとする。

## 6. 端末状態による補正

```text
adjusted_probability = candidate_probability
  * cooldown_factor
  * battery_factor
  * power_mode_factor
  * ble_quality_factor
  * location_quality_factor
  * network_factor
```

係数例：

| 状態 | 係数例 |
|---|---:|
| 直近5分に送信済み | 0.1 |
| 直近10分に送信なし | 1.0 |
| 充電中 | 1.5 |
| 低電力モード | 0.2 |
| バッテリー20%未満 | 0.2 |
| 複数タグを安定検出 | 1.2 |
| タグ検出が弱い・古い | 0〜0.5 |
| 高品質な最終既知位置あり | 1.2 |
| ネットワーク不良 | 0.3 |

最終確率は再び許容範囲内へ制限する。ただし補正後の確率下限を全端末へ強制すると低品質端末まで候補になるため、BLE近傍性や権限不足で不適格な端末は確率0を許容する。

## 7. 人数の急増・急減

### 7.1 急増

推定値が追従するまで一時的に重複レポートが増える。サーバーは同一スロットの重複数を観測し、`estimated_relay_count`を上方補正する。

短期間の重複は障害ではなく、位置統合の品質向上へ利用する。

### 7.2 急減

候補不在スロットが連続した場合、サーバーは既存通信の応答へ短期設定を含める。

```text
candidate_multiplier = 2.0
expires_at = short TTL
```

全端末への即時プッシュには依存しない。設定未受信でも、最大候補確率と次スロットの再抽選により自然回復する。

### 7.3 安全な既定値

人数推定がない、または古すぎる場合：

- 推定10台として20%を使う
- 連続欠落を検出した端末は次スロットだけ確率を段階的に上げてよい
- 重複増加より位置欠落回避を優先する

## 8. 開催判定

位置更新と分離する。

```text
estimated_relay_count >= 10 → ACTIVE候補
estimated_relay_count < 10  → INACTIVE候補
```

- `INACTIVE → ACTIVE`: 10台以上を約5分観測
- `ACTIVE → INACTIVE`: 10台未満を10〜15分観測

開催判定のために30〜60秒ごとの通信を行わない。位置レポート送信時はpresence更新を兼ねる。

## 9. 位置精度優先の測位

候補端末だけが測位する。

1. OSの最終既知位置を確認
2. 20秒以内かつ水平精度20m以内なら採用
3. 条件を満たさなければ単発高精度測位を要求
4. 取得時間上限は8〜12秒
5. 水平精度30m以内を通常採用
6. 30〜50mは複数レポートとの統合候補
7. 50m超は原則不採用
8. 測位完了後は位置更新を停止

最終既知位置は省電力な高速経路であり、古い位置を無条件に採用するものではない。

## 10. 重複送信

同一スロットの複数レポートを正常動作として扱う。

1. 認証、BLE近傍性、時刻を検証
2. 水平精度50m超を原則除外
3. 物理的に不可能なジャンプを除外
4. 外れ値を除外
5. 近接位置を精度と鮮度で重み付け統合
6. 道路スナップと進行方向推定へ渡す

重複をゼロにするためのサーバー照会や分散ロックは導入しない。

## 11. API URI

APIは追跡対象を `resources` として一般化する。

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

`relay-config`は各スロットで呼び出さず、起動時・復旧時・既存通信への応答同梱を基本とする。

## 12. 初期パラメータ

| 項目 | 初期値 |
|---|---:|
| 位置スロット | 30秒 |
| 目標候補数 | 2台／スロット |
| 候補確率範囲 | 3〜40% |
| 候補内待機 | 0〜10秒 |
| 同一端末クールダウン | 5〜10分 |
| presence更新 | 5分 |
| presenceリース | 10分 |
| 開催開始 | 10台以上を約5分 |
| 開催終了 | 10台未満を10〜15分 |
| 最終既知位置採用 | 20秒以内・精度20m以内 |
| 新規測位通常採用 | 30m以内 |
| 新規測位統合候補 | 30〜50m |
| 単発測位タイムアウト | 8〜12秒 |

## 13. 成功条件

- 参加端末数が10〜100台程度で変動しても、スロット当たり送信中央値が1〜3件
- 参加人数に比例して重複送信が増え続けない
- 急減後の連続欠落が限定的で自然回復する
- 送信前サーバーGETなしで動作する
- 非候補端末は位置取得と位置送信を行わない
- 最終既知位置を再利用しても公開位置精度が悪化しない
- 個々の端末の担当間隔が参加数に応じて長くなる