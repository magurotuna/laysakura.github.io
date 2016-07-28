---
title: 'Amazon EC2でクライアント証明書認証のVPNを構築'
id: 'cir0gtu71000kibiu1skemk6c'
tags:
  - テクノロジー
  - Linuxオペレーション
  - セキュリティ
  - VPN
date: 2016-07-24 19:30:00
---

<img src="/img/2016/07-14-structure.png" alt="構成図" width="auto" height="auto">

高いセキュリティが要求されるLAN内のサーバに自宅などのリモートからSSHログインしたい場合、VPNが有効です。
VPNを使うと、リモートからLAN内に仮想的に参加することができるので、プライベートIPアドレスのみが割り振られたサーバにもSSHログインできるようになります。

VPNには色々な認証方式がありますが、SSL-VPNプロトコルでクライアント証明書方式の認証を行えば、誰がいつVPNに参加したのかを高い信頼度でログに記録することができます。

本記事では、OpenVPNを使ったSSL-VPNの構築方法をAmazon EC2での構築も踏まえ詳しく紹介します。

<!-- more -->

## 目的: プライベートネットワークに閉じた環境にリモートアクセスしたい

社内に置かれたサーバは、基本的にはインターネットからのアクセスをできないようにプライベートネットワークに置くことが多いかと思います。
「WebサーバはHTTP(S)のTCPポート80番(443番)だけをインターネットに開放する」などはありますが、そういうものを除けば各サーバにはプライベートIPアドレスのみを割り当て、同じLAN内のマシンのみからアクセスされるようにするのがセキュリティ上望ましいでしょう。

一方で、LAN内からしかアクセスできないサーバにもリモートからアクセスしたい場合があります。
管理者が自宅から緊急対応をする場合などですね。

このようなケースでは、物理的には別々のLANに属していても(例: 社内LANと管理者の自宅のLAN)、仮想的に同一のLANに所属させる **VPN** (Virtual Private Network) が有効です。
管理者のマシンでVPNに参加することで、管理者はプライベートIPアドレスのみが割り当てられたLAN内のサーバにアクセスすることができます。

ただし、誰でもVPNに参加できてしまったら、せっかくプライベートネットワークに閉じたサーバ群も格好の攻撃の的となってしまいます。
**クライアント証明書** を使うことで、VPNに接続することのできる人を限定し、また誰がいつ接続したかを記録することができます。

この記事では、
- VPNを構築し、物理的なLANの外にあるマシンを仮想的にLANに参加させる方法
- VPNに接続できるクライアントを認証により限定し、どのクライアントがVPNに接続したかを記録できる方法
- AWSを使ってVPNサーバと踏み台サーバを構築し、プライベートIPアドレスのみが割り振られた踏み台サーバに対してVPN経由でSSHログインする実践

について解説します。

構築や動作確認に利用するコマンドが出てきますが、サーバサイドはLinux (Debian, Ubuntu系統)、クライアントサイドはMac OS Xを想定しています。

<!-- toc -->

## クイックスタート: 最初のVPN接続

まずはじめに、サーバ・クライアント構成でシンプルなVPNを構築します。
VPNの機能は、 [OpenVPN](https://openvpn.net/) で提供します。
ネットワークスイッチやルータでVPNの機能が使える製品もありますが、本記事では普通のLinuxサーバ上でVPNを使うための方法を記載します。

Linuxサーバはroot権限が取れればどんなものでも構いませんが、Amazon EC2でゼロから構築する場合は先に [Amazon EC2でサンプル環境を構築](#amazon-ec2でサンプル環境を構築) から読み進めても良いでしょう。


### VPNサーバを立ち上げる

Linuxサーバ上での作業です。
まずはOpenVPNをインストールします。

```bash OpenVPNのインストール
$ sudo apt-get install openvpn
```

この時点ではまだOpenVPNのデーモンは立ち上がっていません。
"openvpn" パッケージには各種のサンプルファイルが含まれているので、OpenVPNサーバの立ち上げに必要な設定ファイル・証明書・秘密鍵を一箇所にまとめておきましょう。

```bash 設定ファイル・証明書・秘密鍵をコピー
$ mkdir -p ~/openvpn-test/server
$ cd ~/openvpn-test/server/
 
## 設定ファイルのコピー
$ zcat /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > server.conf
$ zcat /usr/share/doc/openvpn/examples/sample-keys/server.crt.gz > server.crt
$ cp /usr/share/doc/openvpn/examples/sample-keys/{ca.crt,server.key,dh1024.pem} .
$ ls
ca.crt  dh1024.pem  server.conf  server.crt  server.key
```

この時点でOpenVPNサーバを立ち上げることができます。

```bash OpenVPNサーバの起動
$ sudo openvpn server.conf
...
Thu Jul 21 23:05:11 2016 Initialization Sequence Completed
```

現時点の設定ファイルの内容は、以下のようになっている想定です。
(空行、コメント行は除く)

```config server.conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh1024.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 3
```

UDPの1194番ポートを使用していることに注意してください。
VPN接続が確立できないよくある理由の一つに、VPNで利用しているポートがファイアーウォールで遮断されていることがあります。
予め、VPN用ポートが開放できているか確認しておきましょう。

```bash サーバ側でUDP 1194番ポートを待ち受ける
$ nc -l -u -p 1194
```

```bash クライアント側でUDP 1194番ポートへメッセージを送る
$ nc -u your-vpn-server.com 1194
hello
```

ポート開放がうまく行っていれば、サーバ側で "hello" と出力されます。

失敗した場合、VPNサーバ側のファイアーウォールを見なおしてみてください。
EC2を利用している場合、後述の [セキュリティグループの作成](#セキュリティグループの作成) でUDP 1194番ポートを開放しています。


### VPNクライアントを設定

Macでの作業です。
Mac用のVPNクライアントのソフトウェアとして、 [Tunnelblick](https://tunnelblick.net/) を使います。
OpenVPNのGUIクライアントで、OpenVPN形式の設定ファイルが使えるのが便利です。
以下、Tunnelblickがインストールされた前提で進めます。

クライアント側でも設定ファイル・証明書・秘密鍵が必要になるので、OpenVPNのサンプルファイルをクライアント上に配置します。

```bash [サーバ側作業] クライアント用の設定ファイル・証明書・秘密鍵をコピー
$ mkdir -p ~/openvpn-test/client
$ cd ~/openvpn-test/client/
 
## 設定ファイルのコピー
$ cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf .
$ zcat /usr/share/doc/openvpn/examples/sample-keys/client.crt.gz > client.crt
$ cp /usr/share/doc/openvpn/examples/sample-keys/{ca.crt,client.key} .
$ ls
ca.crt  client.conf  client.crt  client.key
```

`scp` などの手段で、Mac上に上記ファイルを配置します。

```bash サーバからクライアント用の設定ファイル・証明書・秘密鍵を取得
$ scp -r you@your-server:~/openvpn-test/client .
 
$ cd client/
$ ls
ca.crt      client.conf client.crt  client.key
```

クライアントの設定ファイルは現時点で以下のようになっている想定です。

```config client.conf
client
dev tun
proto udp
remote my-server-1 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
ns-cert-type server
comp-lzo
verb 3
```

`remote my-server-1 1194` の行は修正が必要です。`my-server-1` の箇所を、利用しているLinuxサーバのIPアドレスまたはFQDNに書き換えてください。

そこまで終わったら、クライアントの設定ファイルをTunnelblickに反映させます。
Finderから、画面上部のTunnelblickアイコンにドラッグ＆ドロップしてください。

<img src="/img/2016/07-14-tunnelblick-DnD.png" alt="Tunnelblickに設定ファイルをD&D" width="auto" height="auto">

### VPNに接続

Mac上の作業です。
いよいよVPNに接続します。
Tunnelblickのアイコンから "client に接続" を選んでください。

サーバのログに接続を表すログが流れ、Tunnelblickでも接続に成功した旨が表示されるはずです。

<img src="/img/2016/07-14-tunnelblick-connecting.png" alt="VPN接続に成功" width="auto" height="auto">

この時点でVPNサーバによりプライベートIPアドレス **10.8.0.6** が割り振られているかと思います。
**10.8.0.0/24** 内での疎通確認をしましょう。

```bash VPNサーバへのプライベートIPアドレスでの疎通確認
$ ping 10.8.0.1
```


## Amazon EC2でサンプル環境を構築

<img src="/img/2016/07-14-structure.png" alt="構成図" width="auto" height="auto">

ここまではVPNサーバとクライアントのみのシンプルな構成でしたが、ここからはVPNを構築する目的をイメージして、上図のような環境を構築します。

**client1** は、 **10.0.0.0/16** のLAN内にある各種 **その他インスタンス** にリモートからSSHログインしたいとします。
**その他インスタンス** には **踏み台サーバ** 経由でのみSSHログインが許可されていて、かつ **踏み台サーバ** プライベートIPアドレスしか振られていないため、リモートから直接SSHログインする手立てがありません。
そこで、VPNを構築し、 **client1** に **10.0.255.0/24** のプライベートIPアドレスを割り振り、その後で踏み台サーバにSSHログインできるようにしましょう。

このシナリオの下、Amazon EC2で
- **vpn**: VPNサーバ
- **step**: 踏み台サーバ

の２台を構築します。
(**その他インスタンス** の構築については本記事のスコープ外とします。)

**vpn** と **step** を共に **10.0.0.0/16** のネットワークに参加させるために、Amazon VPC (Virtual Private Cloud) を構築しつつネットワーク周りの設定をいくつかする必要がありますが、これについても本記事で解説します。

IPアドレスがいくつか出てきますので、割り当てられているNICの名称と共にこちらでまとめておきます。

- **client1**
    - VPNに参加した際に割り当てられるIPアドレス
        - IPアドレス: **10.0.255.6**
        - NIC: **utun0**
- **vpn**
    - 10.0.0.0/16 のLAN内のプライベートIPアドレス
        - IPアドレス: **10.0.1.2**
        - NIC: **eth0**
        - (リモートからグローバルIPアドレスでのアクセスもNATによりここに到達している)
    - VPNに参加したクライアントと疎通するプライベートIPアドレス
        - IPアドレス: **10.0.255.1**
        - NIC: **tun0**
- **step**
    - **10.0.0.0/16** のLAN内のプライベートIPアドレス
        - IPアドレス: **10.0.2.2**
        - NIC: **eth0**

**client1** と **vpn** 間のVPN接続には、 **クライアント証明書方式** を使います。
これにより、VPNに参加したのが誰であるかをログから確認できます。
(ただし、特定のクライアント証明書を特定の個人のみが所有しているという前提のもとです。)


### ネットワークの設定

#### VPCの作成

まずはLANに相当するVPCを作成します。
AWSのコンソールから、 **サービス -> VPC** と遷移し、 **office-lan** という名称で作成します。

<img src="/img/2016/07-14-create-vpc.png" alt="AWSでのVPCの作成" width="auto" height="auto">

#### サブネットの作成

先ほどの **office-lan** VPCの中のサブネットを切ります。
今回の構成ではVPCと全く同じ **10.0.0.0/16** で切れば良いです。
名称は **office-lan-subnet** としました。

<img src="/img/2016/07-14-create-subnet.png" alt="サブネットの作成" width="auto" height="auto">


#### インターネットゲートウェイの作成

**10.0.0.0/16** のLAN内のインスタンスでは、セキュリティのためインバウンドは絞りますが、アウトバウンドはインターネット接続を許可しましょう。
そのために、インターネットゲートウェイを作成します。
名称は **office-inet-gw** としました。

<img src="/img/2016/07-14-create-internet-gateway.png" alt="AWSでのインターネットゲートウェイの作成" width="auto" height="auto">

作成したインターネットゲートウェイを選択し、 **office-lan** のVPCにアタッチします。

<img src="/img/2016/07-14-attach-internet-gateway-to-vpc.png" alt="インターネットゲートウェイをVPCにアタッチ" width="auto" height="auto">


#### ルートテーブルの設定

**office-lan** の中からインターネットへのアウトバウンド接続ができるようにするため、VPCのルーティングテーブルを設定します。
**route-office-lan** という名称でルートテーブルを作成します。

<img src="/img/2016/07-14-create-routing-table.png" alt="ルートテーブルの作成" width="auto" height="auto">

**route-office-lan** を選択し、 **ルート** タブから **office-inet-gw** を追加しましょう。

<img src="/img/2016/07-14-add-internet-gateway-to-routing-table.png" alt="ルートテーブルにインターネットゲートウェイを追加" width="auto" height="auto">


#### セキュリティグループの作成

セキュリティグループは、ファイアーウォールのようなものです。
**vpn** 用のものと **step** 用のものをそれぞれ作りましょう。

まずはシンプルな **step** 用のものを作成します。

<img src="/img/2016/07-14-create-security-group-for-step.png" alt="step用のセキュリティグループの作成" width="auto" height="auto">

作成した **step** を選択し、 **インバウンドルール** を設定します。
踏み台サーバは **10.0.0.0/16** のLAN内のみからSSH接続を受け付けます。

<img src="/img/2016/07-14-inbound-rule-for-step.png" alt="step用のインバウンドルール" width="auto" height="auto">

同様にして、 **vpn** のインバウンドルールも設定します。
VPNサーバは、VPN接続のためにUDPの1194番ポートをグローバルに開放します。
ただし、VPNサーバの設定中はSSHも許可する必要があります。 **VPNの設定が完了したらSSHのインバウンドルールは忘れずに削除しましょう。**

<img src="/img/2016/07-14-create-security-group-for-vpn.png" alt="vpn用のセキュリティグループの作成" width="auto" height="auto">


### VPNサーバ, 踏み台サーバ用インスタンスの作成

**サービス -> EC2** に遷移し、VPNサーバと踏み台サーバのEC2インスタンスを作成します。
**インスタンスの作成** ボタンを押し、ウィザードに従って作成していきます。

まずは下記のように、 **vpn** 用のインスタンスを作成します。

- **1.AMIの選択**
    - Ubuntu Server 14.04 LTS (HVM), SSD Volume Type - ami-a21529cc
- **2.インスタンスタイプの選択**
    - t2.micro
- **3.インスタンスの詳細の設定**
    - ネットワーク: **office-lan**
    - サブネット: **office-lan-subnet**
    - 自動割当パブリックIP: **有効化**
    <img src="/img/2016/07-14-create-ec2-instance-vpn-network.png" alt="vpnのインスタンスの作成 - ネットワーク設定" width="auto" height="auto">
    - eth0のプライマリIP: **10.0.1.2**
    <img src="/img/2016/07-14-create-ec2-instance-vpn-primary-ip.png" alt="vpnのインスタンスの作成 - プライマリIP" width="auto" height="auto">
- **5.インスタンスのタグ付け**
    - Name: **vpn**
- **6.セキュリティグループの設定**
    - **vpn** を選択
    <img src="/img/2016/07-14-create-ec2-instance-vpn-security-group.png" alt="vpnのインスタンスの作成 - セキュリティグループの選択" width="auto" height="auto">

**step** 用のインスタンスも同様にして作成しますが、以下の点が異なります。

- **3.インスタンスの詳細の設定**
    - 自動割当パブリックIP: **無効化**
    - eth0のプライマリIP: **10.0.2.2**
- **5.インスタンスのタグ付け**
    - Name: **step**
- **6.セキュリティグループの設定**
    - **step** を選択

特に、 **自動割当パブリックIP** を設定してしまうと、踏み台サーバがインターネットに晒されてしまうためご注意ください。

### 動作確認

しばらくして作成が完了したら、 **client1** からSSHの接続を確認します。
**vpn** のパブリックIPは、インスタンスの一覧画面から参照してください。
<img src="/img/2016/07-14-create-ec2-instance-vpn-public-ip.png" alt="vpnのインスタンスのパブリックIP確認" width="auto" height="auto">

```bash client1からvpnへのSSH接続
$ ssh -i ~/.ssh/secret-key.pem ubuntu@X.X.X.X
```

```bash vpnからstepへのSSH接続
$ ssh -i ~/.ssh/secret-key.pem 10.0.2.2
```

これでVPNサーバと踏み台サーバの構築は完了です。

以降、クライアント証明書認証のための証明書や鍵を生成することと、 **client1** から **step** へ直接SSH接続するための設定を解説します。
( **client1** から **vpn** へのSSH接続は、あくまでもVPN設定を完了させるまでの一時的なものであることを忘れないでください。VPN設定が完了したらセキュリティグループで塞ぎましょう。)


## 証明書と鍵の生成

[クイックスタート: 最初のVPN接続](#クイックスタート-最初のvpn接続) では、証明書や秘密鍵はOpenVPNのサンプルファイルをそのまま使っていました。
これは言うまでもなく脆弱です。
各VPNサーバ・クライアントごとに独自の証明書や秘密鍵を生成しましょう。

以下、 **vpn** でのroot作業となります。

```bash rootユーザになる
$ sudo su
```

証明書や秘密鍵を生成するためのパッケージ、 **easy-rsa** をインストールします。

```bash easy-rsaのインストール
root> apt-get install easy-rsa
```

**easy-rsa** は複数のシェルスクリプトから成ります。
`dpkg -L easy-rsa |grep build-ca` などで、シェルスクリプト群が置かれたディレクトリを探して `cd` します。

```bash easy-rsaのディレクトリにcd
root> dpkg -L easy-rsa |grep build-ca
/usr/share/easy-rsa/build-ca
 
root> cd /usr/share/easy-rsa/
```

ここからの作業を理解するためには **PKI (Public Key Infrastructure; 公開鍵基盤)** の知識が必要になります。
(HTTPSを理解していれば問題ないです)

### CAの証明書と秘密鍵の生成

まずはCAの証明書に入る基本的な情報を設定します。
`vars` ファイルを編集し、以下の6行を適当に変更してください。

```bash /usr/share/easy-rsa/vars の編集箇所
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"
```

編集後、以下のコマンドでCAの証明書と鍵を生成します。

```bash CAの証明書と秘密鍵の生成
root> . ./vars     # 設定値の読込み
root> ./clean-all  # 既に生成済みの証明書や鍵があれば削除
root> ./build-ca   # 証明書と秘密鍵の対話的生成
...
Common Name (eg, your name or your server's hostname) [Fort-Funston CA]:Office-CA
...
 
root> ls keys/
ca.crt  ca.key  index.txt  serial
```

**Common Name** は `vars` ファイルで指定していないので、必要に応じて入力してください。
(上記例では **Office-CA** としました)


### VPNサーバの証明書と秘密鍵、DHパラメータの生成

VPNサーバの証明書と秘密鍵の生成は、コマンドを1つ打つだけです。
ただし、第一引数はVPNサーバの **Common Name** です。

```bash VPNサーバの証明書と秘密鍵の生成
root> ./build-key-server server  # 証明書と秘密鍵の対話的生成
...
Sign the certificate? [y/n]:y
 
1 out of 1 certificate requests certified, commit? [y/n]y
 
root> ls keys/server.*
keys/server.crt  keys/server.csr  keys/server.key
```

次に、通信の暗号化に使うDH(Diffie Hellman)パラメータを生成します。

```bash DHパラメータの生成
root> ./build-dh
 
root> ls keys/dh*
keys/dh2048.pem
```


### VPNクライアントの証明書と秘密鍵の生成

VPNクライアントの証明書と秘密鍵の生成もコマンドを1つ打つだけです。
第一引数はVPNクライアントの **Common Name** です。

```bash
root> ./build-key client1
...
Sign the certificate? [y/n]:y
 
1 out of 1 certificate requests certified, commit? [y/n]y
 
root> ls keys/client1.*
keys/client1.crt  keys/client1.csr  keys/client1.key
```

クライアントの証明書と秘密鍵・CA証明書は、 **client1** に安全な方法で転送してください。
**クライアント証明書が流出した場合、誰からのVPN接続であるかが不明瞭になったり、最悪の場合許可していないクライアントがLAN内に侵入するのを許してしまいます。**

ここでは、あまり安全な方法ではありませんが、 `scp` で転送します。

```bash [vpnでの作業]
## 一時的に、ubuntuユーザから秘密鍵がreadできるようにする
 
root> chmod 755 keys
root> chmod 644 keys/client1.key
```

```bash [client1での作業] client1から、クライアント証明書と秘密鍵を取得する
$ cd ~/openvpn-test/client/  # 本記事冒頭で作成したディレクトリ
$ rm ca.crt client.{crt,key}
$ ls
client.conf

$ scp -i ~/.ssh/secret-key.pem ubuntu@X.X.X.X:/usr/share/easy-rsa/keys/ca.crt .
$ scp -i ~/.ssh/secret-key.pem ubuntu@X.X.X.X:/usr/share/easy-rsa/keys/client1.{crt,key} .
$ ls
ca.crt      client.conf client1.crt client1.key
```

```bash [vpnでの作業]
## 各種パーミッションを戻す
 
root> chmod 700 keys
root> chmod 600 keys/client1.key
```


## VPNサーバの設定

PKIが整ったので、いよいよVPNサーバの設定に移ります。
最終目標は **client1** から **step** へSSH接続をすることですが、その際 **vpn** にはルータの役割をさせる必要もあるので、その方法も説明します。

### OpenVPNの設定

**vpn** でOpenVPNをインストールします。

```bash OpenVPNのインストール
root> apt-get install openvpn
```

設定ファイルと証明書・秘密鍵・DHパラメータを `/etc/openvpn/` 以下に配置しましょう。

```bash 設定ファイルの配置
root> zcat /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
```

```bash 証明書・秘密鍵・DHパラメータの配置
root> cp /usr/share/easy-rsa/keys/ca.crt /etc/openvpn/
root> cp /usr/share/easy-rsa/keys/server.{crt,key} /etc/openvpn/
root> cp /usr/share/easy-rsa/keys/dh2048.pem /etc/openvpn/
 
root> ls /etc/openvpn/
ca.crt  server.crt  server.key  update-resolv-conf
```

`/etc/openvpn/server.conf` の次の箇所を変更します。

```diff server.conf の変更箇所
+ dh dh2048.pem
- dh dh1024.pem
+ server 10.0.255.0 255.255.255.0
- server 10.8.0.0 255.255.255.0
+ push "route 10.0.0.0 255.255.0.0"
+ status openvpn-status.log
+ log openvpn.log
```

`push "route 10.0.0.0 255.255.0.0"` では、 **client1** に対して「 **10.0.0.0/16** への通信はVPNサーバを経由せよ」というルーティングを足しています。

現時点の設定ファイルの内容は、以下のようになっている想定です。
(空行、コメント行は除く)

```config server.conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh2048.pem
server 10.0.255.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.0.0 255.255.0.0"
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 3
status openvpn-status.log
log openvpn.log
```

この時点で `cd /etc/openvpn/ ; openvpn /etc/openvpn/server.conf` と起動し、 `/etc/openvpn/openvpn.log` に起動ログが正しく流れていることを確認しましょう。

`/etc/init.d/openvpn` もあるので、 `service openvpn {start|stop|status}` などでデーモンとして管理することができます。
マシンが再起動した時にデーモンが自動起動されるようにしましょう。

```bash openvpnの自動起動設定
root> update-rc.d openvpn defaults
```


### IPフォワード

**client1** から **step** に接続するためには、必ず **vpn** を経由しなければなりません。
**client1** と **step** の間には、L2レイヤでの直接の接続がないためです。

**vpn** が **client1** から受け取ったパケットを **step** に転送するためには、 **vpn** にルータとしての設定をしてやる必要があります。

```bash vpnをルータにする
root> echo 1 > /proc/sys/net/ipv4/ip_forward
```

この設定は再起動をすると消えてしまうので、 `/etc/sysctl.conf` のファイルから一行コメントインして永続化します。

```diff /etc/sysctl.confの編集(vpnをルータにする設定の永続化)
+ net.ipv4.ip_forward=1
- #net.ipv4.ip_forward=1
```


### TUNフォワード

**client1**, **vpn**, **step** のNICレベルでの接続を簡易化すると、下図のようになります。

```config L2ネットワーク図
client1 [utun0]
           ^
           |
           v
         [tun0] vpn [eth0]
                      ^
                      |
                      v
                    [eth0] step
```

**vpn:tun0** は **step:eth0** と直接接続されていないので、 **client1** から **step** にパケットを送るためには、 **vpn** の中で **tun0 -> eth0** というフォワーディングが必要になります。
更に、 **step** に到達したパケットが **vpn** 経由で **client1** に返ってくるように、NAT変換する必要もあります。
やることは単純なNAT変換です。

```bash vpnにおいて、TUNフォーワードとNAT変換
root> iptables -A FORWARD -i tun0 -o eth0 -j ACCEPT
root> iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

[IPフォワード](#ipフォワード) と同様、これも永続化する必要があります。
Debian系では `iptables-persistent` パッケージを使って永続化するのが簡単です。

```bash
root> apt-get install iptables-persistent
```

初回は、インストール時の画面で "Yes" と答えるだけで保存されますが、次回以降は `service iptables-persistent save` コマンドで永続化できます。


## クライアントからVPN越しに踏み台サーバへ接続

[クイックスタート: 最初のVPN接続](#クイックスタート-最初のvpn接続) と同様、Tunnelblickを使って接続します。

まずは設定ファイルの編集です。 `remote` , `cert` , `key` の行を編集します。

```config client.conf
client
dev tun
proto udp
remote X.X.X.X 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
ns-cert-type server
comp-lzo
verb 3
```

Tunnelblickに設定ファイルをドラッグ＆ドロップし接続すると、 **client1** から **step** へSSH接続できるようになっているはずです。

```bash client1からstepへのSSH接続
$ ssh -i ~/.ssh/secret-key.pem 10.0.2.2
```

これで、プライベートアドレスのみが割り振られたLAN内のサーバにリモートから接続することができました。

また、VPNサーバのログを確認すると、接続元がクライアント証明書のCommon Name単位で記録されていることも確認できます。
(ここでは **[client1]** と記録されている)

```bash VPN接続元情報がクライアント証明書のCommon Name単位で記録されている
root> less /etc/openvpn/openvpn.log
...
Sun Jul 24 09:35:09 2016 60.236.44.98:54522 [client1] Peer Connection Initiated with [AF_INET]Y.Y.Y.Y:54522
...
```


## まとめと発展的な話題

本記事では、Amazon EC2を使ってクライアント証明書認証方式のSSL-VPNを構築し、プライベートIPアドレスのみが割り振られた踏み台サーバへリモートのクライアントからSSHするまでの実践的な方法を記載しました。
また、PKIに基づくCA, サーバ, クライアントの証明書と秘密鍵の生成方法も紹介しました。

以下、本記事でカバーできなかった重要なセキュリティに関する話題に簡潔に触れます。

### セキュリティの強化

クライアント証明書方式の認証は、 **クライアント証明書を所有しているのが特定の個人に限定される場合** 、セキュリティ面でもロギングの面でも有効です。
しかし、クライアント証明書の入ったマシンの盗難などに備え、認証は「所有」だけではなく「知っていること」と組み合わせるのが良いでしょう。
典型的には、ID/パスワード認証との併用が考えられます。
詳しくは [OpenVPNで使用できる認証方法](http://www.openvpn.jp/document/authentication-methods/) を参照してください。

また、クライアント証明書はVPNサーバ側で **失効** させることができます。
[How To - 証明書の失効](http://www.openvpn.jp/document/how-to/#Revoking) などを読み、マシンの紛失などがあった際に即座に失効の対応を取れるようにしておきましょう。

本記事ではサーバ・クライアントの証明書・秘密鍵を **vpn** 上で生成しましたが、これはあまり褒められた方法ではありません。
VPNサーバのセキュリティが突破された場合、攻撃者が秘密鍵を入手できてしまうからです。
この記事のVPNサーバはグローバルIPアドレスが割り振られており、セキュリティレベルは決して高くありません。
証明書・秘密鍵の生成は、インターネットから隔離されたマシンで行うことなどを検討してください。
クライアントの数が多くなった場合の鍵の配布などは、 [ActiveDirectoryによるユーザ証明書の自動配布](http://www.viva-musen.net/archives/cat_883975.html) なども参考になるかと思います。

**client1** から **step** に接続する際、 **vpn** がNATの役割をしているため、 **step** 側のログに **client1** のIPアドレスを記録することができません。
VPNから **client1** に **10.0.255.6** というIPアドレスが割り当てられても、 **step** サーバから見える送信元IPアドレスは **vpn** の **eth0** の **10.0.1.2** となってしまうのです。
**step** 側のログに「誰がログインしたか」を記録する要件がある場合、SSHのログインユーザのレベルでしか記録できない点に注意してください。
(client-config-dirを使ってVPN参加クライアントのIPアドレスを固定しつつ、 `iptables` のSNATルールをクライアントの数だけ記載すれば、 **vpn** から **step** へのIPアドレスをクライアントごとに固定することがおそらく可能ですが、未検証です)

その他にも、 [How To - OpenVPNのセキュリティを強化する](http://www.openvpn.jp/document/how-to/#Security) にはOpenVPNのセキュリティを高めるための事項がいくつか紹介されています。
併せてお読みください。
