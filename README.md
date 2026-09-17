# MIDIoparator-to-esp32
# MIDI2026 — MIDIコントローラー駆動 BLEメカナムローバー

DJ用MIDIコントローラー（DDJ-FLX4）の入力をPCブリッジ経由でBLEに変換し、ESP32上のメカナムローバー＋4軸サーボアームをワイヤレス操作するプロジェクトです。

```
MIDIコントローラー(DDJ-FLX4)
      │ USB-MIDI
      ▼
  midi_2026.py（PC）
      │ BLE (Nordic UART Service互換)
      ▼
  ESP32 (mechanum_MIDI.ino)
      │ PWM
      ▼
  モーター4輪 + サーボ4軸
```

## ディレクトリ構成

```
MIDI2026/
├── midi_2026.py                 # MIDI→BLE ブリッジ本体（PCで実行）
├── BLE_Checker.py                # 周辺のBLEデバイスをスキャンするツール
├── .venv/                        # Python仮想環境（Windows用、成果物ではない）
└── BLE_midi_to_esp32/
    ├── midi_mechanum/
    │   └── midi_mechanum.ino     # 旧版（4輪メカナムのみ、未完成・要修正）
    └── 最新版はこっち！/
        └── mechanum_MIDI.ino     # 最新版（4輪メカナム＋補助モーター3系統＋サーボ4軸）
```

> `.venv` はPythonの仮想環境で、配布物ではありません。リポジトリに含める場合は `.gitignore` に追加することを推奨します。

## 動作の流れ

1. **`midi_2026.py`をPCで実行** — `rtmidi2`でUSB接続されたMIDIコントローラー（既定では`DDJ-FLX4`）からMIDIメッセージを受信
2. 受信したメッセージはキー（status, data1）ごとにバッファに集約され、8msごとにまとめてBLE経由でESP32のRXキャラクタリスティックへ送信（`response=False`のWrite Without Response）
3. **ESP32（`mechanum_MIDI.ino`）**がBLEで受信し、status/data1/data2をMIDIメッセージとして解釈し、モーターPWMおよびサーボ角度を更新
4. 4輪メカナムローバー＋補助モーター3系統＋サーボアーム4軸が動作

## 各ファイルの役割

### `midi_2026.py`（PC側・MIDI→BLEブリッジ）

- 使用ライブラリ: [`bleak`](https://github.com/hbldh/bleak)（BLE）、[`rtmidi2`](https://pypi.org/project/rtmidi2/)（MIDI入力）
- 設定項目:
  - `MIDI_DEVICE_NAME`: 接続したいMIDIコントローラー名（部分一致、既定`"DDJ-FLX4"`）
  - `MAC_ADDRESS`: 接続先ESP32のBLE MACアドレス（`BLE_Checker.py`の結果から取得して設定）
  - `SERVICE_UUID` / `CHAR_UUID_RX` / `CHAR_UUID_TX`: ESP32側と共通のNordic UART Service互換UUID
- 動作:
  - MIDI受信コールバック（`on_messages`）で `(status, data1) → data2` の形でバッファに保存（同じキーは上書きされ、最新値のみ送信される＝間引き）
  - `send_loop()`が8ms周期でバッファをスナップショットしてクリアし、3バイト×メッセージ数のパケットとしてBLE送信
  - ESP32からの通知（TXキャラクタリスティック）はコンソールにデコード出力するのみ

### `BLE_Checker.py`（PC側・BLEスキャンツール）

- `bleak.BleakScanner`で周辺のBLEデバイスを5秒間スキャンし、RSSI降順で名前・アドレス・RSSI・広告UUIDを一覧表示
- ESP32の実際のMACアドレスを調べて`midi_2026.py`の`MAC_ADDRESS`に設定するために使用

### `BLE_midi_to_esp32/最新版はこっち！/mechanum_MIDI.ino`（ESP32・最新版ファームウェア）

4輪メカナムホイール駆動＋補助モーター2系統＋機構モーター1系統＋4軸サーボアームを制御する完成版です。詳細な配線・BLEコマンド仕様は以下の別紙にまとめています。

- **BLE UUID**: Service `6e400001-...`、RX（Write） `6e400002-...`、TX（Notify） `6e400003-...`（`midi_2026.py`と共通）
- **移動制御**: `updateMecanum()`がstickX/Y（MIDI値63を中心とした±方向）から4輪の目標速度を合成
- **補助モーター**: MIDI Note On（status 153）でDX/FB/MKの3系統をON/OFF制御
- **サーボアーム**: 土台旋回（Dodai）・アーム関節2軸（Arm1/Arm2）・ハンド（Hand）をMIDI CC値から角度制御
- **既知の問題**: `DODAI_PIN`(12)・`ARM1_PIN`(13)がモーターMKのPWM出力ピンと重複しているため、両方を同時使用する場合は配線・ピン番号の見直しが必要

### `BLE_midi_to_esp32/midi_mechanum/midi_mechanum.ino`（旧版・未完成）

4輪メカナムのみを制御する古いバージョンです。

> ⚠️ **このファイルはコンパイルできません。** `_DODAI`・`_Arm1`・`_Arm2`・`_HAND`が`Servo`型ではなく`#define ... 16`という整数マクロとして定義されているため（8〜11行目）、`_DODAI.write(angle)`のような呼び出しはビルドエラーになります。`setup()`内の`_DODAI.attach(Dodai)`も同様に、`Dodai`という未定義の識別子を渡しており不整合です。実運用には最新版（`mechanum_MIDI.ino`）を使用してください。

## MIDIコマンド対応表（共通仕様）

送受信されるBLEパケットは3バイト単位（MIDIメッセージのstatus/data1/data2）です。

| status | data1 | 意味 | 動作 |
|---|---|---|---|
| 177 | 0 | 前後スティック | `stickY`更新→メカナム速度再計算 |
| 176 | 0 | 左右スティック | `stickX`更新→メカナム速度再計算 |
| 176 | 34 | 右旋回ボタン | その場右旋回 |
| 177 | 34 | 左旋回ボタン | その場左旋回 |
| 153 | 0 / 4 | FBモーター前進/後退 | ON/OFF |
| 153 | 1 / 5 | DXモーター後退/前進 | ON/OFF |
| 153 | 2 / 6 | MKモーター前進/後退 | ON/OFF |
| 182 | 24 | 土台（Dodai）角度 | サーボ角度セット |
| 177 | 15 | アーム1角度 | サーボ角度セット |
| 177 | 11 | アーム2角度 | サーボ角度セット |
| 177 | 19 | ハンド角度 | サーボ角度セット |
| 148 | 99 | サーボ一括リセット | 全サーボ90° |
| 148 | 71 | 緊急停止 | 全モーター速度0（サーボは対象外） |

これらのstatus/data1の値はDDJ-FLX4のパッド・ノブ・フェーダー割り当てに依存するため、他のMIDIコントローラーを使う場合は実機のMIDI出力をログして値を確認・調整してください。

## セットアップ手順

1. **ESP32側**: Arduino IDE/PlatformIOで`mechanum_MIDI.ino`を書き込み（`NimBLE-Arduino`または標準`ESP32 BLE Arduino`、`ESP32Servo`ライブラリが必要）。書き込み後、シリアルモニタでBLEアドレスを確認
2. **PC側の準備**:
   ```
   pip install bleak rtmidi2
   ```
3. `BLE_Checker.py`を実行してESP32のMACアドレスを確認
4. `midi_2026.py`内の`MAC_ADDRESS`と`MIDI_DEVICE_NAME`を実機に合わせて編集
5. MIDIコントローラーをPCにUSB接続した状態で`midi_2026.py`を実行
6. コントローラーを操作するとBLE経由でロボットが動作

## 既知の課題・改善候補

- `mechanum_MIDI.ino`: GPIO12/13がサーボとモーターMKのPWM出力で重複（配線見直し要）
- `mechanum_MIDI.ino`: 緊急停止(148,71)がサーボには効かない
- `midi_mechanum.ino`（旧版）: コンパイル不可、削除または修正が必要
- `midi_2026.py`: MACアドレス・デバイス名がソース内にハードコードされており、設定ファイル化の余地あり
- `midi_2026.py`: バッファは同一(status, data1)キーで上書きされるため、連続する同種イベント（同じノブの複数回の変化など）は間引かれる仕様（意図した設計か要確認）
