# PIX-DT295 受信停止の調査と修正候補

PIX-DT295×2とpix-userland、EPGStation、KonomiTVを組み合わせた環境で行った調査・検証の記録です。検証日：2026年11月8日〜9日。

## 報告とパッチ

| 対象 | 内容 | 資料 |
|---|---|---|
| pix-userland | 約10〜11MBでの受信停止を調査。AES復号処理の改善、転送診断、セッション更新失敗の通知、回帰テスト | [報告](pix-userland/REPORT.md)／[パッチ](pix-userland/fix.patch) |
| KonomiTV：擬似全画面 | 字幕・コメントを維持してプレイヤー全体を拡大。利用者によるiPhone 17 Pro Max実機確認済み | [報告](konomitv/PSEUDO-FULLSCREEN.md)／[パッチ](konomitv/pseudo-fullscreen.patch)／[一括パッチ](konomitv/combined-fixes.patch) |
| KonomiTV | 初期データ待ちが10秒を超えると接続が削除される問題を修正。入力待機と出力先の停滞を区別 | [報告](konomitv/REPORT.md)／[パッチ](konomitv/fix.patch) |

両作者向けの報告文とパッチをまとめたZIPは、このリポジトリのReleasesからダウンロードできます。

## 確認できたこと

- 同じチューナー・同じpixdで、30秒以上の映像を含む受信・正常停止・再利用を3回連続で確認。
- EPGStationで約122秒録画しながら、別物理チャンネルをKonomiTVの720pで約61秒配信。
- KonomiTVの初回HTTPS接続からデータ到着まで10.572秒。接続1回、再接続0回。
- チャンネル設定とチューナー2台構成は維持。

## 検証範囲

個別環境で確認した修正候補です。長期連続運用、SIGKILLからの復旧、BS/CS、ブラウザ画面・実スピーカーでの確認は未実施です。元の停止コマンド失敗とファームウェア異常の直接原因には未確定の点があります。メディア端部のデコード警告も残っており、詳細は各報告に記載しています。

パッチの適用基準はpix-userlandの`42364ad24f4fbcac2a611e0992d6f08a1cbd360f`、KonomiTVのv0.14.1です。各プロジェクトへの上流採用は未確認です。

## iPhone向け擬似全画面

字幕・流れるコメントを残した擬似全画面の修正候補を公開しています。利用者によるiPhone 17 Pro Max／iOS 27.2実機確認済み（切り替え・字幕とコメントの維持・解除）。[報告と適用方法](konomitv/PSEUDO-FULLSCREEN.md)を参照してください。擬似全画面の追加ファイルはリポジトリ上で配布し、既存ReleaseのZIPは公開時の内容を保持しています。
