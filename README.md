# yamakasa-tracker

山笠に搭載したBLEタグを周囲の主催者スマートフォンが検出し、端末群で位置取得・送信を分担して、観客向けWebへ山笠の現在位置を公開するシステム。

タグ自体はGPSやLTEを持たず、BLE広告だけを送信する。山笠1基には前・後・左・右の4タグを配置し、巨大な構造物、人形の開閉、板・布、全面のLED・配線、周囲の人体による方向別遮蔽を相互補完する。

## Documentation

- [ハードウェア要件](docs/hardware-requirements.md)
- [ハードウェア設計](docs/hardware-design.md)
- [タグ配置設計](docs/tag-placement.md)

## Current hardware direction

- BLEタグ：4個／山笠
- 配置：前・後・左・右の外向き配置
- 電源：タグごとにCR2032
- 動作期間：最低1週間
- 防水：IP67相当
- BLE広告：固定公開山笠ID＋方向別タグID＋認証コード
- GPS、LTE、加速度センサー：タグには搭載しない
