# Infineon CAN アームアクチュエータ部品選定整理

作成日: 2026-06-18  
更新日: 2026-06-23
入力: [requirements.md](./requirements.md)

## 1. 選定方針

- [requirements.md](./requirements.md) を正とし、現時点で決まっている要件を前提に部品を選定する。
- CSV 形式の BOM は [../../bom/arm_actuator_bom.csv](../../bom/arm_actuator_bom.csv) に分ける。
- Infineon 製品のみを統合した BOM は [../../bom/infineon_integrated_bom.csv](../../bom/infineon_integrated_bom.csv) に分ける。
- モータドライバ、パワー MOSFET、CAN、角度センサ、電流センサ、電源監視は Infineon 部品を基本とする。
- STM32 MCU、コネクタ、ヒューズ、受動部品、BLDC モータ、サイクロイド減速機、機械部品は Infineon 縛りの対象外とする。
- 初期試作では RoboMaster M3508 クラスの 24V BLDC モータと 1:120 サイクロイド減速機を基準にする。ただし、軸ごとの連続電流、ピーク電流、保持トルク、放熱条件が未確定のため、高電流部品は暫定選定とする。
- 2026-06-23 に `dktools` で Digi-Key 取扱、Active 状態、在庫の有無を確認した。Digi-Key 発注に使える品番があるものは、短縮名ではなく発注可能な MPN を優先して記載する。
- EOL、Not for new design、用途に対して不適切な候補は部品候補から外し、この文書には残さない。

## 2. 要件から見た構成

```text
24V Arm Power from PDU
  -> input fuse / TVS / reverse protection / bulk capacitor
  -> DC bus voltage sensing
  -> 3-phase inverter
        -> BLDC motor
        -> cycloidal reducer
        -> joint output

STM32 MCU
  -> PWM / ADC / protection input
  -> 3-phase smart gate driver
  -> phase current shunts
  -> DC bus current sensor for protection / logging
  -> output-axis angle sensor
  -> temperature sensors
  -> CAN transceiver
```

初期方針は、1 駆動軸につき 1 枚のアクチュエータ基板を置く構成とする。対象は 3 関節、手首ロール、ベースロール、ベースホライゾン、把持である。同一基板を複数軸に展開し、軸ごとの差分はモータ、減速比、MOSFET 並列数、電流センサレンジ、温度制限、制御パラメータ、Node ID で吸収する。

## 3. 機能別の主選定

### 3.1 MCU

| 項目 | 選定 |
| --- | --- |
| 主候補 | `STM32G474RET6` |
| 数量目安 | 1 個 / 駆動軸 |
| 担当機能 | PWM 生成、ADC 取得、CAN、SPI、角度センサ読み出し、状態遷移、位置/速度/電流制御 |
| 採用理由 | STM32G4 系はモータ制御向けのタイマ、ADC、DSP/FPU、FDCAN を使いやすく、ARES Project で採用実績の多い STM32 系に揃えられる。 |

注意点:

- `STM32G474RET6` は候補型番であり、最終的なピン数、ADC チャンネル数、SPI 数、CAN、デバッグ端子、設定用ピン数に応じて同系列の別パッケージへ変更してよい。
- ST の `STSPIN32G4` のような統合モータコントローラも存在するが、今回の方針は STM32 + Infineon モータドライバを主構成とする。
- PWM と ADC は電流計測に同期させる。三相シャントを使う場合は、サンプリングタイミングと ADC 入力数を最初に確定する。

### 3.2 BLDC ゲートドライバ・三相インバータ

| 項目 | 選定 |
| --- | --- |
| ゲートドライバ | `6EDL7141XUMA1` (`6EDL7141`) |
| ゲートドライバ数量目安 | 1 個 / 駆動軸 |
| パワー MOSFET | `BSC012N06NSATMA1` (`BSC012N06NS`) |
| MOSFET 数量目安 | 6 個。高負荷軸では 12 個を暫定上限として検討 |
| 担当機能 | 三相 BLDC/PMSM インバータ駆動、ゲート駆動設定、保護、fault 通知、SPI 設定 |
| 採用理由 | `6EDL7141` はバッテリー駆動 BLDC/PMSM 向けの 3 相スマートゲートドライバで、SPI による設定と fault 読み出しができる。`BSC012N06NS` は 60V 低 Rds(on) MOSFET で、24V 系 M3508 クラス BLDC インバータに十分な耐圧余裕を持たせやすい。 |

構成方針:

- 初期試作は 6 MOSFET、つまり各スイッチ 1 個の三相インバータから始める。
- 第一関節や肩関節など高負荷軸では、各スイッチ 2 並列の 12 MOSFET 構成、放熱強化、または基板派生を検討する。
- `6EDL7141` の SPI 設定値は、PWM 周波数、ゲート抵抗、MOSFET、シャント抵抗、過電流閾値、デッドタイムを実機評価で決める。
- PWM 方式はセンサ付き FOC を前提にし、初期から三相電流計測と出力軸エンコーダ読み出しを入れる。
- `6EDL7141XUMA1` と `BSC012N06NSATMA1` は、2026-06-23 時点で Active / Digi-Key 在庫ありを確認済み。

注意点:

- `6EDL7141` はゲートドライバであり、制御演算そのものは STM32 側で行う。
- `BSC012N06NS` は 60V 品のため、24V bus、回生、ケーブルインダクタンス、E-Stop 時の過渡電圧を含めて TVS と DC link 容量を設計する。
- MOSFET の数は電流だけでなく、基板銅箔、サーマルビア、ヒートシンク、筐体放熱で決める。

### 3.3 電流計測

| 測定箇所 | 選定 | 数量目安 | 理由 |
| --- | --- | ---: | --- |
| 三相電流 | `TBD_PHASE_SHUNT` | 3 | センサ付き FOC の相電流計測と電流制御に使う |
| 低〜中電流 DC bus | `TLE4971A025N5UE0001XUMA1` (`TLE4971-A025N5-E0001`) | 0 から 1 | 25A 級の独立電流ログ、過電流検出に向く |
| 高電流 DC bus | `TLE4973A075T5S0001XUMA1` (`TLE4973-A075T5-S0001`) | 0 から 1 | 高負荷軸で ±82A レンジの余裕を持たせる |

採用理由:

- センサ付き FOC では PWM と同期した相電流計測が必要になるため、三相シャントを置く。
- TLE4971/TLE4973 系は coreless current sensor として DC bus の独立電流監視、ログ、保護閾値の確認に併用する。
- 低負荷軸は 25A 級、高負荷軸は 75A/82A 級へ分けると、測定分解能とピーク余裕を両立しやすい。
- `TLE4971A025N5UE0001XUMA1` と `TLE4973A075T5S0001XUMA1` は Infineon 公式サイトで確認済み。

注意点:

- 三相シャント値は、最大相電流、許容損失、アンプゲイン、ADC 入力範囲で決める。
- DC bus 電流センサは保護・ログ用として併用する。TLE4971 と TLE4973 のどちらを使うかは軸ごとの電流レンジで決める。
- 保護用途は `6EDL7141` の fault と STM32 の高速保護入力を優先し、ログ用途の平均電流とは分ける。

### 3.4 角度センサ・エンコーダ

| 項目 | 選定 |
| --- | --- |
| 主候補 | `TLE5012BE1000XUMA1` (`TLE5012B-E1000`) |
| 数量目安 | 1 個 / 駆動軸 |
| 担当機能 | 出力軸の絶対角取得、位置制御、速度推定、原点確認、角度異常検出 |
| 採用理由 | `TLE5012B` 系は 360 deg 磁気角度センサで、SPI 互換の SSC インターフェースと診断機能を持つため、関節出力軸の絶対角センサに使いやすい。 |

構成方針:

- 初期試作では関節出力軸側、または減速機出力軸側に絶対角センサを置く。
- モータ始動や低速制御に必要な場合は、BLDC モータ側のホールセンサまたはモータ軸エンコーダを追加する。
- 減速比、バックラッシュ、弾性を制御で扱うため、モータ軸角度と出力軸角度は同じものとして扱わない。

注意点:

- 磁石の芯ずれ、距離、傾き、固定方法が角度精度に直結するため、機械設計側でセンサ座面と磁石保持を決める。
- 角度ジャンプ、通信異常、診断エラーは `FAULT` 条件に含める。
- `TLE5012BE1000XUMA1` は、2026-06-23 時点で Active / Digi-Key 在庫ありを確認済み。

### 3.5 CAN

| 項目 | 選定 |
| --- | --- |
| 主 IC | `TLE9371VSJXTMA1` (`TLE9371VSJ`) |
| 数量目安 | 1 個 / 駆動軸 |
| 担当機能 | CAN 1Mbps 初期対応、将来の CAN FD 対応、上位ノードとの通信 |
| 採用理由 | `TLE9371VSJ` は CAN FD SIC transceiver で、初期の CAN 2.0 / 1Mbps に対して余裕があり、将来 CAN FD へ拡張しやすい。 |

注意点:

- 終端抵抗 120 Ohm は全ノードに固定実装せず、バス端ノードのみ有効化できる構成にする。
- TVS、コモンモードチョーク、GND、シールド、コネクタピン配置はローバー全体の CAN 配線ルールと揃える。
- Node ID は関節位置または駆動軸に対応する固定 ID とし、同一 bus 上で衝突しないようにする。
- テレメトリ周期は初期値 10 ms から 50 ms 程度で始める。
- `TLE9371VSJXTMA1` は、2026-06-23 時点で Active / Digi-Key 在庫ありを確認済み。

### 3.6 常時電源・監視電源

| 項目 | 選定 |
| --- | --- |
| 主候補 | `TLF35584QKVS2XUMA2` (`TLF35584QKVS2`) |
| 数量目安 | 1 個 / 駆動軸 |
| 担当機能 | STM32 用 3.3V、CAN/センサ用 5V または 3.3V、ウォッチドッグ、リセット監視 |
| 採用理由 | `TLF35584QKVS2` は安全系用途向けの複数出力 PMIC で、MCU 電源、トランシーバ/センサ電源、ウォッチドッグ、リセット監視をまとめやすい。 |

注意点:

- `6EDL7141` にもレギュレータ機能があるため、初期試作では「6EDL7141 周辺電源を活用する簡略構成」と「TLF35584 を使う監視重視構成」を比較する。
- TLF35584 は高機能なぶん、SPI 設定や起動シーケンスが増える。小型・低コストを優先する関節では、別の 3.3V/5V レギュレータ + 外付けウォッチドッグの構成も検討余地がある。
- PDU 側のアーム 24V が OFF になった場合の MCU ログ保持や fault 送信をどう扱うかは、上位システム設計で決める。
- `TLF35584QKVS2XUMA2` は、2026-06-23 時点で Active / Digi-Key 在庫ありを確認済み。

### 3.7 電圧・温度監視

| 測定箇所 | 選定 | 数量目安 | 理由 |
| --- | --- | ---: | --- |
| DC bus 電圧 | `TBD_BUS_VSENSE` | 1 set | 低電圧、過電圧、回生時の過渡電圧を監視する |
| MOSFET 近傍温度 | `TBD_NTC_FET` | 1 | インバータの熱制限と fault 判定に使う |
| モータケース温度 | `TBD_NTC_MOTOR` | 1 | モータケースの熱保護に使う |

設計メモ:

- DC bus 電圧は STM32 ADC で読む前提とし、分圧抵抗、RC フィルタ、ADC 保護を入れる。
- MOSFET 温度はホットスポットに近い位置へ NTC を置き、実測でジャンクション温度との相関を取る。
- モータ温度はモータケース NTC を基本とし、モータ候補に巻線温度センサがある場合は追加で使う。

### 3.8 入力保護・DC link・コネクタ

| 項目 | 選定 |
| --- | --- |
| 入力ヒューズ | `TBD_ACTUATOR_FUSE` |
| 入力保護 | `TBD_ACTUATOR_INPUT_PROTECTION` |
| DC link capacitor | `TBD_DC_LINK_CAP_SET` |
| 電源コネクタ | `TBD_ACTUATOR_POWER_CONNECTOR` |
| モータコネクタ | `TBD_MOTOR_CONNECTOR` |
| CAN コネクタ | `TBD_CAN_CONNECTOR` |

設計メモ:

- PDU 側のアーム出力保護と、各関節基板側の局所保護の役割分担を決める。
- ケーブルが長い場合、E-Stop、回生、負荷急変時の DC bus 過渡電圧が大きくなるため、TVS と bulk capacitor を各関節に置く。
- モータ相線、CAN、センサ、電源はコネクタを分け、配線ミスとノイズ混入を減らす。
- CAN をデイジーチェーンする場合は、CAN コネクタを 2 個置く構成を検討する。

### 3.9 モータ・サイクロイド減速機

| 項目 | 選定 |
| --- | --- |
| BLDC モータ | `TBD_M3508_CLASS_BLDC_MOTOR` |
| サイクロイド減速機 | `TBD_1_120_CYCLOID_REDUCER` |
| 数量目安 | 各 1 個 / 駆動軸 |
| 担当機能 | 関節トルク、速度、保持、外力耐性 |

選定方針:

- モータは RoboMaster M3508 と同程度の 24V BLDC モータを基準とし、連続電流、ピーク電流、Kv、極対数、センサ有無、取付性、入手性で最終型番を選ぶ。
- 第一関節のサイクロイド減速機は 1:120 を基準とし、出力定格トルク約 300N*m、最大出力トルク約 600N*m、出力回転数約 3.9rpm 相当を参考にする。
- 他軸のサイクロイド減速機は、必要出力トルク、許容ピークトルク、効率、バックラッシュ、逆駆動性、サイズ、重量で選ぶ。
- 制御側の部品選定は、最終的なモータ電流と減速比に強く依存するため、先に肩・肘・手首・把持・ベースホライゾンなど軸ごとのトルク表を作る。

## 4. 暫定 BOM

| ブロック | 品番 | 数量目安 | メモ |
| --- | --- | ---: | --- |
| MCU | `STM32G474RET6` | 1 | 型番は IO 数に応じて変更可 |
| 3-phase Gate Driver | `6EDL7141XUMA1` | 1 | Active / Digi-Key 在庫あり。SPI 設定/fault 読み出し |
| Power MOSFET | `BSC012N06NSATMA1` | 6 から 12 | Active / Digi-Key 在庫あり。6 個で標準、12 個で高電流派生 |
| Phase Current Sense | `TBD_PHASE_SHUNT` | 3 | センサ付き FOC 用 |
| DC Bus Current Sensor | `TLE4971A025N5UE0001XUMA1` | 0 から 1 | Active / Digi-Key 在庫あり。低〜中電流軸の保護・ログ用 |
| High Current Sensor | `TLE4973A075T5S0001XUMA1` | 0 から 1 | Infineon 公式確認済み。高負荷軸の保護・ログ用 |
| Angle Sensor | `TLE5012BE1000XUMA1` | 1 | Active / Digi-Key 在庫あり。出力軸絶対角を優先 |
| CAN | `TLE9371VSJXTMA1` | 1 | Active / Digi-Key 在庫あり。CAN 1Mbps、将来 CAN FD |
| Housekeeping | `TLF35584QKVS2XUMA2` | 1 | Active / Digi-Key 在庫あり。監視重視構成。簡略構成では再検討可 |
| FET Temperature | `TBD_NTC_FET` | 1 | MOSFET 近傍 |
| Motor Temperature | `TBD_NTC_MOTOR` | 1 | モータケースまたは巻線 |
| Voltage Sense | `TBD_BUS_VSENSE` | 1 set | 24V bus ADC 測定 |
| Input Protection | `TBD_ACTUATOR_INPUT_PROTECTION` | 1 set | fuse、TVS、逆接保護、bulk capacitor |
| Connectors | `TBD_*_CONNECTOR` | 1 set | power、motor、CAN、SWD |
| BLDC Motor | `TBD_M3508_CLASS_BLDC_MOTOR` | 1 | M3508 クラスを基準に軸ごとに選定 |
| Cycloidal Reducer | `TBD_1_120_CYCLOID_REDUCER` | 1 | 第一関節は 1:120 基準。他軸は軸ごとに選定 |

## 5. 未確定要件に依存する選定ポイント

- 各軸の連続電流、10 秒ピーク、100 ms ピークが決まるまで、`BSC012N06NS` の並列数、シャント抵抗値、電流センサレンジ、コネクタ定格、ヒューズ定格は確定できない。
- 低速・停止付近の制御性能を重視するため、センサ付き FOC と出力軸絶対角センサを前提にする。
- サイクロイド減速機のバックラッシュと弾性が大きい場合、モータ軸センサだけでは関節角度を正確に制御しにくい。出力軸側 `TLE5012B` を優先する。
- E-Stop または PDU 側アーム電源 OFF 時に関節が落下する可能性がある場合、基板部品だけでは解決できない。ブレーキ、機械ストッパ、自己保持性、復帰シーケンスを別途決める。
- CAN bus のノード数と配線長が増える場合、終端、分岐長、シールド、GND 接続、コモンモードチョーク、TVS の仕様をローバー全体で揃える。
- `TLF35584QKVS2` を使うか、より単純なレギュレータ + 外付けウォッチドッグにするかは、基板面積、コスト、安全要求、起動シーケンスの複雑さで判断する。

## 6. 公式参考資料

- STM32G474RE: <https://www.st.com/en/microcontrollers-microprocessors/stm32g474re.html>
- STM32G4 Series: <https://www.st.com/en/microcontrollers-microprocessors/stm32g4-series.html>
- 6EDL7141XUMA1 / 6EDL7141: <https://www.infineon.com/part/6EDL7141>
- EVAL_6EDL7141_1KW_36V: <https://www.infineon.com/evaluation-board/EVAL-6EDL7141-1KW-36V>
- EVAL_6EDL7141_FOC_3SH: <https://www.infineon.com/evaluation-board/EVAL-6EDL7141-FOC-3SH>
- BSC012N06NSATMA1 / BSC012N06NS: <https://www.infineon.com/part/BSC012N06NS>
- TLE5012BE1000XUMA1 / TLE5012B-E1000: <https://www.infineon.com/part/TLE5012B-E1000>
- TLE9371VSJXTMA1 / TLE9371VSJ: <https://www.infineon.com/part/TLE9371VSJ>
- TLF35584QKVS2XUMA2 / TLF35584QKVS2: <https://www.infineon.com/part/TLF35584QKVS2>
- TLE4971A025N5UE0001XUMA1 / TLE4971-A025N5-E0001: <https://www.infineon.com/part/TLE4971-A025N5-E0001>
- TLE4973A075T5S0001XUMA1 / TLE4973-A075T5-S0001: <https://www.infineon.com/part/TLE4973-A075T5-S0001>
