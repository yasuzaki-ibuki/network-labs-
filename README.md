# LAG（mode on）の検証

Packet Tracer で LAG を `mode on` で組み、片側のメンバーポートに設定ミスがあったときの動きを確認しました。

## 構成

![構成図](images/topology.png)

- SW1 Fa0/2〜4 ⇔ SW2 Fa0/2〜4 を LAG（Po1、mode on）
- PC1（.1）→ SW1 → SW2 → R1（.254）
- 検証時の状態は `lag_mode_on.pkt` に保存（Packet Tracer で開けます）

```
Switch(config)# interface range fastEthernet 0/2 - 4
Switch(config-if-range)# channel-group 1 mode on
```

## 学んだこと

### 1. LAG の振り分けは「送り主ごと」

```
Switch# show etherchannel load-balance
EtherChannel Load-Balancing Operational State (src-mac):
```

1つのデータを3本に分けて送っているのではなく、**送信元MACアドレスごとに使うリンクを1本決めている**。
1つのデータを分けると、リンクごとの速度差で順番が入れ替わってしまうため。

そのため、PC1台だけで試すと「全部通る」か「全部落ちる」のどちらかになる。

### 2. LAG で通信が消える例（VLAN の設定ミス）

SW2 の Fa0/4 だけ束から外し、別の VLAN（99）にした。
実際の現場で起こる「片側だけ `channel-group` を入れ忘れ、そのポートが別の設定のまま」という状態を、VLAN を変えることで再現している。

```
SW2(config)# interface fastEthernet 0/4
SW2(config-if)# no channel-group 1
SW2(config-if)# switchport access vlan 99
```

結果

- SW1 の `show etherchannel summary` では、Fa0/2〜4 の3本が `(P)`（使用中）
- SW2 では Fa0/2〜3 の2本だけ
- 同じケーブルを、SW1 は3本・SW2 は2本の束として数えている
- SW1 は相手のミスに気づかず、3本に振り分け続ける
- Fa0/4 に振り分けられた送り主の通信は、SW2 で VLAN 99 に入って R1 に届かない
- CDP の警告（`NATIVE_VLAN_MISMATCH`）は出たが、警告だけで通信の動きは変わらない

SW1

![SW1 etherchannel](images/sw1_etherchannel.png)

SW2

![SW2 etherchannel](images/sw2_etherchannel.png)

`mode on` は相手と何もやり取りしないので、片側だけのミスに気づけない。
メンバー追加や機器交換で、片側だけ `channel-group` を入れ忘れると起こり得る。

### 3. メディアコンバーター（光ファイバーの変換器）

距離がある場所では、LANケーブルと光ファイバーを変換する機器を間に挟むことがある。

```
SW1 ─(LANケーブル)─ 変換器 ═(光ファイバー)═ 変換器 ─(LANケーブル)─ SW2
```

間の光ファイバーが切れても、スイッチから見ると目の前の変換器とはつながっているので、**リンクは up のまま**。
`mode on` はリンクの up/down しか見ていないため、この障害に気づけない。

## まとめ

- `mode on` は設定ミスや「リンクは up なのに通らない」障害を検知できない
- LACP は相手と制御パケットをやり取りするので、届かなくなったポートを自動で束から外せる
- LAG を触ったあとは、**両側で** `show etherchannel summary` を確認する
