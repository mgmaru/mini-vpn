# mini-vpn 実装ロードマップ

## 1. プロジェクト概要

`mini-vpn` は、VPNの仕組みを理解するために、最小構成から段階的に実装していく学習用プロジェクトです。

目的は、既存VPN製品をそのまま再実装することではありません。

このプロジェクトでは、

```text
IP
Routing
TUN
UDP
Encryption
Authentication
NAT
NAT Traversal
```

などの技術を、実際にコードを書きながら理解することを目標にします。

最終的には、2台の端末間で次のような通信を実現します。

```text
PC-A
VPN IP: 10.0.0.1
   │
   │ encrypted tunnel
   ▼
Internet / LAN
   │
   ▼
PC-B
VPN IP: 10.0.0.2
```

ユーザーから見ると、

```bash
ping 10.0.0.2
ssh user@10.0.0.2
```

のように、仮想IP経由で相手へアクセスできる状態を目指します。

---

# 2. 学習上のゴール

このプロジェクトの主な学習目標は次の通りです。

- VPNが「何を運んでいるのか」を理解する
- TUNデバイスの役割を理解する
- IPパケットをプログラムで扱う
- UDPを使ってIPパケットを遠隔ホストへ送る
- ルーティングとVPNの関係を理解する
- 暗号化されたトンネルの仕組みを理解する
- 認証と鍵交換の役割を理解する
- NATがVPN通信に与える影響を理解する
- NAT Traversalの考え方を理解する
- RelayとP2P直接接続の違いを理解する
- VPNの制御プレーンとデータプレーンの違いを理解する

---

# 3. このプロジェクトで作るもの

最終的には、概念的に次の構成を目指します。

```text
[PC-A]

Application
   │
   ▼
TCP / UDP
   │
   ▼
IP packet
Destination: 10.0.0.2
   │
   ▼
TUN
   │
   ▼
mini-vpn
   │
   ├─ packet read
   ├─ encryption
   ├─ authentication
   └─ UDP encapsulation
   │
   ▼
Internet
   │
   ▼
mini-vpn
   │
   ├─ UDP receive
   ├─ authentication
   ├─ decryption
   └─ packet write
   │
   ▼
TUN
   │
   ▼
10.0.0.2

[PC-B]
```

---

# 4. 最初に実装しないもの

学習用プロジェクトなので、最初からすべてを実装しません。

初期段階では次の機能は対象外とします。

- 多数のユーザー管理
- Web管理画面
- 高可用性
- 大規模クラスタ
- 商用VPN互換
- WireGuard互換
- OpenVPN互換
- IPv6完全対応
- モバイルOS対応
- 高度なNAT Traversal
- 本番向け鍵管理
- 自動証明書更新
- 帯域制御
- QoS
- 複雑なACL
- マルチテナント

最初は、

> 2台のLinuxホスト間で安全にIPパケットを運ぶ

ことに集中します。

---

# 5. 開発環境の想定

学習を簡単にするため、最初はLinuxを推奨します。

例：

```text
Host OS
  │
  ├─ VM-A
  │   └─ mini-vpn client
  │
  └─ VM-B
      └─ mini-vpn peer
```

または、

```text
PC-A
Linux VM

PC-B
Linux VM
```

でも構いません。

TUNデバイスやルーティング操作を行うため、Linux環境が扱いやすいです。

---

# 6. 使用言語

実装言語としてGoを想定します。

主に使う概念：

```text
net
net/netip
os
io
encoding/binary
crypto/*
context
goroutine
channel
```

VPNでは、

```text
受信
送信
再接続
keepalive
```

など複数処理を同時に扱うため、Goの並行処理との相性も良いです。

---

# 7. 実装ロードマップ全体

全体は次の順番で進めます。

```text
Phase 0
UDP通信確認
   ↓
Phase 1
TUNデバイス作成
   ↓
Phase 2
TUNからIPパケット取得
   ↓
Phase 3
UDPでパケット転送
   ↓
Phase 4
VPN IPとルーティング
   ↓
Phase 5
双方向トンネル
   ↓
Phase 6
独自プロトコル
   ↓
Phase 7
暗号化
   ↓
Phase 8
認証
   ↓
Phase 9
鍵交換
   ↓
Phase 10
Keepalive / 再接続
   ↓
Phase 11
複数Peer
   ↓
Phase 12
NAT Traversal
   ↓
Phase 13
Relay
   ↓
Phase 14
運用・観測機能
```

---

# 8. Phase 0：UDP通信を確認する

## 目的

VPNを作る前に、2台のホスト間でUDP通信できることを確認します。

```text
PC-A
   │
   │ UDP
   ▼
PC-B
```

実装するもの：

- UDP sender
- UDP receiver
- 指定IP・ポートへの送信
- 受信データの表示

---

## 完了条件

PC-Aから、

```text
hello
```

を送り、

PC-B側で、

```text
hello
```

を受信できること。

---

# 9. Phase 1：TUNデバイスを作る

## 目的

VPNで使う仮想ネットワークインターフェースを理解します。

概念：

```text
Application
   │
   ▼
Kernel
   │
   ▼
TUN
   │
   ▼
mini-vpn
```

TUNデバイスは、EthernetフレームではなくIPパケットを扱う仮想インターフェースです。

---

## 実装するもの

- TUNデバイスの作成
- TUNインターフェース名の取得
- TUNへのIP割り当て
- インターフェースUP

例：

```text
tun0
10.0.0.1/24
```

---

## 完了条件

OS上で、

```bash
ip addr
```

などを確認したとき、

```text
tun0
10.0.0.1/24
```

が存在すること。

---

# 10. Phase 2：TUNからIPパケットを読む

## 目的

OSが仮想IP宛に生成したIPパケットを、プログラムから取得します。

```text
ping 10.0.0.2
   │
   ▼
Kernel
   │
   ▼
tun0
   │
   ▼
mini-vpn
```

---

## 実装するもの

- TUNからのread
- パケットサイズ表示
- IPv4ヘッダの簡単な解析
- source / destination IP表示

---

## 完了条件

例えば、

```bash
ping 10.0.0.2
```

を実行したとき、mini-vpn側で、

```text
src=10.0.0.1
dst=10.0.0.2
protocol=ICMP
```

などを確認できること。

---

# 11. Phase 3：IPパケットをUDPで送る

## 目的

TUNから取得したIPパケットを、そのまま相手ホストへ送ります。

```text
TUN
 ↓
IP packet
 ↓
mini-vpn
 ↓
UDP
 ↓
Network
 ↓
Peer
```

---

## 実装するもの

- TUNからread
- UDP送信
- UDP受信
- 受信したデータの確認

この段階では暗号化しません。

---

# 12. Phase 4：受信パケットを相手のTUNへ戻す

## 目的

UDPで受け取ったIPパケットを、相手側のTUNデバイスへ書き込みます。

```text
PC-A

tun0
 ↓
mini-vpn
 ↓
UDP
 ↓

Network

 ↓
UDP
 ↓
mini-vpn
 ↓
tun0

PC-B
```

---

## 実装するもの

- UDP receive
- TUN write
- パケット長チェック
- 基本的なエラー処理

---

## 完了条件

PC-Aから、

```bash
ping 10.0.0.2
```

が成功すること。

ここで初めて、

> 仮想IP同士で通信できる最小VPN

が完成します。

---

# 13. Phase 5：双方向トンネル

## 目的

PC-A→PC-Bだけでなく、

```text
PC-B → PC-A
```

も同時に通信できるようにします。

```text
PC-A
  ⇄
PC-B
```

---

## 実装するもの

- TUN read loop
- UDP receive loop
- goroutineによる並行処理
- graceful shutdown

---

## 完了条件

両方から、

```bash
ping
```

が成功すること。

---

# 14. Phase 6：独自VPNプロトコルを定義する

## 目的

UDPで生のIPパケットを送るだけでは拡張しにくいため、簡単なメッセージ形式を定義します。

例：

```text
VERSION
TYPE
PEER_ID
SEQUENCE
PAYLOAD_LENGTH
PAYLOAD
```

---

## メッセージ種別例

```text
DATA
HELLO
AUTH
KEEPALIVE
ERROR
```

---

## 完了条件

送受信するUDPパケットについて、

```text
これはVPN DATA
これはKEEPALIVE
```

と区別できること。

---

# 15. Phase 7：暗号化

## 目的

VPN内のIPパケットを平文で送らないようにします。

```text
Original IP packet
        ↓
Encrypt
        ↓
Encrypted payload
        ↓
UDP
```

---

## 重要

暗号アルゴリズムそのものは自作しません。

既存の安全な暗号ライブラリを使います。

---

## 実装するもの

- 共通鍵による暗号化
- nonce管理
- 認証付き暗号
- 改ざん検出

---

## 完了条件

Wiresharkなどで見ても、

```text
元のIPパケット内容
```

が平文で見えないこと。

---

# 16. Phase 8：Peer認証

## 目的

誰でもUDPパケットを送ればVPNに参加できる状態を防ぎます。

```text
Peer A
   │
   │ authenticate
   ▼
Peer B
```

---

## 実装候補

学習初期：

```text
事前共有鍵
PSK
```

その後：

```text
公開鍵
```

へ発展させます。

---

## 完了条件

正しい鍵を持つPeerだけがDATA通信できること。

---

# 17. Phase 9：鍵交換

## 目的

暗号鍵そのものをネットワーク上へ直接送らず、安全にセッション鍵を作ります。

概念：

```text
Peer A
  │
  │ Key Exchange
  ▼
Peer B

↓
双方が同じSession Keyを導出
```

---

## 実装するもの

- 鍵ペア
- 公開鍵交換
- セッション鍵導出
- 鍵更新の基本設計

---

# 18. Phase 10：Keepaliveと再接続

## 目的

VPNは長時間動作するため、

```text
Peerが生きているか
NAT mappingが残っているか
```

を確認する必要があります。

---

## 実装するもの

- keepalive packet
- timeout
- reconnect
- peer state

状態例：

```text
DISCONNECTED
CONNECTING
CONNECTED
TIMEOUT
```

---

# 19. Phase 11：複数Peer対応

## 目的

2台だけでなく、複数端末を扱います。

```text
       PC-B
        ▲
        │
PC-A ───┼─── PC-C
        │
        ▼
       PC-D
```

---

## 必要になるもの

- Peer ID
- VPN IP → Peer対応表
- Endpoint管理
- Peerごとの暗号鍵
- Peerごとの状態管理

---

## 例

```text
10.0.0.2 → Peer-B
10.0.0.3 → Peer-C
10.0.0.4 → Peer-D
```

---

# 20. Phase 12：NAT Traversal

## 目的

手動ポートフォワードなしで、NAT内のPeer同士が直接通信できる可能性を作ります。

ここでは、

```text
STUN
UDP hole punching
```

を学びます。

---

## 構成

```text
PC-A
   │
   ▼
NAT-A
   │
   ▼
Internet
   ▲
   │
NAT-B
   ▲
   │
PC-B
```

---

## Step 12-1：外部Endpoint取得

STUNの考え方を使い、

```text
自分は外部から
203.0.113.10:53124
に見えている
```

という情報を取得します。

---

## Step 12-2：Endpoint交換

Peer同士で、

```text
PC-A:
203.0.113.10:53124

PC-B:
198.51.100.20:48763
```

を共有します。

---

## Step 12-3：UDP hole punching

双方が相手EndpointへUDPを送ります。

```text
PC-A
   │
   ▼
198.51.100.20:48763

PC-B
   │
   ▼
203.0.113.10:53124
```

---

## 完了条件

対応可能なNAT環境で、手動ポートフォワードなしにP2P通信が成立すること。

---

# 21. Phase 13：Relay対応

## 目的

NAT traversalが失敗した場合にも通信できるようにします。

```text
PC-A
  │
  ▼
Relay
  │
  ▼
PC-B
```

---

## 接続戦略

```text
Direct connection
      ↓
成功 → P2P

失敗
      ↓
Relay
```

---

## 学習ポイント

- P2PとRelayの違い
- データプレーン
- fallback
- latency
- bandwidth

---

# 22. Phase 14：制御プレーンを分離する

VPNが複雑になると、

```text
通信データ
```

と、

```text
Peer情報や鍵情報
```

を分離したくなります。

---

## Data Plane

実際のIPパケットを運ぶ。

```text
PC-A ←──── encrypted packets ────→ PC-B
```

---

## Control Plane

次の情報を管理する。

```text
Peer identity
VPN IP
Public key
Endpoint
Connection status
```

---

## 目標

```text
Control Server
      │
      ├─ Peer A
      ├─ Peer B
      └─ Peer C
```

で接続情報だけを管理し、

可能ならData Planeは、

```text
Peer A ←────→ Peer B
```

と直接通信します。

---

# 23. Phase 15：観測・デバッグ機能

VPNはネットワーク障害時の原因特定が難しいため、観測性が重要です。

実装候補：

- 接続状態表示
- 現在のEndpoint
- 送信パケット数
- 受信パケット数
- 転送バイト数
- Drop数
- RTT
- Keepalive状況
- 暗号化セッション状態

---

## CLI例

```text
mini-vpn status
```

出力イメージ：

```text
Peer: node-b
VPN IP: 10.0.0.2
Endpoint: 198.51.100.20:48763
Connection: direct
State: connected
TX: 1240 packets
RX: 1189 packets
```

---

# 24. MTUへの対応

VPNでは、元のIPパケットをさらにUDPなどで包むため、パケットサイズが増えます。

```text
Original packet
+
VPN header
+
UDP header
+
IP header
```

そのためMTUを考慮する必要があります。

---

## 学習内容

- MTU
- Fragmentation
- Path MTU
- VPN overhead

---

## 対応候補

- TUN MTUを小さく設定
- oversized packetの検出
- ログ表示

---

# 25. IP Forwarding

将来的に、

```text
VPN Peer
   ↓
その先のLAN
```

までアクセスしたい場合、IP forwardingが必要になります。

例：

```text
PC-A
10.0.0.1
   │
   ▼
VPN Gateway
10.0.0.2
192.168.50.1
   │
   ▼
192.168.50.100
```

ここまで実装すると、

```text
Host-to-Host VPN
```

から、

```text
Site-to-Site VPN
```

へ理解を広げられます。

---

# 26. Site-to-Siteへの発展

将来的な発展として、

```text
Network A
192.168.1.0/24
       │
       ▼
VPN Gateway A
       │
       │ Tunnel
       ▼
VPN Gateway B
       │
       ▼
Network B
192.168.50.0/24
```

を実装できます。

---

# 27. セキュリティ上の注意

`mini-vpn` は学習用です。

本番環境で利用することを前提にはしません。

特に次の点に注意します。

- 独自暗号を作らない
- 秘密鍵をログに出さない
- nonce再利用を避ける
- 認証前のDATAを信用しない
- パケット長を検証する
- 不正な入力でpanicしない
- replay対策を考える
- 鍵ローテーションを考える
- DoS耐性を意識する

---

# 28. 予定ディレクトリ構成

初期案：

```text
mini-vpn/
├── cmd/
│   └── mini-vpn/
│       └── main.go
│
├── internal/
│   ├── tun/
│   │   └── ...
│   │
│   ├── tunnel/
│   │   └── ...
│   │
│   ├── protocol/
│   │   └── ...
│   │
│   ├── crypto/
│   │   └── ...
│   │
│   ├── peer/
│   │   └── ...
│   │
│   ├── routing/
│   │   └── ...
│   │
│   └── nat/
│       └── ...
│
├── docs/
│   ├── architecture.md
│   ├── protocol.md
│   └── roadmap.md
│
├── go.mod
└── README.md
```

ただし、最初から細かく分けすぎる必要はありません。

機能が増えて責務が明確になった段階で分割します。

---

# 29. 最小構成時のディレクトリ

最初は次の程度でも十分です。

```text
mini-vpn/
├── main.go
├── tun.go
├── udp.go
├── packet.go
└── go.mod
```

実装が進んだら分割します。

---

# 30. 最初のMVP

最初の完成目標は、

> PC-AとPC-Bの仮想IP間でpingを通す

ことです。

構成：

```text
PC-A
tun0: 10.0.0.1
   │
   ▼
mini-vpn
   │
   │ UDP
   ▼
mini-vpn
   │
   ▼
tun0: 10.0.0.2
PC-B
```

---

## MVPでは実装しないもの

- 暗号化
- 認証
- NAT traversal
- Relay
- 複数Peer
- Control Server

まずは、

```text
IP packet
↓
TUN
↓
UDP
↓
TUN
```

だけを完成させます。

---

# 31. MVP後の優先順位

MVP完成後：

```text
1. 安定した双方向通信
2. 独自プロトコル
3. 暗号化
4. 認証
5. Keepalive
6. Peer管理
7. NAT traversal
8. Relay
9. Control Plane
```

の順で進めます。

---

# 32. テスト方針

## Unit Test

対象：

```text
Protocol encode/decode
Packet validation
Crypto wrapper
Peer table
Routing lookup
```

---

## Integration Test

VMを2台使い、

```text
VM-A
10.0.0.1

VM-B
10.0.0.2
```

で通信を確認します。

確認項目：

- ping
- TCP
- UDP
- SSH
- 切断時
- 再接続時
- 不正パケット

---

# 33. パケット観察

学習では、実装するだけでなくパケットを観察します。

利用候補：

```text
tcpdump
Wireshark
ip
ss
ping
traceroute
```

例えば、

```bash
tcpdump -i tun0
```

で仮想ネットワーク上のパケットを確認します。

また、

```bash
tcpdump -i eth0 udp
```

でVPN外側のUDP通信を確認します。

---

# 34. 観察したい重要ポイント

VPN実装中は、

```text
TUN側では元のIP packet
```

が見え、

```text
物理NIC側ではVPN用UDP
```

が見えることを確認します。

```text
tun0

10.0.0.1 → 10.0.0.2

        ↓ encapsulate

eth0

203.x.x.x:port → 198.x.x.x:port
```

この違いを自分で観察することが、VPN理解に非常に重要です。

---

# 35. 学習上のマイルストーン

## Milestone 1

```text
UDPで2台通信
```

## Milestone 2

```text
TUNからIP packet取得
```

## Milestone 3

```text
TUN → UDP → TUN
```

## Milestone 4

```text
VPN IP同士でping成功
```

## Milestone 5

```text
暗号化VPN
```

## Milestone 6

```text
認証付きVPN
```

## Milestone 7

```text
複数Peer
```

## Milestone 8

```text
NAT Traversal
```

## Milestone 9

```text
Relay fallback
```

## Milestone 10

```text
Control Plane分離
```

---

# 36. 最終的に目指すアーキテクチャ

```text
                  Control Server
                 /      |       \
                /       |        \
            Peer-A    Peer-B    Peer-C
               \        |        /
                \       |       /
                 \      |      /

             Data Plane

Peer-A ←──── encrypted P2P ────→ Peer-B
   │
   └──── Relay if P2P fails ────┘
```

各Peer内部：

```text
Application
   │
   ▼
Kernel Routing
   │
   ▼
TUN
   │
   ▼
mini-vpn
   │
   ├─ Routing
   ├─ Peer lookup
   ├─ Encryption
   ├─ Protocol
   └─ UDP Transport
   │
   ▼
Internet
```

---

# 37. このプロジェクトで得たい理解

最終的には、VPNを単なる便利ツールではなく、

```text
Virtual Interface
      ↓
IP packet
      ↓
Routing
      ↓
Encryption
      ↓
Encapsulation
      ↓
UDP
      ↓
Internet
      ↓
UDP
      ↓
Decryption
      ↓
IP packet
      ↓
Virtual Interface
```

という処理として説明できることを目指します。

---

# 38. 最重要方針

`mini-vpn` の実装では、

> 最初から完成形を作らない

ことを重視します。

まず、

```text
TUN
↓
UDP
↓
TUN
```

を実装します。

その後、

```text
暗号化
認証
Peer管理
NAT Traversal
Relay
Control Plane
```

を1つずつ追加します。

こうすることで、

> それぞれの機能が「なぜ必要なのか」

を理解しながら実装できます。

---

# 39. まとめ

`mini-vpn` の最初のゴールは非常にシンプルです。

```text
PC-A
10.0.0.1
   │
   │ VPN Tunnel
   ▼
PC-B
10.0.0.2
```

この状態で、

```bash
ping 10.0.0.2
```

を成功させます。

そこから段階的に、

```text
UDP
↓
TUN
↓
Routing
↓
Protocol
↓
Encryption
↓
Authentication
↓
Key Exchange
↓
Peer Management
↓
NAT Traversal
↓
Relay
↓
Control Plane
```

へ進めます。

最終的な目的は、VPN製品を作ることそのものではなく、

> **VPNという「魔法」に見える技術を、自分のコードで1つずつ分解して理解すること**

です。
