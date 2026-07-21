# yamakasa-tracker

山笠に搭載したBLEタグを周囲の主催者スマートフォンが検出し、端末群で位置取得・送信を分担して、観客向けWebへ山笠の現在位置を公開するシステム。

## Primary constraint

**ハードウェアは山笠の外観を損なってはならない。**

これは無線性能、小型化、保守性、取り付け容易性と同列の要件ではなく、それらに優先する第一級要件である。タグ、筐体、ブラケット、ラベル、配線、固定具は通常の観覧位置から視認されず、山笠の意匠、造形、電飾、可動人形の見え方を変えてはならない。外観を損なわずに無線性能を満たせない配置案は不採用とし、配置、アンテナ、筐体、タグ数またはソフトウェア側を再設計する。

タグ自体はGPSやLTEを持たず、BLE広告だけを送信する。山笠1基には前・後・左・右を担当する4タグを配置し、巨大な構造物、人形の開閉、板・布、全面のLED・配線、周囲の人体による方向別遮蔽を相互補完する。ただし、4タグの存在は外観から認識できないことを前提とする。

## Documentation

- [ハードウェア要件](docs/hardware-requirements.md)
- [ハードウェア設計](docs/hardware-design.md)
- [タグ配置設計](docs/tag-placement.md)
- [プロトタイプ・ハードウェア方針](docs/prototype-hardware.md)
- [ソフトウェア設計](docs/software-design.md)
- [適応型分散送信設計](docs/adaptive-relay-coordination.md)

## Document authority

- ハードウェアの要求と受け入れ条件：`hardware-requirements.md`
- 回路、筐体、アンテナ、ファームウェア：`hardware-design.md`
- 山笠への年次配置と外観・無線試験：`tag-placement.md`
- 初期検証から製品化までの段階：`prototype-hardware.md`
- ソフトウェア全体、API、測位品質、開催判定：`software-design.md`
- 参加人数変動に対する候補選出の詳細：`adaptive-relay-coordination.md`

補助文書と全体設計が異なる場合は、上記の責務を持つ正本文書を優先する。

## Current hardware direction

- 最優先制約：山笠の外観・意匠を損なわない
- BLEタグ：4個／山笠
- 配置：前・後・左・右を担当する隠蔽配置
- 電源：タグごとにCR2032
- 動作期間：最低1週間
- 防水：IP67相当
- BLE広告：固定公開`yamakasa_id`＋方向別`node_id`＋認証コード
- GPS、LTE、加速度センサー：タグには搭載しない
- 初期プロトタイプ：一体型BLEモジュールまたは評価基板の内蔵アンテナ
- 製品化：小型一体型を基本とし、実測で必要な方向だけ分離FPCアンテナ化

## Current software direction

- APIと保存層では追跡対象を一般化して`resource_id` / `resourceId`と呼ぶ
- BLE広告の`yamakasa_id`はサーバー側で`resource_id`へ対応付ける
- 主催者アプリ群がBLEタグを検出し、位置取得と送信を分散
- 30秒スロット、目標候補数2台、候補確率3〜40%の適応型選出
- 送信前の競合確認GETや厳密な単一リーダー選出は行わない
- 最終既知位置は20秒以内・精度20m以内の場合だけ再利用
- 新規測位は30m以内を通常採用、30〜50mは統合候補、50m超は原則不採用
- presence更新5分、リース10分
- 暫定開催判定：10台以上を約5分で`ACTIVE`、10台未満を10〜15分で`INACTIVE`
- Google Roads APIで近傍道路へ補正
- 道路補正済み位置の30〜90秒時系列から進行方向を推定
- Google Roads API障害時は生位置または平滑化位置で継続

## Generalized API URI

```text
POST /v1/resources/{resourceId}/presence
POST /v1/resources/{resourceId}/location-reports
GET  /v1/resources/{resourceId}/public-state
GET  /v1/resources/{resourceId}/stream
GET  /v1/resources/{resourceId}/relay-config
```