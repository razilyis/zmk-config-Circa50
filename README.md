# Circa50 — ZMK config

分割キーボード Circa50 用の [ZMK](https://zmk.dev/) ファームウェア設定です。

## 仕様

- コントローラー: Seeed XIAO nRF52840 **Plus** × 2（ビルドターゲット `xiao_ble//zmk`）
- キー数: 47（左24キー / 右23キー）、17mmピッチ
- 左右分割、無線（Bluetooth）接続
  - **右側がセントラル**です。PCには右側をUSBまたはBluetoothで接続します。
  - 左側はペリフェラルで、右側と接続します。
- トラックボール: 右側に PAW3222 センサー（760 CPI）
- ディープスリープ: 15分無操作で移行。キーを押すと復帰します。

## ファームウェアの入手

1. このリポジトリを Fork します。
2. GitHub の **Actions** タブで `Build Circa50` を実行します（push でも自動実行されます）。
3. 完了後、Artifacts の `firmware` をダウンロードして展開します。

| ファイル | 用途 |
|---|---|
| `Circa50-left.uf2` | 左側用 |
| `Circa50-right.uf2` | 右側用 |
| `Circa50-settings-reset.uf2` | ペアリング情報などの設定リセット用 |
| `Circa50-left-usb-logging.uf2` / `Circa50-right-usb-logging.uf2` | 不具合調査用のログ出力版 |

## 書き込み

1. XIAO をUSBでPCに接続し、リセットボタンを素早く2回押します。
2. `XIAO-SENSE` などの名前のドライブが表示されます。
3. 左側には `Circa50-left.uf2`、右側には `Circa50-right.uf2` をコピーします。

左右は必ず同じバージョンのファームウェアを書き込んでください。

### 左右がつながらない / ペアリングをやり直したいとき

1. `Circa50-settings-reset.uf2` を**左右両方**に書き込みます。
2. それぞれに通常のファームウェアを書き込み直します。
3. PC側に古い `Circa50` の登録があれば削除し、再度ペアリングします。

設定リセットを行うと、保存されているペアリング情報はすべて消去されます。

## キーマップ

キーマップは [`config/circa50.keymap`](config/circa50.keymap) で定義しています。
OSのキーボード配列はUS配列を想定しています。

### Keymap Editor

[ZMK Keymap Editor](https://nickcoutsos.github.io/keymap-editor/) で編集できます。
[`config/circa50.json`](config/circa50.json) に実機と同じキー配置のレイアウトを定義しているため、
エディター上でも実機どおりの並びで表示されます。

### オートマウスレイヤー

トラックボールを動かすと、クリック用のマウスレイヤーに自動で切り替わります。

- ボールの動きが止まってから 500ms 後に元のレイヤーへ戻ります。
- マウスレイヤーでは J / K / L の位置が左クリック / 中クリック / 右クリックになります。
- クリック以外のキーを押すと、その時点でマウスレイヤーを抜けます。
- タイピング直後（150ms以内）はボールに触れても切り替わりません。

戻るまでの時間は [`circa50_right.overlay`](config/boards/shields/circa50/circa50_right.overlay) の
`input-processors = <&zip_temp_layer 4 500>;` の `500`（ミリ秒）で調整できます。
`4` はマウスレイヤーのレイヤー番号です。キーマップのレイヤー順を変えた場合は合わせて変更してください。

### スクロール

レイヤー1が有効な間（`/` または左親指の `Space` を押している間）は、
トラックボールの操作が縦・横スクロールになります。

- ボールを奥へ転がすと上へスクロールします（マウスホイールと同じ向き）。
- スクロール速度は同じ overlay の `<&zip_scroll_scaler 1 16>` の `16` で調整できます
  （大きくすると遅く、小さくすると速くなります）。

## ローカルでビルドする

ZMK のビルド環境（[公式ドキュメント](https://zmk.dev/docs/development/local-toolchain/setup)）がある場合は、
このリポジトリ直下で次を実行します。

```sh
west init -l config
west update
west zephyr-export
west build -s zmk/app -d build/left -b xiao_ble//zmk -- -DZMK_CONFIG="$PWD/config" -DSHIELD=circa50_left
west build -s zmk/app -d build/right -b xiao_ble//zmk -- -DZMK_CONFIG="$PWD/config" -DSHIELD=circa50_right
```

## USBログ版（不具合調査用）

`*-usb-logging.uf2` は、USBシリアル経由で動作ログを出力する診断用ファームウェアです。
キー入力・左右間通信・トラックボールの状態を確認できます。
ログは書き込んだ側のUSB接続から取得します。トラックボールを調べる場合は右側に書き込んでください。

1. 調べたい側にログ版を書き込み、データ通信対応のUSBケーブルでPCに接続します。
2. シリアルポートをシリアル端末で開きます。
   - Windows: デバイスマネージャーの「ポート（COMとLPT）」でCOM番号を確認し、PuTTY / Tera Term などで開きます。
   - macOS / Linux: `/dev/cu.usbmodem*` / `/dev/ttyACM*` を `screen` などで開きます。
3. ログは起動から約8秒後に出力が始まります。問題の操作を再現してログを保存します。

トラックボールに関しては、`paw32xx` の `Invalid product id` などのエラーや、
ボールを動かしたときの `x=... y=...` が確認ポイントです。

> [!CAUTION]
> ログにはキー入力の情報が含まれます。取得中にパスワードなどを入力しないでください。
> また、ログを共有する前に内容を確認してください。

ログ版はスリープが無効で消費電力が大きいため、調査が終わったら通常版に戻してください
（settings-reset の書き込みは不要です）。

参考: [ZMK USB Logging](https://zmk.dev/docs/development/usb-logging)

## ハードウェア情報

### 配線（nRF GPIO）

| 信号 | 左 | 右 |
|---|---|---|
| Col0 | P0.02 | P0.02 |
| Col1 | P0.03 | P0.03 |
| Col2 | P0.28 | P0.28 |
| Col3 | P0.29 | P0.29 |
| Col4 | P0.15 | P0.04 |
| Col5 | P0.19 | P0.05 |
| Row0 | P1.01 | P1.11 |
| Row1 | P1.07 | P1.12 |
| Row2 | P1.05 | P1.13 |
| Row3 | P1.03 | P1.14 |

- ダイオード方向は `col2row` です。
- 左側は XIAO nRF52840 Plus の追加端子を使用するため、通常版の XIAO では動作しません。

### トラックボール（右側 FFC コネクタ）

| ピン | 信号 | GPIO |
|---|---|---|
| 1 | GND | — |
| 2 | MOTION | P1.15 |
| 3 | SDIO | P1.07 |
| 4 | CS | P1.05 |
| 5 | SCLK | P1.03 |
| 6 | 3.3V | — |

センサー基板とはピン番号どうしが1対1になるよう接続してください。5Vは接続しないでください。
センサーの軸方向が合わない場合は、右側 overlay の設定で調整してください。

## 設定の検証

設定ファイルどうしの整合性（キー数、配列、GPIO割り当てなど）を確認するスクリプトです。
Python 標準ライブラリのみで動作します。

```sh
python scripts/validate.py
```

## 使用しているソフトウェア

- [ZMK Firmware](https://github.com/zmkfirmware/zmk)
- [zmk-driver-paw3222](https://github.com/sekigon-gonnoc/zmk-driver-paw3222)

バージョンは [`config/west.yml`](config/west.yml) で固定しています。

## ライセンス

[MIT License](LICENSE)
