# infineon_bom

ARES X 向けに、Infineon 部品を中心とした PDU と CAN アームアクチュエータの要件、部品選定、BOM を整理するためのリポジトリです。

## 内容

- `docs/pdu/`: マキタ 18V バッテリー 2 本を入力とする PDU の要件整理と部品選定
- `docs/arm_actuator/`: CAN 接続アームアクチュエータの要件整理と部品選定
- `bom/`: PDU、アームアクチュエータ、統合構成の CSV BOM

## 方針

- 電源保護、スイッチ、CAN、センサなどは Infineon 部品を基本候補とする。
- STM32、XBee、コネクタ、ヒューズ、受動部品、機械部品は用途に応じて別メーカーも含める。
- EOL 品や Not for new design の部品は候補から外し、現行で入手性の高い部品を優先する。
