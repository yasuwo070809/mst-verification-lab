# 試験計画

対象ラボ: `mst-verification-lab` (CML lab id `8cd6a53c-6870-4891-b51c-7ebe586f8360`)、IOL-L2 (`ioll2-xe`, image `ioll2-xe-17-18-02`) ×4台。

共通設定はREADMEおよび`configs/SW{1-4}.cfg`を参照。すべての試験はCML APIで作成したラボ上で、コンソール(WebSocket)経由の実CLI投入・実show出力採取によって実施し、期待結果ではなく実測結果を記録する。

## 試験1: VLANマッピング

**目的**: VLAN 10,20がMST1に、VLAN 30,40がMST2に正しくマッピングされていることを確認する。

**手順**: 全スイッチで以下を実行。
```
show spanning-tree mst configuration
show spanning-tree mst
show spanning-tree summary
```

**保存先**: `outputs/baseline/`

## 試験2: ルートブリッジと負荷分散

**目的**: MST1のルートがSW1、MST2のルートがSW2であること、かつMST1とMST2でポートロール(特にブロックされるリンク)が異なることを確認する。

**手順**: 全スイッチで以下を実行。
```
show spanning-tree mst 1
show spanning-tree mst 2
show spanning-tree root
```

**保存先**: `outputs/baseline/`

## 試験3: 高速収束

**目的**: Root Portが属するリンク障害発生時の、Alternate PortからRoot/Designatedへの遷移と収束時間を実測する。

**制約への対応**: 元仕様にはVLAN10上の「端末」やIPアドレッシングが含まれていないため、ping代替としてVLAN10のSVI (`192.168.10.1-4/24`) を試験専用に一時追加する(ユーザー確認済み)。

**手順**:
1. SW4のMST1における現在のRoot Port(Et0/0, SW2方向)とAlternate Port(Et0/1, SW3方向)を確認する。
2. SW1コンソールから`ping 192.168.10.4 repeat 2 timeout 1`を短い間隔で繰り返し実行しながら、SW2コンソールで`interface Ethernet0/1` → `shutdown`によりSW2–SW4間リンクを片側shutdownする。
3. SW4コンソールで`show spanning-tree mst 1`を反復ポーリングし、ポートロール遷移をタイムスタンプ付きで記録する。
4. 安定後、SW2側で`no shutdown`により復旧し、元のRoot Port構成に戻ることを確認する。
5. 最後に`show spanning-tree mst 1 detail`を取得する。

**保存先**: `outputs/failover/` (`timeline.log`, `raw-capture.log`, 最終detail)

## 試験4: リージョン名不一致

**目的**: `spanning-tree mst configuration`の`name`がSW4だけ異なる場合に、SW4との接続がリージョン境界として扱われることを確認する。

**手順**: SW4のみ`name WRONG-MST`に変更 → SW2・SW3・SW4で`show spanning-tree mst configuration`, `show spanning-tree mst detail`を取得 → `name CML-MST`へ復旧し再取得。

**保存先**: `outputs/region-name-mismatch/` (`mismatch-*`, `reverted-*`)

## 試験5: リビジョン番号不一致

**目的**: リビジョン番号のみが異なる場合も別リージョンとして扱われることを確認する。あわせて「リビジョン番号が大きい方が優先される」という誤解が実際には成立しないことを示す。

**手順**: SW4のみ`revision 2`に変更 → 同様に採取 → `revision 1`へ復旧。

**保存先**: `outputs/revision-mismatch/`

## 試験6: VLANマッピング不一致

**目的**: インスタンスへのVLAN割当てが異なる場合も別リージョンとして扱われることを確認する。

**手順**: SW4のみ`instance 1 vlan 10` / `instance 2 vlan 20,30,40`に変更 → 同様に採取 → 共通設定(`instance 1 vlan 10,20` / `instance 2 vlan 30,40`)へ復旧。

**保存先**: `outputs/vlan-mapping-mismatch/`

## 試験7: Rapid PVST+との比較

**目的**: MST(CIST+2インスタンス、計3インスタンス)とRapid PVST+(VLANごとに独立、本ラボでは4インスタンス)の構造的な違いを実測し比較する。

**手順**:
1. 全スイッチの`show running-config`をバックアップ。
2. 全スイッチで`spanning-tree mode rapid-pvst`に変更。
3. `show spanning-tree summary`, `show spanning-tree`を取得。
4. 全スイッチで`spanning-tree mode mst`に復旧し、`spanning-tree mst configuration`が保持されていることを確認。

**評価方針**: CPU使用率など負荷に関する定量比較は、VLAN数がわずか4つのラボ環境では有意差が出ない可能性が高いため実施しない。STPインスタンス数とBPDU構造(MSTP BPDU 1本にMレコードとして複数インスタンス情報を multiplexする方式 vs. VLANごとに個別のPVST+ BPDUを送信する方式)の構造的差異のみで比較する。

**保存先**: `outputs/rapid-pvst-comparison/`
