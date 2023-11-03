# 必要な情報
## 事前に決める又は調査する情報
- このルーターに設定するパスワード
- このルーターのWAN側(ローカルIPv4)のIPv4アドレス
- 上位のルーターのIPv4アドレス(デフォルトゲートウェイ)

## 資料から読み取り準備する情報
- このルータのホスト名
- eth1のネットワークアドレスとIPアドレス(IPv4)
- wg0(VPN用のこのルータのIF)と、wg0のネットワークアドレスのIPアドレス
- 接続先のリージョン名
- 接続先リージョンのルーターのIPv6アドレス
- 接続先リージョンのルーターの公開鍵

# ハードウェア要求
## 最小要件
RAM 512MiB  
SSD 2GiB  

## 設定値
RAM 1GiB  
CPU 2Core  
SSD 2GiB  

# VyOSのインストール
- インストールディスクを起動する
- イメージをシステムディスクに書き込む
[https://docs.vyos.io/en/equuleus/installation/install.html?highlight=requirment#permanent-installation](https://docs.vyos.io/en/equuleus/installation/install.html?highlight=requirment#permanent-installation)

# VyOSの設定
```[名前]```の部分は適切な値に変更する必要があります。

## ホストに関する初期設定
### パスワードの設定
```
set system login user vyos authentication plaintext-password [このルータのパスワード]
```
### sshの有効化
```
set service ssh port '22'
```
### ホスト名の設定
```
set system host-name '[このルータのホスト名]'
```

## インターネット接続の設定
### インターフェスのアドレス設定
```
set interfaces ethernet eth0 address '[eth0(WAN)のIPアドレス]/24'
set interfaces ethernet eth0 ipv6 address autoconf
```
IPv4固定  
IPv6は上位ルータからのRAにより自動設定  

### ゲートウェイの設定
```
set protocols static route 0.0.0.0/0 next-hop [上位ルーターのIPv4アドレス]
```
IPv4は上位ルーターをデフォルトゲートウェイとして設定  
IPv6は上位ルータからのRAにより自動設定  

### NATの設定
```
set nat source rule 100 outbound-interface 'eth0'
set nat source rule 100 source address '[eth1のネットワークアドレス]/24'
set nat source rule 100 translation address 'masquerade'
```
eth1のIPアドレスは各リージョンのローカルIPアドレスを設定

## リージョン内ネットワークの設定
```
set interfaces ethernet eth1 address '[eth1のIPアドレス]/24'
```

## リージョン間接続の設定(拠点間接続数分)
### VPN鍵の作成
```
generate wireguard named-keypairs nclab
```
### VPN接続の設定
```
set interfaces wireguard wg0 address '[wg0(VPN用のこのルータのIF)のIPアドレス]/24'
set interfaces wireguard wg0 description '[接続先のリージョン名]'
set interfaces wireguard wg0 peer to-wg1 address '[接続先リージョンのルーターのIPv6アドレス]'
set interfaces wireguard wg0 peer to-wg1 allowed-ips '172.16.0.0/20'
set interfaces wireguard wg0 peer to-wg1 allowed-ips '224.0.0.5/32'
set interfaces wireguard wg0 peer to-wg1 allowed-ips '224.0.0.6/32'
set interfaces wireguard wg0 peer to-wg1 port '51820'
set interfaces wireguard wg0 peer to-wg1 pubkey '[接続先リージョンのルーターの公開鍵]'
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 private-key 'nclab'
set protocols static interface-route 172.16.0.0/24 next-hop-interface wg0 disable
```
### 動的ルーティングの設定
```
set protocols ospf area 0 network '[wg0のネットワークアドレス]/24'
set protocols ospf area 0 network '[eth1のネットワークアドレス]/24'
set protocols ospf parameters router-id '[eth1のIPv4アドレス]'
```
