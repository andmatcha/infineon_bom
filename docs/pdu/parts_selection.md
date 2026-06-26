# Infineon PDU 部品選定整理

作成日: 2026-06-18  
更新日: 2026-06-26
入力: [requirements.md](./requirements.md)

## 1. 選定方針

- [requirements.md](./requirements.md) を正とし、解決済みの要件を前提に部品を再選定する。
- 2026-06-23 の要件変更により、入力はマキタ 40V バッテリー (定格 36V) 2 個へ変更された。18V 入力前提の電圧範囲、低電圧閾値、DC/DC 構成、BOM 数量は現行要件として使わない。
- 2026-06-26 の要件変更により、自作アームアクチュエータの設計要件と専用 BOM は除外し、アーム系は既製アクチュエータへの電源供給だけを PDU 側で扱う。
- 本文中の部品候補は、40V バッテリー 2 個の合成方式、足回り出力電圧、常時給電枝の仕様が確定するまで暫定候補として扱う。
- 2026-06-26 に `dktools` で Digi-Key 取扱、Active 状態、在庫の有無を再確認した。Digi-Key 発注に使える品番があるものは、短縮名ではなく発注可能な MPN を優先して記載する。Infineon 部品は、Digi-Key 在庫がない場合でも Infineon 公式サイトで確認できれば採用候補として維持する。
- CSV 形式の BOM は [../../bom/pdu_bom.csv](../../bom/pdu_bom.csv) に分ける。
- 電源保護、スイッチ、CAN、センサは Infineon を基本とする。
- STM32 MCU、XBee、コネクタ、ヒューズ、受動部品は Infineon 縛りの対象外とする。
- EOL、Not for new design、用途に対して不適切な候補は部品候補から外し、この文書には残さない。
- できるだけ現行かつ新しい製品を優先する。ただし、電流、放熱、SOA、PCB 銅箔、コネクタ定格が未確定の高電流部は暫定選定とする。

## 2. 要件から見た構成

```text
Makita 40V Battery A, 36V nominal
  -> hot-swap / input protection / telemetry
Makita 40V Battery B, 36V nominal
  -> hot-swap / input protection / telemetry
        -> protected input bus
        -> E-Stop controlled high-power bus
        -> Foot DC/DC or protected output, 24V to 36V class
              -> Foot output bus
              -> high-side switch -> Foot drive, provisional 40A class
        -> Arm actuator 24V supply, 30A cont / 60A 10s / 90A 100ms provisional
              -> high-side switch -> off-the-shelf arm actuator power input

Always-on protected auxiliary bus, not E-Stop controlled
  -> Mini PC 19V power board / rail
  -> Wi-Fi antenna module power board / rail

Always-on control rail
  -> STM32 MCU
  -> XBee
  -> CAN transceiver
  -> telemetry / watchdog / E-Stop monitoring
```

## 3. 機能別の主選定

### 3.1 バッテリー入力・ホットスワップ

| 項目 | 選定 |
| --- | --- |
| 主 IC | `XDP711001XUMA1` (`XDP711-001`) |
| 数量目安 | 2 個、Battery A/B に各 1 個 |
| 外付け FET | `ISC035N10NM5LF2ATMA1` |
| FET 数量目安 | 4 から 8 個。back-to-back 構成と並列数で変動 |
| 担当機能 | ホットスワップ、突入電流制限、入力保護、PMBus テレメトリ、FET SOA 保護、下流系の Disable |
| 採用理由 | `XDP711-001` は `+7V` から `+80V` のホットスワップ/監視コントローラで、現行世代の新しい候補。`ISC035N10NM5LF2` は hot swap、battery protection、e-fuse 用途向けの 100V Linear FET で、リニア領域の SOA を重視する入力保護に向く。 |

設計メモ:

- マキタ 40V バッテリー 2 個を個別に保護、監視する。
- 2 個のバッテリーの直列、並列、ORing、負荷分担方式は未定義のため、部品耐圧と FET SOA は合成方式確定後に再確認する。
- 逆流防止は FET の向きと back-to-back 構成で決める。
- E-Stop のハードウェア停止経路は足回り・アーム高電力系を遮断し、Mini PC / Wi-Fi アンテナモジュール用の常時給電枝は遮断しない。
- `XDP711-001` は +80V までの候補であるため、バッテリー 2 個を直列扱いする場合や過渡電圧が大きい場合は耐圧余裕を再評価する。
- XDP711 の PMBus 設定値、FET SOA プロファイル、短絡遮断条件は、実負荷電流と入力容量が決まった後に確定する。
- `XDP711001XUMA1` と `ISC035N10NM5LF2ATMA1` は Infineon 公式サイトで確認済みのため、Digi-Key 在庫がなくても採用候補として維持する。2026-06-26 時点の Digi-Key ではどちらも Active だが在庫 0 であるため、調達経路、入力合成方式、ヒューズ定格は設計段階で別途確認する。

### 3.2 主電源生成

| 項目 | 選定 |
| --- | --- |
| デジタル電源コントローラ | `XDPP1188-200C` |
| 数量目安 | 2 個。足回り用 DC/DC とアーム用 24V 電源に各 1 個 |
| ゲートドライバ | `1EDN7116GXTMA1` |
| パワー MOSFET | `ISC022N10NM6ATMA1` |
| 担当機能 | マキタ 40V バッテリー 2 個系入力から、足回り用 24V から 36V 程度の主電源と既製アームアクチュエータ用 24V 電源を生成または制御し、故障監視と STM32 へのテレメトリ連携を行う |
| 採用理由 | `XDPP1188-200C` は新しい XDP デジタル電源コントローラで、8 PWM、PMBus/I2C/UART/SPI を備え、isolated/non-isolated の高密度 DC/DC に使える。`1EDN7116G` と `ISC022N10NM6` は 100V half-bridge 評価構成でも使われており、40V 系から 24V/36V 級を扱う候補になる。 |

構成方針:

- 足回りとアームのノイズ分離、負荷変動、片系統故障時の継続運用を優先し、足回り主電源とアーム 24V は 2 系統に分ける。
- 足回り用電源とアーム用 24V 電源は、Enable、fault/warn、電圧・電流監視を個別に扱う。
- 片方の DC/DC または出力系統が故障した場合、入力電源、E-Stop、MCU、通信、共通保護に異常がなければ、他方の主電源系統と常時給電系を継続運用できる構成にする。
- DC/DC の詳細トポロジは、各系統の入力電圧範囲、必要電流、効率目標、放熱条件から決定する。
- `XDPP1188-200C` は Infineon 公式サイトで確認済みのため、2026-06-26 時点で Digi-Key API から該当品を取得できなくても Infineon アーキテクチャ候補として採用する。足回り出力電圧と各系統電流が確定した時点で、トポロジ適合性を再評価する。
- `1EDN7116GXTMA1` は 2026-06-26 時点で Active / Digi-Key 在庫 7369 を確認済み。`ISC022N10NM6ATMA1` は Active だが、同日時点の Digi-Key 在庫は 0 である。

### 3.3 足回り・アーム ON/OFF

| 項目 | 足回り出力 | アーム 24V |
| --- | --- | --- |
| 主選定 | `BTH500151LUAAUMA1` (`BTH50015-1LUA`) | `BTH500151LUAAUMA1` (`BTH50015-1LUA`) または外付け FET 型 eFuse |
| 数量目安 | 1 個 | 1 個 |
| 想定電流 | 初期 24V / 40A 級。10 秒ピーク、100 ms ピークは下流仕様または実測で再確認 | 旧来同等として連続 30A、10 秒ピーク 60A、100 ms ピーク 90A |
| 担当機能 | 高電流ハイサイド ON/OFF、過電流保護、短絡保護、温度保護、診断、電流センス | 高電流ハイサイド ON/OFF、過電流保護、短絡保護、温度保護、診断、電流センス |

採用理由:

- `BTH50015-1LUA` は Power PROFET+ 24/48V 系のスマートハイサイドスイッチで、90A max クラス、低オン抵抗、診断機能を持つため、足回り 40A 級とアーム 30A/60A/90A 初期設計負荷の出力遮断候補にする。
- `BTH500301LUAAUMA1` (`BTH50030-1LUA`) は 2026-06-26 時点で Active / Digi-Key 在庫 2545 だが 25A max クラスのため、足回り 40A 級やアーム 30A/60A/90A 初期設計負荷には採用しない。既製アクチュエータの型番確定後、実負荷が十分低ければ再評価できる。
- `BTH500151LUAAUMA1` は Infineon 公式サイトで確認済みで、2026-06-26 時点の Digi-Key でも Active だが在庫 0 である。Digi-Key 在庫がなくても採用候補として維持し、電流・熱条件に対して不足する場合は、並列構成または XDP711 + 外付け MOSFET による eFuse 構成を検討する。
- 足回り 40A 級は最終電流が未確定のため、`BTH50015-1LUA` 単品で足りるか、放熱強化・並列・ディスクリート構成が必要かを最終設計で判断する。
- アーム電源出力は既製アクチュエータ向けだが、型番未定の間は旧来のアーム電源要求と同等の 24V / 30A 連続 / 60A 10 秒ピーク / 90A 100 ms ピークを前提に `BTH50015-1LUA`、並列構成、または XDP711 + 外付け MOSFET による eFuse 構成を検討する。型番確定後に実際の入力電流、突入電流、回生またはブレーキ時の挙動で最適化する。

注意点:

- PROFET は負荷スイッチであり、モータ PWM 駆動用ではない。既製アームアクチュエータ内部のモータ制御は PDU 側の設計対象外とする。
- 大容量負荷を直投入する場合は、XDP711 側のソフトスタートや DC/DC 側の立ち上げシーケンスと組み合わせる。
- E-Stop では足回り・アーム側の PROFET 入力を落とし、同時に該当 DC/DC 停止も組み合わせる。常時給電枝の入力保護は E-Stop で落とさない。

### 3.4 常時電源・監視電源

| 項目 | 選定 |
| --- | --- |
| 主 IC | `TLF35584QKVS2XUMA2` (`TLF35584QKVS2`) |
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
| 主 IC | `TLE9371VSJXTMA1` (`TLE9371VSJ`) |
| 数量目安 | 1 個 |
| 担当機能 | CAN 1Mbps 初期対応、将来の CAN FD 対応、Power Board と Rover MCU / Arm MCU の接続 |
| 採用理由 | `TLE9371VSJ` は active and preferred の CAN FD SIC transceiver で、最大 8Mbit/s、3.3V/5V supply、EMC 改善を狙える。初期 1Mbps CAN に対して十分な余裕があり、将来の CAN FD 化にもつなげやすい。 |

注意点:

- 終端抵抗 120 Ohm、TVS、コモンモードチョーク、コネクタピン配置、GND/シールド接続は基板仕様で決める。
- STM32 側の CAN peripheral が CAN FD 非対応の場合でも、1Mbps CAN transceiver として利用する。

### 3.6 電流センサ

| 測定箇所 | 選定 | 数量目安 | 理由 |
| --- | --- | ---: | --- |
| Battery A/B 入力 | `TLE4973A075T5S0001XUMA1` (`TLE4973-A075T5-S0001`) | 0 から 2 | ±82A 範囲で入力電流ピークに余裕を持たせる |
| 足回り出力 | `TLE4973A075T5S0001XUMA1` (`TLE4973-A075T5-S0001`) | 0 から 1 | 暫定 40A 級でピーク未確定のため余裕を持たせる |
| アーム 24V | `TLE4973A075T5S0001XUMA1` (`TLE4973-A075T5-S0001`) | 0 から 1 | 初期設計ピーク 90A に近いため高レンジを優先し、型番確定後にログ分解能とのトレードオフを再評価 |

採用理由:

- TLE4973 系は XENSIV の新しい coreless current sensor で、5V 駆動、内蔵 current rail、過電流検出、診断インターフェースを持つ。
- 保護用途は XDP711/PROFET の内蔵診断と高速 fault を優先し、ログ精度や独立電流測定が必要な箇所に TLE4973 を追加する。
- `TLE4973A075T5S0001XUMA1` と 25A レンジの `TLE4973A025T5S0001XUMA1` は Infineon 公式サイトで確認済みで、2026-06-26 時点の Digi-Key 在庫もそれぞれ 1838 / 2482 である。

## 4. 暫定 BOM

| ブロック | 品番 | 数量目安 | メモ |
| --- | --- | ---: | --- |
| Battery Input | `XDP711001XUMA1` | 2 | Infineon 公式確認済み。Digi-Key Active / 在庫 0。Battery A/B に各 1 |
| Battery Input FET | `ISC035N10NM5LF2ATMA1` | 4 から 8 | Infineon 公式確認済み。Digi-Key Active / 在庫 0。back-to-back と並列数で変動 |
| Main Digital Power | `XDPP1188-200C` | 2 | Infineon 公式確認済み。Foot/Arm 電源に各 1。40V 系入力、足回り 24V から 36V 化、アーム 24V 30A/60A/90A 初期設計負荷を前提に再確認 |
| Main Power Gate Driver | `1EDN7116GXTMA1` | 4 から 8 | Active / Digi-Key 在庫 7369。2 系統分。DC/DC トポロジとスイッチ数で変動 |
| Main Power MOSFET | `ISC022N10NM6ATMA1` | 8 から 32 | Infineon 公式確認済み。Digi-Key Active / 在庫 0。2 系統分。相数、並列数、入力合成方式で変動 |
| Foot Load Switch | `BTH500151LUAAUMA1` | 1 | Infineon 公式確認済み。Digi-Key Active / 在庫 0。40A 級として暫定。熱設計で再確認 |
| Arm Load Switch | `BTH500151LUAAUMA1` | 1 | Infineon 公式確認済み。Digi-Key Active / 在庫 0。24V / 30A 連続 / 60A 10 秒ピーク / 90A 100 ms ピークを前提に並列または eFuse 構成も検討。型番確定後に最適化 |
| Housekeeping | `TLF35584QKVS2XUMA2` | 1 | Active / Digi-Key 在庫 1812。STM32/通信/監視電源 |
| CAN | `TLE9371VSJXTMA1` | 1 | Active / Digi-Key 在庫 6968。CAN 1Mbps、将来 CAN FD |
| Current Sensor | `TLE4973A075T5S0001XUMA1` | 0 から 3 | Active / Digi-Key 在庫 1838。入力/足回りの独立電流測定が必要な場合 |
| Current Sensor | `TLE4973A025T5S0001XUMA1` | 0 から 1 | Active / Digi-Key 在庫 2482。低電流レンジが有利な箇所のみ |

## 5. 電流条件に依存する選定ポイント

- 足回りが 40A 連続を本当に要求する場合、`BTH50015-1LUA` 単品では熱的・定格的に不足する可能性がある。最終電流確定後に、並列化、放熱強化、または XDP711 + 外付け MOSFET による出力 eFuse 構成を検討する。
- アームは旧来同等の初期設計値として連続 30A、10 秒ピーク 60A、100 ms ピーク 90A とするため、`BTH50030-1LUA` 単品ではなく `BTH50015-1LUA`、並列構成、または XDP711 + 外付け MOSFET の eFuse 構成へ寄せる。既製アクチュエータの型番確定後、実負荷と突入/回生条件に合わせて最適化する。
- 足回り主電源とアーム 24V は 2 系統前提とし、各系統の相数、MOSFET 並列数、インダクタ、電流検出方式は足回り/アームそれぞれの連続電流とピーク電流に合わせて決める。
- Mini PC / Wi-Fi アンテナモジュール用常時給電枝は E-Stop 対象外のため、足回り・アーム出力スイッチとは別に、専用ヒューズ、eFuse、入力フィルタ、TVS、power good/fault 線を持つ構成として再選定する。
- TLE4973 の電流レンジは、測定精度とピーク余裕のトレードオフで決める。最終電流が低いほど小さいレンジを選ぶ方がログ分解能を取りやすい。

## 6. 公式参考資料

- XDP711001XUMA1 / XDP711-001: <https://www.infineon.com/part/XDP711-001>
- ISC035N10NM5LF2ATMA1 / ISC035N10NM5LF2: <https://www.infineon.com/part/ISC035N10NM5LF2>
- XDPP1188-200C: <https://www.infineon.com/part/XDPP1188-200C>
- 1EDN7116GXTMA1 / 1EDN7116G: <https://www.infineon.com/part/1EDN7116G>
- EVAL_7116G_100V_SSO8: <https://www.infineon.com/evaluation-board/EVAL-7116G-100V-SSO8>
- ISC022N10NM6ATMA1 / ISC022N10NM6: <https://www.infineon.com/part/ISC022N10NM6>
- BTH500151LUAAUMA1 / BTH50015-1LUA: <https://www.infineon.com/part/BTH50015-1LUA>
- TLF35584QKVS2XUMA2 / TLF35584QKVS2: <https://www.infineon.com/part/TLF35584QKVS2>
- TLF35584QVVS2: <https://www.infineon.com/part/TLF35584QVVS2>
- TLE9371VSJXTMA1 / TLE9371VSJ: <https://www.infineon.com/part/TLE9371VSJ>
- TLE4973A075T5S0001XUMA1 / TLE4973-A075T5-S0001: <https://www.infineon.com/part/TLE4973-A075T5-S0001>
- TLE4973A025T5S0001XUMA1 / TLE4973-A025T5-S0001: <https://www.infineon.com/part/TLE4973-A025T5-S0001>
