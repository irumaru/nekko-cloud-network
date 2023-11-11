# 概要
このドキュメントはnekko-cloudのリージョンネットワークの構築に使用するVyOSの構築方法についてまとめたものである。

# 目次
- 概要
- 目次
- 必要な情報
- ハードウェア要求
- VyOSのインストール
- VyOSの基本操作
- VyOSの設定

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

## インターフェスと接続先
eth0 上位ルーター(インターネット接続を提供)への接続用  
eth1 リージョン内ネットワークへの接続用  
(仮想インターフェス)wg0 リージョン間VPN接続用  

# VyOSのインストール
- インストールディスクを起動する  
- イメージをシステムディスクに書き込む  
[https://docs.vyos.io/en/equuleus/installation/install.html?highlight=requirment#permanent-installation](https://docs.vyos.io/en/equuleus/installation/install.html?highlight=requirment#permanent-installation)

初期ユーザー名: vyos  
初期パスワード: vyos  

# VyOSの基本操作
## 設定モードへ入る
```
show configuration commands
```
## 設定を一時保存(再起動すると消える)
```
commit
```
## 設定を永続保存
```
save
```
## 設定モードから出る
```
exit
```

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

### DNSの設定
```
set system name-server '2606:4700:4700::1111'
set system name-server '2606:4700:4700::1001'
```
[https://1.1.1.1/ja-jp/](https://1.1.1.1/ja-jp/)

### DNSフォワーディングの設定
```
set service dns forwarding allow-from '[リージョンのネットワークアドレス]/16'
set service dns forwarding listen-address '[eth1のIPアドレス]'
set service dns forwarding name-server '2606:4700:4700::1111'
set service dns forwarding name-server '2606:4700:4700::1001'
set service dns forwarding no-serve-rfc1918
```

## リージョン内ネットワークの設定
```
set interfaces ethernet eth1 address '[eth1のIPアドレス]/24'
```

## リージョン間接続の設定(拠点間接続数分)

### Wireguard接続の削除
以前にWireguardの接続を行っていた場合は、wireguard接続を削除してください。  
理由: フレッツ網内でWireguardのレスポンドパケット消失が発生。原因が不明で未解決のため。  
[これに関する議論](https://github.com/irumaru/nekko-cloud-network/issues/1)  
```
delete interface wireguard [インターフェス名]
```
インターフェス名は```wg0```など。

### IPIP6トンネル用外側インターフェスを作成
IPIP6トンネルは、IPv6パケットの中に直接オーバーレイネットワークのIPv4パケットを格納するため、1つの外側IPv6インターフェス(IPv6アドレス)を占有する。  
そのため、IPIP6トンネルごとに専用IPv6アドレスを作成する必要。  
```
set interfaces ethernet eth0 address [このルータの拠点間接続用一意なIPv6アドレス]/64
```
例
```
set interfaces ethernet eth0 address '2401:2500:10a:103a::1000/64'
```

### IPIP6トンネルの作成
```
set interfaces tunnel tun0 address '[tun0(VPN用のこのルータのIF)のIPアドレス]/24'
set interfaces tunnel tun0 encapsulation 'ipip6'
set interfaces tunnel tun0 mtu '1460'
set interfaces tunnel tun0 remote '[接続先リージョンのルーターのIPv6アドレス]'
set interfaces tunnel tun0 source-address '[このルータの一意なIPv6アドレス]'
```
例
```
set interfaces tunnel tun0 address '172.16.3.2/24'
set interfaces tunnel tun0 encapsulation 'ipip6'
set interfaces tunnel tun0 mtu '1460'
set interfaces tunnel tun0 remote '2001:f71:b200:a51:98b8:11ff:fe5d:cfe0'
set interfaces tunnel tun0 source-address '2401:2500:10a:103a::36'
```

### 動的ルーティングの設定
```
set protocols ospf area 0 network '[tun0のネットワークアドレス]/24'
set protocols ospf area 0 network '[eth1のネットワークアドレス]/24'
set protocols ospf parameters router-id '[eth1のIPv4アドレス]'
```
