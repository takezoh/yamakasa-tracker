# yamakasa-tracker

山笠に搭載したBLEタグを周囲の主催者スマートフォンが検出し、端末群で位置取得・送信を分担して、観客向けWebへ現在位置を公開するシステム。

## Primary constraint

**ハードウェアは山笠の外観を損なってはならない。**

これは無線性能、小型化、保守性、取り付け容易性より優先する第一級要件である。タグ、アンテナ、筐体、固定具、ラベルは通常の観覧位置から視認されず、意匠、造形、電飾、可動人形の見え方を変えてはならない。

## Documentation

- [ハードウェア要件](docs/hardware-requirements.md)
- [ハードウェア設計](docs/hardware-design.md)
- [タグ配置設計](docs/tag-placement.md)
- [プロトタイプ・ハードウェア方針](docs/prototype-hardware.md)
- [ソフトウェア設計](docs/software-design.md)
- [適応型分散送信設計](docs/adaptive-relay-coordination.md)
- [タグ・中継端末間の位置補正設計](docs/location-correction.md)
- [アプリ公開・配布設計](docs/app-distribution.md)

## Document authority

- システム全体、識別子、API、測位品質、開催判定の正本：`docs/software-design.md`
- 参加人数変動と候補端末選出の詳細：`docs/adaptive-relay-coordination.md`
- タグと中継端末の位置関係、複数位置統合、固定オフセット補正：`docs/location-correction.md`
- ストア公開、招待、権限、審査、配布運用：`docs/app-distribution.md`
- 物理要件の正本：`docs/hardware-requirements.md`
- 回路・電源・広告ペイロード・筐体：`docs/hardware-design.md`
- 年次の物理配置と試験：`docs/tag-placement.md`
- 試作段階の選択と移行条件：`docs/prototype-hardware.md`

補助文書が正本と異なる場合は正本を優先する。

## Identifier model

追跡対象の識別子は全層で`resource_id`に統一する。

- 論理型：unsigned 64-bit
- BLE広告：8-byte big-endian binary
- API・JSON・ログ：16桁の小文字16進文字列
- DB：`BIGINT`相当または8-byte binary

```text
BLE:  01 af 34 c9 81 2d e7 04
API:  "01af34c9812de704"
```

`yamakasa_id`、`beacon_id`、`resource_ref`などの別名や変換用IDは設けない。

同一の追跡対象に属する4タグは同じ`resource_id`を持ち、`node_id`で`front`、`rear`、`left`、`right`を区別する。

## Current hardware direction

- BLEタグ：4個／山笠
- 配置：前・後・左・右を担当する隠蔽配置
- 電源：タグごとにCR2032
- 動作期間：最低168時間
- 防水：IP67相当
- BLE広告：固定公開`resource_id`＋`node_id`＋認証コード
- GPS、LTE、加速度センサー：タグには搭載しない
- 初期プロトタイプ：一体型BLEモジュールまたは評価基板の内蔵アンテナ
- 製品化：小型一体型を基本とし、実測で必要な方向だけ短い同軸付きFPCアンテナ化

## Current software direction

- 主催者アプリ群がBLEタグを検出し、位置取得と送信を分散
- 参加端末数の増減に応じて候補確率を適応調整
- 1スロット当たりの期待候補数：概ね2台
- 暫定開催判定：推定有効端末10台以上。判定遅延は許容
- 位置更新目標：30秒
- 非候補端末は位置取得・位置送信を行わない
- 送信前のサーバー状態GETは行わない
- BLEは距離測定ではなく、山笠近傍性と位置レポート品質の評価に使用
- 複数端末のGNSS位置を検証・統合し、初期版ではタグ固定オフセットを主要経路に置かない
- Google Roads APIで近傍道路へ補正
- 道路補正済み位置の時系列から進行方向を推定
- Google Roads API障害時は生位置で継続
- API URI：`/v1/resources/{resourceId}/...`

## Current distribution direction

- アプリは無料配布
- iOS：非表示App Storeを基本とする
- Android：通常のGoogle Play Production公開を基本とする
- 利用権限は期限付き招待URIまたはQRコードで有効化
- 招待認可前にBluetooth・位置権限を要求しない
- バックグラウンドBLE・位置取得をコア機能として審査申告する
