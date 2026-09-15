# トポロジ詳細

## 物理構成

```mermaid
graph TD
    SW1["SW1<br/>MST1 root primary<br/>MST2 root secondary"]
    SW2["SW2<br/>MST1 root secondary<br/>MST2 root primary"]
    SW3["SW3<br/>non-root"]
    SW4["SW4<br/>non-root"]

    SW1 ---|"Et0/0 -- Et0/0<br/>trunk 10,20,30,40"| SW2
    SW1 ---|"Et0/1 -- Et0/0<br/>trunk 10,20,30,40"| SW3
    SW2 ---|"Et0/1 -- Et0/0<br/>trunk 10,20,30,40"| SW4
    SW3 ---|"Et0/1 -- Et0/1<br/>trunk 10,20,30,40"| SW4
```

インターフェース番号はすべてCML API (`GET /api/v0/labs/{lab}/nodes/{node}/interfaces?data=true`) で実際に払い出された値であり、推測していない。`topology.yaml`に機械可読な形式で記録。

## ベースラインのMST1/MST2フォワーディングツリー(実測値、`outputs/baseline/`のshow出力に基づく)

```mermaid
graph TD
    subgraph MST1["MST1 (VLAN 10,20) - Root: SW1"]
        A1[SW1] -->|Desg/FWD| A2[SW2]
        A1 -->|Desg/FWD| A3[SW3]
        A2 -->|Desg/FWD| A4[SW4]
        A3 -.->|Altn/BLK<br/>SW3側| A4
    end
```

```mermaid
graph TD
    subgraph MST2["MST2 (VLAN 30,40) - Root: SW2"]
        B2[SW2] -->|Desg/FWD| B1[SW1]
        B2 -->|Desg/FWD| B4[SW4]
        B1 -->|Desg/FWD| B3[SW3]
        B3 -.->|Altn/BLK<br/>SW3側| B4
    end
```

MST1ではSW3–SW4リンクがSW4側でブロックされ、MST2では同じリンクがSW3側でブロックされる。ブロック位置が異なるため、MST1とMST2で実際の転送パスが分かれ、負荷分散が成立している(試験2で実測確認)。

## ノード・インターフェース対応表

| ノード | Et0/0 | Et0/1 | Et0/2 | Et0/3 |
|---|---|---|---|---|
| SW1 | trunk → SW2 Et0/0 | trunk → SW3 Et0/0 | shutdown(未使用) | shutdown(未使用) |
| SW2 | trunk → SW1 Et0/0 | trunk → SW4 Et0/0 | shutdown(未使用) | shutdown(未使用) |
| SW3 | trunk → SW1 Et0/1 | trunk → SW4 Et0/1 | shutdown(未使用) | shutdown(未使用) |
| SW4 | trunk → SW2 Et0/1 | trunk → SW3 Et0/1 | shutdown(未使用) | shutdown(未使用) |
