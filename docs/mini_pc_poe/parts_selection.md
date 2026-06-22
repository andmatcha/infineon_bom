# Mini PC / Passive PoE 電源基板 部品選定整理

作成日: 2026-06-18  
入力: [requirements.md](./requirements.md)

## 1. 選定方針

- [requirements.md](./requirements.md) を正とし、現時点で決まっている要件を前提に部品を選定する。
- CSV 形式の BOM は [../../bom/mini_pc_poe_bom.csv](../../bom/mini_pc_poe_bom.csv) に分ける。
- Infineon 製品のみを統合した BOM は [../../bom/infineon_integrated_bom.csv](../../bom/infineon_integrated_bom.csv) に分ける。
- 電源保護、出力スイッチ、パワー MOSFET は Infineon を基本候補とする。
- DC/DC コントローラ、RJ45、コネクタ、ヒューズ、TVS、受動部品は Infineon 縛りの対象外とする。
- 19V 出力は基板定格 5A とし、接続する Mini PC 実機は 19V/3.42A (64.98W)、外径 5.5mm / 内径 2.5mm DC ジャック、センタープラスを前提にする。
- Passive PoE は IEEE PoE ではなく、24V fixed passive injector として扱う。非対応機器へ接続すると破損し得るため、ラベルと pinout を明確にする。
- EOL、Not for new design、用途に対して不適切な候補は部品候補から外し、この文書には残さない。

## 2. 要件から見た構成

```text
Makita 18V Battery C
  -> input fuse / XDP711 hot-swap protection / UVLO
        -> protected input bus
        -> LM51772-Q1 19V buck-boost
              -> BTT6020-1ERA high-side switch
              -> Mini PC 19V / 5A design output
              -> 5.5mm OD / 2.5mm ID center-positive DC plug
        -> LM51561-Q1 24V boost
              -> BTT6030-1ERA high-side switch
              -> fuse / PTC
              -> RJ45 pins 4,5 = +24V; pins 7,8 = GND

Data In RJ45
  -> pins 1,2,3,6 straight-through
  -> PoE Out RJ45
```

入力保護は PDU と同系統の Infineon 部品で揃え、DC/DC コントローラは出力条件に合う TI の現行品を使う。19V 系はバッテリー電圧が 19V をまたぐため 4-switch buck-boost、24V/0.5A は入力より常に高い電圧を作るため boost とする。低電圧遮断は 15.0V、Ethernet は 100Mbps 固定、Mini PC へのシャットダウン通知は初期試作では実装しない。

## 3. 機能別の主選定

### 3.1 バッテリー入力・保護

| 項目 | 選定 |
| --- | --- |
| 主 IC | `XDP711-001` |
| 数量目安 | 1 個 |
| 外付け FET | `ISC035N10NM5LF2` |
| FET 数量目安 | 2 から 4 個。back-to-back 構成と SOA 余裕で変動 |
| 担当機能 | バッテリー入力保護、突入電流制限、低電圧/過電圧/過電流/短絡/過温度保護、入力 telemetry の将来拡張 |
| 採用理由 | `XDP711-001` は +7V から +80V の hot-swap / monitoring controller で、PMBus telemetry と FET SOA 制御を持つ。`ISC035N10NM5LF2` は hot-swap、battery protection、eFuse 用途向けの 100V Linear FET で、バッテリー挿抜と入力容量充電に向く。 |

設計メモ:

- 設計上限では Mini PC 19V/5A と PoE 24V/0.5A の合計出力は 107W である。実接続 Mini PC では 19V/3.42A のため、PoE と合わせた通常負荷は約 77W 級になる。
- 変換効率を見込むと、低入力電圧時の入力連続電流は 8A から 10A 級になる。
- 入力ヒューズは 15A 級を暫定候補とし、バッテリー、配線、XDP711 設定、DC/DC 最大入力電流に合わせて決める。
- XDP711 の低電圧遮断は 15.0V を初期値として設定する。
- XDP711 の PMBus を初期試作で使わない場合でも、SDA/SCL/ALERT/test point を出しておくと低電圧予告や故障ログへ拡張しやすい。
- 低コスト試作では、XDP711 を省いた fuse + TVS + 逆接保護 MOSFET 構成も可能だが、最終候補は入力保護と突入制御を持つ構成とする。

### 3.2 Mini PC 19V/5A buck-boost

| 項目 | 選定 |
| --- | --- |
| DC/DC コントローラ | `LM51772-Q1` |
| 数量目安 | 1 個 |
| パワー MOSFET | `ISC022N10NM6` |
| MOSFET 数量目安 | 4 個。熱評価で並列化を検討 |
| 出力スイッチ | `BTT6020-1ERA` |
| 出力スイッチ数量目安 | 1 個 |
| 担当機能 | マキタ 18V バッテリー入力から基板定格 19V/5A を生成し、実機定格 19V/3.42A の Mini PC へ常時給電する |
| 採用理由 | `LM51772-Q1` は 55V 4-switch buck-boost controller で、入力が出力より高い、同等、低い場合のいずれでも電圧を安定化できる。評価モジュールに 9V から 48V 入力、20V/5A 出力の近い構成があるため、19V/5A Mini PC 電源へ展開しやすい。`ISC022N10NM6` は 100V OptiMOS 6 MOSFET で、既存 PDU の 24V DC/DC と部品を揃えやすい。 |

設計メモ:

- 19V/5A は 95W であり、入力 15V 時は 7A 以上の入力電流を見込む。実接続 Mini PC は 19V/3.42A (64.98W) だが、電源設計は 5A 余裕を残す。
- `LM51772-Q1` は I2C 設定にも対応するが、初期試作では固定 19V 設定で成立する構成を優先する。
- `BTT6020-1ERA` は 20mOhm の 24V 系 smart high-side switch で、Mini PC 出力の保護、診断、fault 表示に使う。5A 連続では約 0.5W 級の導通損失が出るため、銅箔放熱を必ず確認する。
- 既存の LM5176 設計資産を流用したい場合は `LM5176-Q1` も実績重視の代替候補になる。ただし新規設計では、20V/5A に近い評価構成がある `LM51772-Q1` を主候補にする。

### 3.3 Passive PoE 24V/0.5A boost

| 項目 | 選定 |
| --- | --- |
| DC/DC コントローラ | `LM51561-Q1` |
| 数量目安 | 1 個 |
| パワー MOSFET | `ISC022N10NM6` |
| MOSFET 数量目安 | 1 個。低コスト化時は小型 80V/100V MOSFET へ置換可 |
| 出力スイッチ | `BTT6030-1ERA` |
| 出力スイッチ数量目安 | 1 個 |
| 出力ヒューズ/保護 | `TBD_POE_PTC_0P75A` または同等の fuse/PTC |
| 担当機能 | マキタ 18V バッテリー入力から Passive PoE 用 24V/0.5A を生成する |
| 採用理由 | `LM51561-Q1` は wide VIN の non-synchronous boost / SEPIC / flyback controller で、hiccup 保護を持つ。24V/0.5A の小電力 boost として使いやすく、外付け MOSFET を Infineon に寄せられる。 |

設計メモ:

- マキタ 18V 系バッテリーの満充電電圧は 24V より低いため、PoE 系は単純な boost で成立する。
- `BTT6030-1ERA` の保護電流は 0.5A PoE 負荷に対して高めになるため、PoE 出力には別途 0.75A から 1A 程度のヒューズまたは PTC を直列に入れる。
- 24V/0.5A は 12W であり、RJ45 の 4,5 と 7,8 の 2 本並列で給電する。各導体の電流は理想的には約 0.25A となるが、ケーブル長、接触抵抗、片線断線時の偏りを考慮する。
- 24V boost のスイッチングノードは RJ45 と Ethernet データペアから離し、PoE 24V にはコモンモードノイズと差動ノイズのフィルタを入れる。

### 3.4 RJ45 / Passive PoE 配線

| 項目 | 選定 |
| --- | --- |
| RJ45 ジャック | `615008137421` |
| メーカー | Wurth Elektronik |
| 数量目安 | 2 個。Data In と PoE Out |
| Ethernet ESD | `TPD4E05U06DQAR` |
| ESD 数量目安 | 1 個。1,2,3,6 の 4 線を保護 |
| 担当機能 | 100Mbps Ethernet pass-through、24V Passive PoE 注入、ESD 保護 |
| 採用理由 | `615008137421` は shielded 8P8C RJ45 jack で、磁気部品なしのため、Data In と PoE Out の間を基板上で直結する passive injector に使いやすい。`TPD4E05U06DQAR` は低容量 4ch ESD 保護で、Ethernet データ線の保護に使える。 |

Pinout:

| PoE Out RJ45 pin | 機能 |
| ---: | --- |
| 1 | Data pair 1、Data In pin 1 へ直結 |
| 2 | Data pair 1、Data In pin 2 へ直結 |
| 3 | Data pair 2、Data In pin 3 へ直結 |
| 4 | +24V |
| 5 | +24V |
| 6 | Data pair 2、Data In pin 6 へ直結 |
| 7 | GND |
| 8 | GND |

注意点:

- この構成は 10BASE-T / 100BASE-TX 用である。1000BASE-T 以上では 4,5,7,8 もデータに使うため、今回の pinout とは両立しない。
- Data In 側の 4,5,7,8 は未接続または保護用に処理し、上流スイッチやルータへ 24V を戻さない。
- RJ45 シールドは、0ohm、RC、DNP を選べるフットプリントにしておくと、ローバー搭載時のノイズ評価で調整しやすい。
- PoE Out コネクタには `24V PASSIVE POE ONLY` と明記する。

### 3.5 コネクタ・ヒューズ・受動部品

| 項目 | 選定 |
| --- | --- |
| バッテリーマウント | `TBD_MAKITA_18V_BATTERY_MOUNT` |
| 入力ヒューズ | `TBD_INPUT_FUSE_15A` |
| Mini PC DC プラグ | `KLDX-PA-0202-B` |
| PoE 出力ヒューズ/PTC | `TBD_POE_PTC_0P75A` |
| 入力/出力 TVS | `TBD_TVS_SET` |
| 19V 電源インダクタ | `TBD_19V_POWER_INDUCTOR` |
| 24V boost インダクタ | `TBD_POE_BOOST_INDUCTOR` |
| 24V boost ダイオード | `TBD_POE_BOOST_DIODE` |
| 状態表示 | `TBD_LED_TESTPOINT_SET` |

設計メモ:

- Mini PC 出力ケーブルは外径 5.5mm / 内径 2.5mm、センタープラスの DC プラグとする。`KLDX-PA-0202-B` は 2.5mm ID / 5.5mm OD のケーブル実装プラグ候補である。
- バッテリーマウントはローバー搭載時の振動で抜けない構造にする。
- 入力ヒューズはバッテリー直近に置き、基板上の eFuse/Hot-swap 保護と役割を分ける。
- PoE 24V は RJ45 へ入る直前に fuse/PTC を置き、ケーブル短絡時に配線と RJ45 が過熱しないようにする。

## 4. 暫定 BOM

| ブロック | 品番 | 数量目安 | メモ |
| --- | --- | ---: | --- |
| Battery Input | `XDP711-001` | 1 | 入力保護、突入制限、UVLO、telemetry 拡張 |
| Battery Input FET | `ISC035N10NM5LF2` | 2 から 4 | back-to-back と SOA 余裕で変動 |
| 19V Buck-Boost Controller | `LM51772-Q1` | 1 | 基板定格 19V/5A、実機負荷 19V/3.42A |
| 19V / 24V Power MOSFET | `ISC022N10NM6` | 5 | 19V buck-boost に 4、24V boost に 1 |
| 19V Output Switch | `BTT6020-1ERA` | 1 | 5A 連続の熱設計が必要 |
| 24V Boost Controller | `LM51561-Q1` | 1 | Passive PoE 24V/0.5A |
| 24V Output Switch | `BTT6030-1ERA` | 1 | PoE 出力。別途 fuse/PTC を併用 |
| RJ45 Jack | `615008137421` | 2 | Data In / PoE Out |
| Ethernet ESD | `TPD4E05U06DQAR` | 1 | 1,2,3,6 の 4 線用 |
| Input Fuse | `TBD_INPUT_FUSE_15A` | 1 | バッテリー直近 |
| PoE Fuse/PTC | `TBD_POE_PTC_0P75A` | 1 | 0.75A から 1A 級で要確認 |
| Battery Mount | `TBD_MAKITA_18V_BATTERY_MOUNT` | 1 | 機械固定含めて選定 |
| Mini PC DC Plug | `KLDX-PA-0202-B` | 1 | 外径 5.5mm / 内径 2.5mm、センタープラス |
| Power Passives | `TBD_MINI_PC_POE_PASSIVE_SET` | 1 set | インダクタ、コンデンサ、分圧、sense、TVS、フィルタ |
| LED / Test Point | `TBD_LED_TESTPOINT_SET` | 1 set | 入力、19V、24V、fault |

## 5. 反映済み条件と設計注意

- Mini PC 実機は 19V/3.42A だが、基板側は 19V/5A 定格で設計する。突入電流と入力容量が大きい場合、19V 出力スイッチの current limit、soft-start、出力容量を再設定する。
- Mini PC コネクタが USB-C PD である場合、固定 19V 出力ではなく USB-C PD source controller が必要になる。現時点では barrel jack などの固定 19V 入力を前提にしている。
- 低電圧遮断閾値は 15.0V とする。使用可能時間とバッテリー保護のバランスは実運用で確認する。
- 24V Passive PoE 機器の許容入力範囲が狭い場合、ケーブル電圧降下を見込んで 24V 出力精度と電流制限を詰める。
- Ethernet は 100Mbps 固定とする。1000BASE-T が必要になった場合、この RJ45 pinout は使えないため、IEEE PoE PSE、別電源線、または 4-pair 対応の設計へ切り替える。
- RJ45 シールドは 0ohm、RC、DNP を選べるフットプリントを用意し、ローバー全体のシャーシ/GND 方針で実装を決める。
- 初期試作では Mini PC へのシャットダウン通知やバッテリー残量通知は実装せず、常時給電のみとする。
- `ISC022N10NM6` は PoE 12W boost には過剰性能である。部品共通化を優先して暫定採用し、量産や小型化ではより小さい 80V/100V MOSFET へ置き換える。

## 6. 公式参考資料

- XDP711-001: <https://www.infineon.com/part/XDP711-001>
- ISC035N10NM5LF2: <https://www.infineon.com/part/ISC035N10NM5LF2>
- LM51772-Q1: <https://www.ti.com/product/LM51772-Q1>
- ISC022N10NM6: <https://www.infineon.com/part/ISC022N10NM6>
- BTT6020-1ERA: <https://www.infineon.com/part/BTT6020-1ERA>
- BTT6030-1ERA: <https://www.infineon.com/part/BTT6030-1ERA>
- LM51561-Q1: <https://www.ti.com/product/LM51561-Q1>
- TPD4E05U06: <https://www.ti.com/product/TPD4E05U06>
- 615008137421: <https://www.we-online.com/components/products/datasheet/615008137421.pdf>
- KLDX-PA-0202-B: <https://www.kycon.com/Catalog_PDF/KLDX-P.pdf>
