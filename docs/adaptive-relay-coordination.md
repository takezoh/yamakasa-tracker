# 適応型分散送信設計

## 1. 目的

開催中に参加端末数が大きく増減しても、位置更新の精度と可用性を維持しながら、端末群全体および個々の端末の電池消費を抑える。

固定の候補確率は採用しない。参加端末が増えるほど候補確率を下げ、減るほど候補確率を上げる。

## 2. 設計原則

- 端末数が変化しても、1スロットあたりの期待送信候補数を概ね1〜3台に保つ
- 厳密な単一リーダー選出は行わず、少数の重複を許容する
- 候補判定は原則ローカルで行い、送信前のサーバー照会を要求しない
- 位置取得とネットワーク送信は候補端末だけが行う
- 開催・非開催判定の遅延は許容し、presence通信を低頻度にする
- 位置精度を確保するため、候補端末には必要時の単発高精度測位を許可する

## 3. 参加端末数の推定

各端末は、サーバーから低頻度に配布される推定参加端末数 `estimated_relay_count` を保持する。

この値は厳密な瞬間人数ではなく、直近数分のpresenceおよび位置レポートから算出した平滑化値でよい。

```text
smoothed_count = EWMA(observed_active_relays)
```

推奨初期値：

- presence更新：5分ごと
- presenceリース：10分
- 人数推定更新：1〜5分ごと
- クライアント設定有効期限：10分

端末は設定更新のためだけに定期GETを行わない。次の既存通信の応答へ設定を同梱する。

- presence送信応答
- 位置レポート送信応答
- アプリ起動時の設定取得
- OSが許可したバックグラウンド同期

設定が期限切れでも、最後の値または安全な既定値で動作する。

## 4. 適応型候補確率

1スロット当たりの目標候補数を `target_candidates` とする。

```text
candidate_probability = clamp(
  target_candidates / max(estimated_relay_count, minimum_count),
  minimum_probability,
  maximum_probability
)
```

推奨初期値：

```text
target_candidates = 2
minimum_count = 5
minimum_probability = 0.03
maximum_probability = 0.40
```

例：

| 推定端末数 | 候補確率 | 期待候補数 |
|---:|---:|---:|
| 5 | 40% | 2.0 |
| 10 | 20% | 2.0 |
| 20 | 10% | 2.0 |
| 50 | 4% | 2.0 |
| 100 | 3%下限 | 3.0 |

候補判定は決定論的疑似乱数で行う。

```text
u = Uniform01(Hash(resource_id, slot_id, relay_session_id))
is_candidate = u < adjusted_probability
```

同じ入力なら再計算しても結果が変わらない。

## 5. 端末状態による補正

候補確率へ以下の係数を掛ける。

```text
adjusted_probability = candidate_probability
  * cooldown_factor
  * battery_factor
  * power_mode_factor
  * ble_quality_factor
  * location_quality_factor
  * network_factor
```

推奨例：

| 状態 | 係数例 |
|---|---:|
| 直近5分に送信済み | 0.1 |
| 直近10分に送信なし | 1.0 |
| 充電中 | 1.5 |
| 低電力モード | 0.2 |
| バッテリー20%未満 | 0.2 |
| 複数タグを安定検出 | 1.2 |
| タグ検出が弱い・古い | 0〜0.5 |
| 高品質なキャッシュ位置あり | 1.2 |
| ネットワーク不良 | 0.3 |

最終確率は再度上限・下限で制限する。

## 6. 人数推定が古い場合

参加人数が急増した直後は候補数が一時的に増える。急減した直後は候補不在スロットが増える可能性がある。

これを許容しつつ、次の方法で自己修正する。

### 6.1 急増時

サーバーは同一スロットの重複レポート数を観測する。

```text
重複が継続的に多い
→ estimated_relay_countを上方補正
→ 次回配布時に候補確率を低下
```

重複レポートは最良位置の選択や位置統合に利用できるため、短期間の増加は障害ではない。

### 6.2 急減時

位置更新の欠落スロットが連続した場合、サーバーは応答に探索強化設定を含める。

```text
candidate_multiplier = 2.0
expires_at = short TTL
```

設定を受け取った端末は一時的に候補確率を上げる。

ただし、欠落通知を全端末へ即時プッシュする設計には依存しない。設定を受信できない場合でも、端末側の最大候補確率と次スロットの再抽選で自然回復する。

## 7. 開催判定

開催判定のタイミングは厳密でなくてよいため、位置更新とは分離する。

暫定条件：

```text
estimated_relay_count >= 10 → ACTIVE候補
estimated_relay_count < 10  → INACTIVE候補
```

推奨ヒステリシス：

- `INACTIVE → ACTIVE`：10台以上を5分程度観測
- `ACTIVE → INACTIVE`：10台未満を10〜15分程度観測

presenceは位置取得を伴わず、5分程度に1回とする。位置レポート送信時はpresence更新を兼ねる。

開催判定のために30〜60秒ごとの通信を行わない。

## 8. 位置精度優先の測位方針

候補端末だけが測位する。

1. OSの最終既知位置を確認する
2. 20秒以内かつ水平精度20m以内なら採用候補とする
3. 条件を満たさない場合は単発の高精度位置取得を要求する
4. 取得時間上限は8〜12秒程度とする
5. 水平精度50m以内を通常採用、50〜100mは他候補がない場合のみ低信頼度で採用する
6. 取得完了後は位置更新を停止する

キャッシュ位置は省電力な高速経路であり、古い位置を無条件に採用するものではない。

## 9. 重複送信の扱い

同一スロットで複数端末から届くことを正常動作として扱う。

バックエンドは次の順で処理する。

1. 認証・タグ近傍性・時刻を検証
2. 水平精度の悪いレポートを除外
3. 物理的に不可能なジャンプを除外
4. 近接する複数位置を精度で重み付け統合
5. 道路スナップと進行方向推定へ渡す

重複をゼロにするためのサーバー照会やロックは導入しない。

## 10. 一般化したAPI URI

URIは山笠固有の語を使わず、追跡対象を `resources`、中継端末を `relays` として一般化する。

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

位置レポートの例：

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
  ]
}
```

## 11. 初期パラメータ

| 項目 | 初期値 |
|---|---:|
| 位置スロット | 30秒 |
| 目標候補数 | 2台／スロット |
| 候補確率範囲 | 3〜40% |
| 同一端末クールダウン | 5〜10分 |
| presence更新 | 5分 |
| presenceリース | 10分 |
| 開催開始判定 | 10台以上を5分 |
| 開催終了判定 | 10台未満を10〜15分 |
| キャッシュ位置採用 | 20秒以内・精度20m以内 |
| 単発測位タイムアウト | 8〜12秒 |
| 通常採用精度 | 50m以内 |

## 12. 成功条件

- 参加人数が10〜100台程度で変動しても、1スロット当たりの送信レポート中央値が1〜3件に収まる
- 参加人数増加に比例して重複送信数が増え続けない
- 端末数の急減後も、連続欠落が限定的で自然回復する
- 送信前のサーバーGETなしで動作する
- 非候補端末は位置取得と位置送信を行わない
- 個々の端末の平均担当間隔が参加端末数に応じて長くなる
- 開催判定通信が位置更新通信より大きな電池負荷にならない
