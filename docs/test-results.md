# 試験結果(実測)

すべての値はCMLコンソール(WebSocket)経由で実際に投入したCLIコマンドの実出力を証跡として記録したものであり、推測・期待値の記載ではない。出典ファイルは各セクションに記載。

## 試験1: VLANマッピング

`outputs/baseline/SW1-show-spanning-tree-mst-configuration.txt` (全台同一内容を確認):

```
Name      [CML-MST]
Revision  1     Instances configured 3

Instance  Vlans mapped
--------  ---------------------------------------------------------------------
0         1-9,11-19,21-29,31-39,41-4094
1         10,20
2         30,40
```

**確認結果**: VLAN 10, 20 → MST1、VLAN 30, 40 → MST2 のマッピングを4台全てで実測確認。期待結果との差異なし。

## 試験2: ルートブリッジと負荷分散

出典: `outputs/baseline/SW{1-4}-show-spanning-tree-mst-{1,2}.txt`, `SW{1-4}-show-spanning-tree-root.txt`

### MST1 (VLAN 10,20) ポートロール表

| スイッチ | Et0/0 (相手) | Et0/1 (相手) | Root/備考 |
|---|---|---|---|
| SW1 | Desg/FWD (SW2) | Desg/FWD (SW3) | **Root** (priority 24577) |
| SW2 | Root/FWD (SW1) | Desg/FWD (SW4) | |
| SW3 | Root/FWD (SW1) | Desg/FWD (SW4) | |
| SW4 | Root/FWD (SW2) | **Altn/BLK (SW3)** | |

### MST2 (VLAN 30,40) ポートロール表

| スイッチ | Et0/0 (相手) | Et0/1 (相手) | Root/備考 |
|---|---|---|---|
| SW1 | Root/FWD (SW2) | Desg/FWD (SW3) | |
| SW2 | Desg/FWD (SW1) | Desg/FWD (SW4) | **Root** (priority 24578) |
| SW3 | Root/FWD (SW1) | **Altn/BLK (SW4)** | |
| SW4 | Root/FWD (SW2) | Desg/FWD (SW3) | |

**確認結果**: MST1のルートはSW1、MST2のルートはSW2であることを実測確認(期待結果と一致)。SW3–SW4リンクは、MST1ではSW4側がブロック、MST2ではSW3側がブロックされており、**MST1とMST2でブロック位置(=転送パス)が異なる**ことを実測確認した(要求されていた「一部のRoot PortまたはAlternate Portが異なること」を満たす)。

## 試験3: 高速収束

出典: `outputs/failover/timeline.log`, `raw-capture.log`, `SW4-show-spanning-tree-mst-1-detail-final.txt`

**試験対象**: SW4のMST1 Root Port(Et0/0, SW2方向)。対向リンク(SW2側Et0/1)をshutdownして障害を模擬。ping送信元はSW1、宛先はSW4のVLAN10 SVI(`192.168.10.4`、試験専用に追加)。SW4は自身のEt0/0/Et0/1ロールを`show spanning-tree mst 1`で反復ポーリング(約0.3〜2.4秒間隔)。

**試験前の状態(SW4, MST1)**: Et0/0=Root/FWD、Et0/1=Alternate/BLK

### タイムライン(実測、`timeline.log`より抜粋)

| 時刻 | イベント |
|---|---|
| 23:24:39.145 | **障害発生**: SW2にて`interface Ethernet0/1` → `shutdown` 投入(SW2–SW4リンクダウン) |
| 23:24:39.144〜39.853 | 直前・直後の観測ではまだ旧状態(Et0/0=Root/FWD, Et0/1=Altn/BLK)。この時点のpingは成功(2/2) |
| 23:24:42.257 | ping 0% (0/2) — 断絶を検知した最初のサンプル |
| 23:24:42.620 | SW4のSTP状態はまだ旧状態のまま(未収束) |
| 23:24:45.003 | ping 0% (0/2) — 断絶継続 |
| 23:24:45.360 | SW4のSTP状態が新状態に変化: Et0/1=**Root/FWD**、Et0/0=Designated/BLK(収束をこの時点で確認) |
| 23:24:45.676 | ping 100% (2/2) — **復旧後最初の成功ping** |
| 23:24:49.630 | **復旧操作**: SW2にて`no shutdown`投入 |
| 23:24:49.630〜53.242 | この間、pingはすべて100%(欠損なし)— 旧経路(SW3経由)が有効なまま切替待ちのため無瞬断 |
| 23:24:53.587 | SW4のSTP状態が元に戻る: Et0/0=**Root/FWD**、Et0/1=Alternate/BLK(コスト優先経路への復帰) |

### 測定結果まとめ

- **障害発生時刻**: 23:24:39.145
- **Alternate→Root遷移ポート**: SW4 Et0/1 (Ethernet0/1、SW3方向)
- **Forwarding移行の確認時刻**: 23:24:45.360より前(直近の既知未収束サンプルは23:24:42.620)。ポーリング間隔の制約により、**実際の収束は23:24:42.620〜23:24:45.360の間(障害発生から約3.5〜6.2秒後)**と特定した。
- **ping欠損数**: 4パケット(iter4: 0/2, iter5: 0/2の2バーストぶん)。復旧側(no shutdownからの再収束)は欠損0。
- **推定収束時間**: 約3.5〜6.2秒(STP状態遷移ベース)。ping到達再開ベースでは障害発生から約6.5秒(39.145→45.676)。
- **測定精度に関する注記**: 本測定はコンソールCLIへのポーリング(約0.3〜2.4秒間隔、CML APIおよびWebSocketの往復遅延を含む)によるものであり、ハードウェアキャプチャによる厳密な収束時間測定ではない。実際のMST/RSTP収束はこのポーリング間隔より短い可能性がある。数値は「実測でこの範囲に収まった」という記録であり、それ以上の精度を主張しない。

**試験後の復旧確認**: `SW4-show-spanning-tree-mst-1-detail-final.txt`にてEt0/0=root forwarding、Et0/1=alternate blockingを確認し、`forward transitions 3`(Et0/0)/`1`(Et0/1)のカウンタ増加から、実際に状態遷移が発生したことを確認した。試験後はSW2側を`no shutdown`済みで、インターフェースは正常状態に復旧している。

## 試験4: リージョン名不一致

出典: `outputs/region-name-mismatch/mismatch-*.txt`, `reverted-*.txt`

SW4のみ`spanning-tree mst configuration`の`name`を`WRONG-MST`に変更した結果:

- **SW4自身の視点** (`mismatch-SW4-show-spanning-tree-mst-detail.txt`): CIST(MST0)で`Regional Root this switch`、MST1/MST2でも`Root this switch`と表示され、**SW4単独で独立したMSTリージョンを形成**したことを確認。ポートロールも`master forwarding`(リージョン境界ポート)に変化。
- **隣接スイッチSW2の視点** (`mismatch-SW2-show-spanning-tree-mst-detail.txt`): SW2自身はCML-MSTリージョンに留まり、SW4向けポート(Et0/1)の`Design. regional root`の値がEt0/0(リージョン内)と異なる形で表示され、境界ポートとして扱われていることを確認。

**確認結果**: リージョン名の不一致により、SW4との接続がリージョン境界(CISTのみでBPDU交換)として扱われることを実測確認。復旧後(`reverted-*.txt`)は全台で`Name [CML-MST]`に戻り、正常なMST1/MST2構成(試験1・2と同じ状態)に復帰したことを確認した。

## 試験5: リビジョン番号不一致

出典: `outputs/revision-mismatch/mismatch-*.txt`, `reverted-*.txt`

SW4のみ`revision`を`2`に変更(name・VLANマッピングは共通設定のまま)した結果、試験4と同様に**SW4が単独リージョン化**することを実測確認(`mismatch-SW4-show-spanning-tree-mst-detail.txt`: `Regional Root this switch`)。name・VLANマッピングが完全に一致していてもrevisionが1つでも異なれば別リージョンとして扱われることを確認した。

**リビジョン番号の大小に関する注記**: MSTリージョンの一致条件は「リージョン名・リビジョン番号・VLANマッピングの3要素が完全に一致すること」であり、**リビジョン番号が大きい方が優先される・勝つ、という比較ロジックは存在しない**。本試験でもrevision 2のSW4が他を「上書き」するような挙動は一切見られず、単に別リージョンとして扱われただけであった(README「MSTリージョン一致条件」参照)。

復旧後(`reverted-*.txt`)、revision 1に戻し正常状態への復帰を確認した。

## 試験6: VLANマッピング不一致

出典: `outputs/vlan-mapping-mismatch/mismatch-*.txt`, `reverted-*.txt`

SW4のみ`instance 1 vlan 10` / `instance 2 vlan 20,30,40`に変更(name・revisionは共通設定のまま)した結果 (`mismatch-SW4-show-spanning-tree-mst-configuration.txt`):

```
Instance  Vlans mapped
1         10
2         20,30,40
```

これにより試験4・5と同様にSW4が単独リージョン化することを確認した。VLANマッピングのみが異なる場合も、name・revisionが一致していても別リージョンとして扱われることを実測確認した。

復旧後(`reverted-*.txt`)、`instance 1 vlan 10,20` / `instance 2 vlan 30,40`に戻し正常状態への復帰を確認した。

## 試験7: Rapid PVST+との比較

出典: `outputs/rapid-pvst-comparison/SW{1-4}-show-spanning-tree-summary.txt`, `SW{1-4}-show-spanning-tree.txt`, `SW{1-4}-post-revert-*.txt`

### Rapid PVST+移行後のSW1 `show spanning-tree summary`(抜粋)

```
Switch is in rapid-pvst mode
Root bridge for: VLAN0010, VLAN0020, VLAN0030, VLAN0040

Name                   Blocking Listening Learning Forwarding STP Active
---------------------- -------- --------- -------- ---------- ----------
VLAN0010                     0         0        0          2          2
VLAN0020                     0         0        0          2          2
VLAN0030                     0         0        0          2          2
VLAN0040                     0         0        0          2          2
4 vlans                      0         0        0          8          8
```

**観測された構造的な違い**:

| 項目 | MST | Rapid PVST+ (本ラボでの実測) |
|---|---|---|
| STPインスタンス数 | 3 (CIST + MST1 + MST2) | 4 (VLAN10/20/30/40 それぞれ独立) |
| ルートブリッジ | インスタンスごとに個別設定可能。本ラボではMST1=SW1、MST2=SW2の2つのルート | 本ラボでは per-VLAN priority を明示設定していないため、デフォルト優先度(32768)とMACアドレスのタイブレークにより**4 VLAN全てSW1がルート**になった(`Root bridge for: VLAN0010, VLAN0020, VLAN0030, VLAN0040`) |
| BPDU構造 | 1つのMSTP BPDU(CIST BPDU)にMレコード(M-Record)として各インスタンスの情報が多重化されて運ばれる | VLANごとに個別のPVST+ BPDU(802.1Qタグ付き、VLANごとのSTPインスタンス)が送信される |

**確認結果**: VLANごとに独立したSTPインスタンスが存在することを実測確認した。MSTでは2つのインスタンスに意図的にルートを分散できたのに対し、Rapid PVST+では(追加のper-VLAN priority設定をしない限り)全VLANが同一のルートブリッジに収束する、という設計上の違いを実測データから確認した。

**CPU/負荷比較についての方針(ユーザー指示に基づく)**: 本ラボはVLAN数が4つと少なく、MST(3インスタンス)とRapid PVST+(4インスタンス)のBPDU処理負荷の差は、この規模では有意な差として観測できる可能性が低い。実際、CMLのCPU使用率を比較する測定は実施していない。したがって本結果では「MSTの方がスケールする」という一般論の裏付けとしてSTPインスタンス数とBPDU構造の違いのみを事実として記録し、性能上の優劣について断定的な結論は出さない。

復旧後(`SW{1-4}-post-revert-mst-configuration.txt`, `SW{1-4}-post-revert-show-spanning-tree-mst.txt`)、`spanning-tree mst configuration`(name/revision/VLANマッピング)が保持されたまま`mst`モードに復帰し、試験1・2と同一のMST1=SW1ルート/MST2=SW2ルート構成に戻ったことを確認した。

## 最終確認(Phase 8)

出典: `outputs/baseline/SW{1-4}-final-show-spanning-tree-mst.txt`, `SW{1-4}-final-show-vlan-brief.txt`, `SW{1-4}-final-show-interfaces-status.txt`, `configs/SW{1-4}.cfg`

全4台について、試験1・2で確認したベースラインと同一のMST1/MST2構成(SW1がMST1ルート、SW2がMST2ルート、SW3–SW4リンクのブロック位置もMST1/MST2で異なる状態)に復帰していることを再確認した。各スイッチのEt0/0・Et0/1は`connected / trunk`状態、未使用のEt0/2・Et0/3は`disabled`(shutdown)状態であることを`show interfaces status`で確認した。

## 期待結果との差異

| 項目 | 期待結果 | 実測結果 | 差異 |
|---|---|---|---|
| MST1のルート | SW1 | SW1 | なし |
| MST2のルート | SW2 | SW2 | なし |
| MST1/MST2でブロック位置が異なる | 想定通り | SW3–SW4リンクでSW4側/SW3側とブロック位置が異なる | なし |
| `switchport trunk encapsulation dot1q` | 非対応なら省略 | **実際には投入・保持され動作した**(`show running-config interface`で確認) | 仕様は「非対応なら省略」だったが、本イメージ(ioll2-xe, IOS-XE 17.18)では対応していたため省略せず採用 |
| 試験3のping | 端末間ping(任意) | 端末が仕様にないため、VLAN10のSVIをスイッチに一時追加してping代用(ユーザー確認済み) | 仕様外の追加設定 |
| 試験3の収束時間 | (規定なし) | 約3.5〜6.2秒(STP状態)/約6.5秒(ping再開ベース) | ポーリング間隔起因の精度制約あり(上記参照) |
| リビジョン番号の優先順位 | (規定なし、誤解注意のみ指示) | 大小比較のロジックは存在しないことを実測で確認 | なし(指示通り） |
