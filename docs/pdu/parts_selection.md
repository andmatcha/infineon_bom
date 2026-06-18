# Infineon PDU 部品選定整理

作成日: 2026-06-18  
入力: [requirements.md](./requirements.md)

## 1. 選定方針

- [requirements.md](./requirements.md) を正とし、解決済みの要件を前提に部品を再選定する。
- CSV 形式の BOM は [../../bom/pdu_bom.csv](../../bom/pdu_bom.csv) に分ける。
- 電源保護、スイッチ、CAN、センサは Infineon を基本とする。
- STM32 MCU、XBee、コネクタ、ヒューズ、受動部品は Infineon 縛りの対象外とする。
- EOL、Not for new design、用途に対して不適切な候補は部品候補から外し、この文書には残さない。
- できるだけ現行かつ新しい製品を優先する。ただし、電流、放熱、SOA、PCB 銅箔、コネクタ定格が未確定の高電流部は暫定選定とする。

## 2. 要件から見た構成

```text
Makita 18V Battery A
  -> hot-swap / input protection / telemetry
Makita 18V Battery B
  -> hot-swap / input protection / telemetry
        -> protected input bus
        -> Foot 24V DC/DC, 24V +/-5%
              -> Foot 24V bus
              -> high-side switch -> Foot drive, provisional 40A class
        -> Arm 24V DC/DC, 24V +/-5%
              -> Arm 24V bus
              -> high-side switch -> Arm, current TBD for multi-axis actuator load

Always-on rail
  -> STM32 MCU
  -> XBee
  -> CAN transceiver
  -> telemetry / watchdog / E-Stop monitoring
```

## 3. 機能別の主選定

### 3.1 バッテリー入力・ホットスワップ

| 項目 | 選定 |
| --- | --- |
| 主 IC | `XDP711-001` |
| 数量目安 | 2 個、Battery A/B に各 1 個 |
| 外付け FET | `ISC035N10NM5LF2` |
| FET 数量目安 | 4 から 8 個。back-to-back 構成と並列数で変動 |
| 担当機能 | ホットスワップ、突入電流制限、入力保護、PMBus テレメトリ、FET SOA 保護、下流系の Disable |
| 採用理由 | `XDP711-001` は `+7V` から `+80V` のホットスワップ/監視コントローラで、現行世代の新しい候補。`ISC035N10NM5LF2` は hot swap、battery protection、e-fuse 用途向けの 100V Linear FET で、リニア領域の SOA を重視する入力保護に向く。 |

設計メモ:

- 2 本のマキタ 18V バッテリーは独立冗長入力として扱う。
- 逆流防止は FET の向きと back-to-back 構成で決める。
- E-Stop のハードウェア停止経路では、下流スイッチだけでなく `XDP711-001` の Enable/Disable も遮断候補に含める。
- XDP711 の PMBus 設定値、FET SOA プロファイル、短絡遮断条件は、実負荷電流と入力容量が決まった後に確定する。

### 3.2 24V DC/DC 生成

| 項目 | 選定 |
| --- | --- |
| デジタル電源コントローラ | `XDPP1188-200C` |
| 数量目安 | 2 個。足回り用 DC/DC とアーム用 DC/DC に各 1 個 |
| ゲートドライバ | `1EDN7116G` |
| パワー MOSFET | `ISC022N10NM6` |
| 担当機能 | 18V 系入力から足回り用 24V ±5% とアーム用 24V ±5% の 2 系統生成、電圧/電流制御、故障監視、STM32 へのテレメトリ連携 |
| 採用理由 | `XDPP1188-200C` は新しい XDP デジタル電源コントローラで、8 PWM、PMBus/I2C/UART/SPI を備え、isolated/non-isolated の高密度 DC/DC に使える。`1EDN7116G` と `ISC022N10NM6` は 100V half-bridge 評価構成でも使われており、18V/24V 系より十分高い耐圧余裕を持たせやすい。 |

構成方針:

- 足回りとアームのノイズ分離、負荷変動、片系統故障時の継続運用を優先し、24V DC/DC は 2 系統に分ける。
- 足回り用 DC/DC とアーム用 DC/DC は、Enable、fault/warn、電圧・電流監視を個別に扱う。
- 片方の DC/DC または出力系統が故障した場合、入力電源、E-Stop、MCU、通信、共通保護に異常がなければ、他方の 24V 系統を継続運用できる構成にする。
- DC/DC の詳細トポロジは、各系統の入力電圧範囲、必要電流、効率目標、放熱条件から決定する。

### 3.3 足回り・アーム 24V ON/OFF

| 項目 | 足回り 24V | アーム 24V |
| --- | --- | --- |
| 主選定 | `BTH50015-1LUA` | `BTH50030-1LUA` を低電流暫定候補とし、総電流確定後に再選定 |
| 数量目安 | 1 個 | 1 個 |
| 想定電流 | 暫定 40A 級。ただし単品公称は 35A 級のため熱設計と実電流で再確認 | ARES X の複数アクチュエータ軸の総電流で再見積もり |
| 担当機能 | 高電流ハイサイド ON/OFF、過電流保護、短絡保護、温度保護、診断、電流センス | 高電流ハイサイド ON/OFF、過電流保護、短絡保護、温度保護、診断、電流センス |

採用理由:

- `BTH50015-1LUA` と `BTH50030-1LUA` は Power PROFET+ 24/48V 系の新しいスマートハイサイドスイッチで、24V/48V power net 向けに設計されている。
- 両部品とも ISO 26262-ready、診断機能付きで、ローバー PDU の出力遮断と fault 検出に向く。
- 足回り 40A 級は最終電流が未確定のため、`BTH50015-1LUA` 単品で足りるか、放熱強化・並列・ディスクリート構成が必要かを最終設計で判断する。
- アーム出力は 5 軸、ベースホライゾン、把持を含む複数の M3508 クラスアクチュエータへ供給するため、`BTH50030-1LUA` は低電流暫定候補として扱い、総電流確定後に `BTH50015-1LUA`、並列構成、または XDP711 + 外付け MOSFET による eFuse 構成を再検討する。

注意点:

- PROFET は負荷スイッチであり、モータ PWM 駆動用ではない。下流のモータ制御 PWM は各モータドライバ側で行う。
- 大容量負荷を直投入する場合は、XDP711 側のソフトスタートや DC/DC 側の立ち上げシーケンスと組み合わせる。
- E-Stop では PROFET 入力を落とし、同時に DC/DC 停止または XDP711 停止も組み合わせる。

### 3.4 常時電源・監視電源

| 項目 | 選定 |
| --- | --- |
| 主 IC | `TLF35584QKVS2` |
| 小型パッケージ候補 | `TLF35584QVVS2` |
| 数量目安 | 1 個 |
| 担当機能 | STM32 用 3.3V、通信/センサ用 5V、ウォッチドッグ、リセット監視、安全状態制御 |
| 採用理由 | `TLF35584QKVS2` / `TLF35584QVVS2` は 3.3V マイコン、トランシーバ、センサ向けの複数出力 PMIC で、常時電源と監視機能を 1 IC にまとめやすい。 |

注意点:

- XBee の消費電流が大きい場合、XBee は別 3.3V レギュレータに分け、PMIC は STM32 と監視系を中心に供給する。
- STM32 の具体型番と周辺電流が決まった後、3.3V/5V の各レール電流を再確認する。

### 3.5 CAN

| 項目 | 選定 |
| --- | --- |
| 主 IC | `TLE9371VSJ` |
| 数量目安 | 1 個 |
| 担当機能 | CAN 1Mbps 初期対応、将来の CAN FD 対応、Power Board と Rover MCU / Arm MCU の接続 |
| 採用理由 | `TLE9371VSJ` は active and preferred の CAN FD SIC transceiver で、最大 8Mbit/s、3.3V/5V supply、EMC 改善を狙える。初期 1Mbps CAN に対して十分な余裕があり、将来の CAN FD 化にもつなげやすい。 |

注意点:

- 終端抵抗 120 Ohm、TVS、コモンモードチョーク、コネクタピン配置、GND/シールド接続は基板仕様で決める。
- STM32 側の CAN peripheral が CAN FD 非対応の場合でも、1Mbps CAN transceiver として利用する。

### 3.6 電流センサ

| 測定箇所 | 選定 | 数量目安 | 理由 |
| --- | --- | ---: | --- |
| Battery A/B 入力 | `TLE4973-A075T5-S0001` | 0 から 2 | ±82A 範囲で入力電流ピークに余裕を持たせる |
| 足回り 24V | `TLE4973-A075T5-S0001` | 0 から 1 | 暫定 40A 級でピーク未確定のため余裕を持たせる |
| アーム 24V | `TLE4973-A025T5-S0001` または `TLE4973-A075T5-S0001` | 0 から 1 | 総電流が低ければ ±27A、複数軸同時駆動で大きい場合は ±82A を検討 |

採用理由:

- TLE4973 系は XENSIV の新しい coreless current sensor で、5V 駆動、内蔵 current rail、過電流検出、診断インターフェースを持つ。
- 保護用途は XDP711/PROFET の内蔵診断と高速 fault を優先し、ログ精度や独立電流測定が必要な箇所に TLE4973 を追加する。

## 4. 暫定 BOM

| ブロック | 品番 | 数量目安 | メモ |
| --- | --- | ---: | --- |
| Battery Input | `XDP711-001` | 2 | Battery A/B に各 1 |
| Battery Input FET | `ISC035N10NM5LF2` | 4 から 8 | back-to-back と並列数で変動 |
| 24V Digital Power | `XDPP1188-200C` | 2 | Foot/Arm DC/DC に各 1 |
| 24V Gate Driver | `1EDN7116G` | 4 から 8 | 2 系統分。DC/DC トポロジとスイッチ数で変動 |
| 24V Power MOSFET | `ISC022N10NM6` | 8 から 32 | 2 系統分。相数、並列数で変動 |
| Foot Load Switch | `BTH50015-1LUA` | 1 | 40A 級として暫定。熱設計で再確認 |
| Arm Load Switch | `BTH50030-1LUA` | 1 | 低電流時の暫定候補。ARES X のアーム総電流で再選定 |
| Housekeeping | `TLF35584QKVS2` | 1 | STM32/通信/監視電源 |
| CAN | `TLE9371VSJ` | 1 | CAN 1Mbps、将来 CAN FD |
| Current Sensor | `TLE4973-A075T5-S0001` | 0 から 3 | 入力/足回りの独立電流測定が必要な場合 |
| Current Sensor | `TLE4973-A025T5-S0001` | 0 から 1 | アーム電流測定が必要な場合 |

## 5. 未確定電流に依存する選定ポイント

- 足回りが 40A 連続を本当に要求する場合、`BTH50015-1LUA` 単品では熱的・定格的に不足する可能性がある。最終電流確定後に、並列化、放熱強化、または XDP711 + 外付け MOSFET による出力 eFuse 構成を検討する。
- アームが複数アクチュエータ同時駆動で 20A を大きく超える場合、`BTH50030-1LUA` から `BTH50015-1LUA`、並列構成、または XDP711 + 外付け MOSFET の eFuse 構成へ寄せる。
- 24V DC/DC は 2 系統前提とし、各系統の相数、MOSFET 並列数、インダクタ、電流検出方式は足回り/アームそれぞれの連続電流とピーク電流に合わせて決める。
- TLE4973 の電流レンジは、測定精度とピーク余裕のトレードオフで決める。最終電流が低いほど小さいレンジを選ぶ方がログ分解能を取りやすい。

## 6. 公式参考資料

- XDP711-001: <https://www.infineon.com/part/XDP711-001>
- ISC035N10NM5LF2: <https://www.infineon.com/part/ISC035N10NM5LF2>
- XDPP1188-200C: <https://www.infineon.com/part/XDPP1188-200C>
- 1EDN7116G: <https://www.infineon.com/part/1EDN7116G>
- EVAL_7116G_100V_SSO8: <https://www.infineon.com/evaluation-board/EVAL-7116G-100V-SSO8>
- ISC022N10NM6: <https://www.infineon.com/part/ISC022N10NM6>
- BTH50015-1LUA: <https://www.infineon.com/part/BTH50015-1LUA>
- BTH50030-1LUA: <https://www.infineon.com/part/BTH50030-1LUA>
- TLF35584QKVS2: <https://www.infineon.com/part/TLF35584QKVS2>
- TLF35584QVVS2: <https://www.infineon.com/part/TLF35584QVVS2>
- TLE9371VSJ: <https://www.infineon.com/part/TLE9371VSJ>
- TLE4973-A075T5-S0001: <https://www.infineon.com/part/TLE4973-A075T5-S0001>
- TLE4973-A025T5-S0001: <https://www.infineon.com/part/TLE4973-A025T5-S0001>
