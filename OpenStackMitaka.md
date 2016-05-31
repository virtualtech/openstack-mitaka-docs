Title: OpenStack構築手順書 Mitaka版
Company: 日本仮想化技術<br>
Version:0.9.9-3<br>

# OpenStack構築手順書 Mitaka版

<div class="title">
バージョン：0.9.9-3 (2016/05/31作成) <br>
日本仮想化技術株式会社
</div>

<!-- BREAK -->

## 変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.9.0|2016/05/13|Mitaka版執筆開始|
|0.9.9-1|2016/05/30|ベータ版を公開|
|0.9.9-2|2016/05/30| 「1-5-2 プロキシーの設定」を修正、改行の調整|
|0.9.9-3|2016/05/31| 改行の調整|

````
筆者注:このドキュメントに対する提案や誤りの指摘は
Issue登録か、日本仮想化技術までメールにてお願いします。
https://github.com/virtualtech/openstack-mitaka-docs/issues
````

<!-- BREAK -->

## 目次

<!--TOC max3-->
<!-- BREAK -->

# Part.1 OpenStack 構築編
<br>
本章は、OpenStack Foundationが公開している公式ドキュメント「OpenStack Installation Guide for Ubuntu 14.04」の内容から、「Block Storage Service」までの構築手順をベースに加筆したものです。
OpenStackをUbuntu Server 14.04 ベースで構築する手順を解説しています。

Canonical社が提供するCloud Archiveリポジトリーを使って、OpenStackの最新版Mitakaを導入しましょう。

<!-- BREAK -->

## 1. 構築する環境について

### 1-1 環境構築に使用するOS

本書はCanonicalのUbuntu ServerとCloud Archiveリポジトリーのパッケージを使って、OpenStack Mitakaを構築する手順を解説したものです。

Ubuntu Serverでは新しいハードウェアのサポートを積極的に行うディストリビューションです。そのため、Linux KernelのバージョンをTrustyの場合は14.04.2以降のLTSのポイントリリースごとに、スタンダート版のUbuntuと同様のバージョンに置き換えてリリースしています。

- <https://wiki.ubuntu.com/Kernel/LTSEnablementStack>

一般的な利用では特に問題ありませんが、OpenStackとSDNのソリューションを連携した環境を作る場合などに、Linux KernelやOSのバージョンを考慮しなくてはならない場合があります。
また、Trustyの場合はLinux Kernel v3.13以外のバージョンのサポート期間はTrustyのサポート期間と同様ではありません。期限が切れた後はLinux Kernel v3.13にダウングレードするか、新しいLinux Kernelをインストールすることができるメタパッケージを手動で導入する必要がありますので注意してください。

本書ではサポート期間が長く、Trustyの初期リリースの標準カーネルであるLinux Kernel v3.13を使うために、以下のURLよりイメージをダウンロードしたUbuntu Server 14.04.1 LTS(以下Ubuntu Server)のイメージを使ってインストールします。インストール後`apt-get dist-upgrade`を行って最新のアップデートを適用した状態にしてください。Trustyではこのコマンドを実行してもカーネルのバージョンがアップグレードされることはありません。

本書は3.13.0-86以降のバージョンのカーネルで動作するUbuntu 14.04.4を想定しています。

- <http://old-releases.ubuntu.com/releases/14.04.1/ubuntu-14.04.1-server-amd64.iso>

<!-- BREAK -->

### 1-2 作成するサーバー（ノード）

本書はOpenStack環境をController,Computeの2台のサーバー上に構築することを想定しています。

| コントローラー | コンピュート 
| -------------- | --------------
| RabbitMQ       | Linux KVM
| NTP            | Nova Compute
| MariaDB        | Linux Bridge Agent
| Keystone       
| Glance
| Nova
| Neutron Server
| Linux Bridge Agent
| L3 Agent
| DHCP Agent
| Metadata Agent
| Cinder

<!-- BREAK -->

### 1-3 ネットワークセグメントの設定

IPアドレスは以下の構成で構築されている前提で解説します。

|インターフェース|eth0|
|:---|:---|:---|
|ネットワーク|10.0.0.0/24|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

### 1-4 各ノードのネットワーク設定

各ノードのネットワーク設定は以下の通りです。

+ コントローラーノード

|インターフェース|eth0|
|:---|:---|:---|
|IPアドレス|10.0.0.101|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

+ コンピュートノード

|インターフェース|eth0|
|:---|:---|:---|
|IPアドレス|10.0.0.102|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

<!-- BREAK -->

### 1-5 Ubuntu Serverのインストール

#### 1-5-1 インストール

2台のサーバーに対し、Ubuntu Serverをインストールします。要点は以下の通りです。

+ 優先ネットワークインターフェースをeth0に指定 
 + インターネットへ接続するインターフェースはeth0を使用するため、インストール中はeth0を優先ネットワークとして指定します。
+ OSは最小インストール
 + パッケージ選択ではOpenSSH serverのみ選択


【インストール時の設定パラメータ例】

|設定項目|設定例|
|:---|:---|
|初期起動時のLanguage|English|
|起動|Install Ubuntu Server|
|言語|English - English|
|地域の設定|other→Asia→Japan|
|地域の言語|United States - en_US.UTF-8|
|キーボードレイアウトの認識|No|
|キーボードの言語|Japanese→Japanese|
|優先するNIC|eth0: Ethernet|
|ホスト名|それぞれのノード名(controller, compute)|
|ユーザ名とパスワード|フルネームで入力|
|アカウント名|ユーザ名のファーストネームで設定される|
|パスワード|任意のパスワード|

パスワードを入力後、Weak passwordという警告が出た場合はYesを選択するか、警告が出ないようなより複雑なパスワードを設定してください。

<!-- BREAK -->

|設定項目|設定例|
|:---|:---|
|ホームの暗号化|任意|
|タイムゾーン|Asia/Tokyoであることを確認|
|パーティション設定|Guided - use entire disk and set up LVM|
|パーティション選択|sdaを選択|
|パーティション書き込み|Yesを選択|
|パーティションサイズ|デフォルトのまま|
|変更の書き込み|Yesを選択|
|HTTP proxy|環境に合わせて任意|
|アップグレード|No automatic updatesを推奨|
|ソフトウェア|OpenSSH serverのみ選択|
|GRUB|Yesを選択|
|インストール完了|Continueを選択|

```
筆者注:
Ubuntuインストール時に選択した言語がインストール後も使われます。
Ubuntu Serverで日本語の言語を設定した場合、標準出力や標準エラー出力が文字化けするなど様々な問題が起きますので、言語は英語を設定されることを推奨します。
```

#### 1-5-2 プロキシーの設定

外部ネットワークとの接続にプロキシーの設定が必要な場合は、aptコマンドを使ってパッケージの照会やダウンロードを行うために次のような設定をする必要があります。

+ システムのプロキシー設定

```
# vi /etc/environment
http_proxy="http://proxy.example.com:8080/"
https_proxy="https://proxy.example.com:8080/"
no_proxy=localhost,controller,compute
```

+ APTのプロキシー設定

```
# vi /etc/apt/apt.conf
Acquire::http::proxy "http://proxy.example.com:8080/";
Acquire::https::proxy "https://proxy.example.com:8080/";
```

より詳細な情報は下記のサイトの情報を確認ください。

- <https://help.ubuntu.com/community/AptGet/Howto>
- <http://gihyo.jp/admin/serial/01/ubuntu-recipe/0331>

<!-- BREAK -->

### 1-6 Ubuntu Serverへのログインとroot権限

Ubuntuはデフォルト設定でrootユーザーの利用を許可していないため、root権限が必要となる作業は以下のように行ってください。

+ rootユーザーで直接ログインできないので、インストール時に作成したアカウントでログインする。
+ root権限が必要な場合には、sudoコマンドを使用する。
+ rootで連続して作業したい場合には、sudo -iコマンドでシェルを起動する。

### 1-7 設定ファイル等の記述について

+ 設定ファイルは特別な記述が無い限り、必要な設定を抜粋したものです。
+ 特に変更の必要がない設定項目は省略されています。
+ [見出し]が付いている場合、その見出しから次の見出しまでの間に設定を記述します。
+ コメントアウトされていない設定項目が存在する場合には、値を変更してください。多くの設定項目は記述が存在しているため、エディタの検索機能で検索することをお勧めします。
+ 特定のホストでコマンドを実行する場合はコマンドの冒頭にホスト名を記述しています。

【設定ファイルの記述例】

```
controller# vi /etc/glance/glance-api.conf ←コマンド冒頭にこのコマンドを実行するホストを記述

[DEFAULT]
debug=true                      ← コメントをはずす

[database]
#connection = sqlite:////var/lib/glance/glance.sqlite   ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記

[keystone_authtoken] ← 見出し
#auth_host = 127.0.0.1 ← 既存設定をコメントアウト
auth_host = controller ← 追記

auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0 ← 追記
admin_tenant_name = service ← 変更
admin_user = glance ← 変更
admin_password = password ← 変更
```
<!-- BREAK -->

## 2. OpenStackインストール前の設定

OpenStackパッケージのインストール前に各々のノードで以下の設定を行います。

+ ネットワークデバイスの設定
+ ホスト名と静的な名前解決の設定
+ リポジトリーの設定とパッケージの更新
+ Chronyサーバーのインストール（コントローラーノードのみ）
+ Chronyクライアントのインストール
+ Python用MySQL/MariaDBクライアントのインストール
+ MariaDBのインストール（コントローラーノードのみ）
+ RabbitMQのインストール（コントローラーノードのみ）
+ 環境変数設定ファイルの作成（コントローラーノードのみ）
+ memcachedのインストールと設定（コントローラーノードのみ）

### 2-1 ネットワークデバイスの設定

各ノードの/etc/network/interfacesを編集し、IPアドレスの設定を行います。

#### 2-1-1 コントローラーノードのIPアドレスの設定

```
controller# vi /etc/network/interfaces
auto eth0
iface eth0 inet static
      address 10.0.0.101
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```

#### 2-1-2 コンピュートノードのIPアドレスの設定

```
compute# vi /etc/network/interfaces
auto eth0
iface eth0 inet static
      address 10.0.0.102
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```
<!-- BREAK -->

#### 2-1-3 ネットワークの設定を反映

各ノードで変更した設定を反映させるため、ホストを再起動します。

```
# shutdown -r now
```

### 2-2 ホスト名と静的な名前解決の設定

ホスト名でノードにアクセスするにはDNSサーバーで名前解決する方法やhostsファイルに書く方法が挙げられます。
本書では各ノードの/etc/hostsに各ノードのIPアドレスとホスト名を記述してhostsファイルを使って名前解決します。127.0.1.1の行はコメントアウトします。

#### 2-2-1 各ノードのホスト名の設定

各ノードのホスト名をhostnamectlコマンドを使って設定します。反映させるためには一度ログインしなおす必要があります。

（例）コントローラーノードの場合

```
controller# hostnamectl set-hostname controller
# cat /etc/hostname
controller
```

#### 2-2-2 各ノードの/etc/hostsの設定

すべてのノードで127.0.1.1の行をコメントアウトします。
またホスト名で名前引きできるように設定します。

（例）コントローラーノードの場合

```
controller# vi /etc/hosts
127.0.0.1 localhost
#127.0.1.1 controller ← 既存設定をコメントアウト
#ext
10.0.0.101 controller
10.0.0.102 compute
```

<!-- BREAK -->

### 2-3 リポジトリーの設定とパッケージの更新


コントローラーノードとコンピュートノードで以下のコマンドを実行し、Mitaka向けUbuntu Cloud Archiveリポジトリーを登録します。


```
# apt-get install software-properties-common
# add-apt-repository cloud-archive:mitaka

  Ubuntu Cloud Archive for OpenStack Mitaka
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive

Press [ENTER] to continue or ctrl-c to cancel adding it    ← Enterキーを押す
...
Importing ubuntu-cloud.archive.canonical.com keyring

OK

Processing ubuntu-cloud.archive.canonical.com removal keyring

gpg: /etc/apt/trustdb.gpg: trustdb created

OK
```

各ノードのシステムをアップデートします。Ubuntuではパッケージのインストールやアップデートの際にまず`apt-get update`を実行してリポジトリーの情報の更新が必要です。そのあと`apt-get -y dist-upgrade`でアップグレードを行います。カーネルの更新があった場合は再起動してください。

なお、`apt-get update`は頻繁に実行する必要はありません。日をまたいで作業する際や、コマンドを実行しない場合にパッケージ更新やパッケージのインストールでエラーが出る場合は実行してください。以降の手順では`apt-get update`を省略します。

```
# apt-get update && apt-get dist-upgrade
```

### 2-4 OpenStackクライアントとMariaDBクライアントのインストール

コントローラーノードでOpenStackクライアントとPython用のMySQL/MariaDBクライアントをインストールします。依存するパッケージは全てインストールします。

```
controller# apt-get install python-openstackclient python-pymysql
```

<!-- BREAK -->

### 2-5 時刻同期サーバーのインストールと設定

#### 2-5-1 時刻同期サーバーChronyの設定

各ノードで時刻を正確にするために時刻同期サーバーのChronyをインストールします。

```
# apt-get install -y chrony
```

#### 2-5-2 コントローラーノードの時刻同期サーバーの設定

コントローラーノードで公開NTPサーバーと同期するNTPサーバーを構築します。
適切な公開NTPサーバー(ex.ntp.nict.jp etc..)を指定します。ネットワーク内にNTPサーバーがある場合はそのサーバーを指定します。

設定を変更した場合はNTPサービスを再起動します。

```
controller# service chrony restart
```

<!-- BREAK -->

#### 2-5-3 その他ノードの時刻同期サーバーの設定

コンピュートノードでコントローラーノードと同期するNTPサーバーを構築します。

```
compute# vi /etc/chrony/chrony.conf

#server 0.debian.pool.ntp.org offline minpoll 8 #デフォルト設定はコメントアウト
#server 1.debian.pool.ntp.org offline minpoll 8
#server 2.debian.pool.ntp.org offline minpoll 8
#server 3.debian.pool.ntp.org offline minpoll 8

server controller iburst
```

設定を適用するため、NTPサービスを再起動します。

```
compute# service chrony restart
```

#### 2-5-4 NTPサーバーの動作確認

構築した環境でコマンドを実行して、各NTPサーバーが同期していることを確認します。

- 公開NTPサーバーと同期しているコントローラーノード

```
controller# chronyc sources
chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* chobi.paina.jp                2   8    17     1    +47us[ -312us] +/-   13ms
^- v157-7-235-92.z1d6.static     2   8    17     1  +1235us[+1235us] +/-   45ms
^- edm.butyshop.com              3   8    17     0  -2483us[-2483us] +/-   82ms
^- y.ns.gin.ntt.net              2   8    17     0  +1275us[+1275us] +/-   35ms
```

- コントローラーノードと同期しているその他ノード

```
compute# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* controller                3   6    77    25   -509us[-1484us] +/-   13ms
```

<!-- BREAK -->

### 2-6 MariaDBのインストール

コントローラーノードにデータベースサーバーのMariaDBをインストールします。

#### 2-6-1 パッケージのインストール

apt-getコマンドでmariadb-serverパッケージをインストールします。

```
controller# apt-get install -y mariadb-server
```

インストール中にパスワードの入力を要求されますので、MariaDBのrootユーザーに対するパスワードを設定します。
本書ではパスワードとして「password」を設定します。

#### 2-6-2 MariaDBの設定を変更

OpenStack用のMariaDB設定ファイル /etc/mysql/conf.d/openstack.cnf を作成し、以下を設定します。


+ バインドアドレス：eth0へ割り当てたIPアドレス

+ 文字コード：UTF-8


別のノードからMariaDBへアクセスできるようにするためバインドアドレスを変更します。加えて使用する文字コードをutf8に変更します。

※文字コードをutf8に変更しないとOpenStackモジュールとデータベース間の通信でエラーが発生します。

```
controller# vi /etc/mysql/conf.d/openstack.cnf

[mysqld]
bind-address = 10.0.0.101                   ←controllerのIPアドレス
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
character-set-server = utf8
```

<!-- BREAK -->

#### 2-6-3 MariaDBサービスの再起動

変更した設定を反映させるためMariaDBのサービスを再起動します。

```
controller# service mysql restart
```

#### 2-6-4 MariaDBクライアントのインストール

コンピュートノードにMariaDBクライアントをインストールします。コントローラーノードでインストール済みのMariaDBと同様のバージョンを指定します。

````
compute# apt-get install -y mariadb-client-5.5 mariadb-client-core-5.5
````

### 2-7 RabbitMQのインストール

OpenStackは、オペレーションやステータス情報を各サービス間で連携するためにメッセージブローカーを使用しています。OpenStackではRabbitMQ、Qpid、ZeroMQなど複数のメッセージブローカーサービスに対応しています。
本書ではRabbitMQをインストールする例を説明します。

#### 2-7-1 パッケージのインストール

コントローラーノードにrabbitmq-serverパッケージをインストールします。

```
controller# apt-get install rabbitmq-server
```

#### 2-7-2 openstackユーザーの作成とパーミッションの設定

RabbitMQにアクセスするためのユーザーとしてopenstackユーザーを作成し、必要なパーミッションを設定します。
以下コマンドはRabbitMQのパスワードをpasswordに設定する例です。

```
controller# rabbitmqctl add_user openstack password
controller# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

<!-- BREAK -->

#### 2-7-3 待ち受けIPアドレス・ポートとセキュリティ設定の変更

以下の設定ファイルを作成し、RabbitMQの待ち受けポートとIPアドレスを設定します。

+ 待ち受け設定の追加

```
controller# vi /etc/rabbitmq/rabbitmq-env.conf
RABBITMQ_NODE_IP_ADDRESS=10.0.0.101    ← controllerのIPアドレス
RABBITMQ_NODE_PORT=5672
HOSTNAME=controller
```

#### 2-7-4 RabbitMQサービス再起動と確認

+ ログの確認

メッセージブローカーサービスが正常に動いていないと、OpenStackの各コンポーネントは正常に動きません。RabbitMQサービスの再起動と動作確認を行い、確実に動作していることを確認します。

```
controller# service rabbitmq-server restart
controller# tailf /var/log/rabbitmq/rabbit@controller.log
...
 =INFO REPORT==== 16-May-2016::13:41:30 ===

started TCP Listener on 10.0.0.101:5672  ←待受IPとポートを確認

 =INFO REPORT==== 16-May-2016::13:41:30 ===

Server startup complete; 0 plugins started.
```

※新たなエラーが表示されなければ問題ありません。

<!-- BREAK -->

### 2-8 環境変数設定ファイルの作成

#### 2-8-1 admin環境変数設定ファイルの作成

adminユーザー用の環境変数設定ファイルを作成します。

```
controller# vi ~/admin-openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(admin)\$ '
```

#### 2-8-2 demo環境変数設定ファイルの作成

demoユーザー用の環境変数設定ファイルを作成します。

```
controller# vi ~/demo-openrc.sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='\u@\h \W(demo)\$ '
```

<!-- BREAK -->

### 2-9 memcachedのインストールと設定

コントローラーノードで認証機構のトークンをキャッシュするための用途でmemcachedをインストールします。

```
controller# apt-get install memcached python-memcache
```

設定ファイル /etc/memcached.conf を変更します。
既存の行 -l 127.0.0.1 をcontrollerのIPアドレスに変更します。

```
controller# vi /etc/memcached.conf
-l 10.0.0.101    ← controllerのIPアドレス
```

memcachedサービスを再起動します。

```
controller## service memcached restart
```
<!-- BREAK -->

## 3. Keystoneのインストールと設定（コントローラーノード）

各サービス間の連携時に使用する認証サービスKeystoneのインストールと設定を行います。

### 3-1 データベースを作成

MariaDBにKeystoneで使用するデータベースを作成しアクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 3-2 データベースの確認

MariaDBに keystoneユーザーでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -u keystone -p
Enter password:  ← MariaDBのkeystoneパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
```

<!-- BREAK -->

### 3-3 admin_tokenの決定

Keystoneのadmin_tokenに設定する10桁の文字列を次のコマンドで決定します。出力結果はランダム英数字になります。

```
controller# openssl rand -hex 10
45742a05a541f26ddee8
```

### 3-4 パッケージのインストール

Keystoneのインストール後、サービスが自動的に起動しないように以下のコマンドを実行し無効化します。

```
controller# echo "manual" > /etc/init/keystone.override
```

apt-getコマンドでkeystoneおよび必要なパッケージをインストールします。

```
controller# apt-get install keystone apache2 libapache2-mod-wsgi
```

<!-- BREAK -->

### 3-5 Keystoneの設定変更

keystoneの設定ファイルを変更します。

```
controller# vi /etc/keystone/keystone.conf

[DEFAULT]
admin_token = 45742a05a541f26ddee8   ← 追記(3-3で出力されたキーを入力)
log_dir = /var/log/keystone          ← 設定されていることを確認
...
[database]
#connection = sqlite:////var/lib/keystone/keystone.db    ← 既存設定をコメントアウト
connection = mysql+pymysql://keystone:password@controller/keystone  ← 追記
...

[token]
...
provider = fernet          ← 追記 
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/keystone/keystone.conf | grep -v "^\s*$" | grep -v "^\s*#"
```


### 3-6 データベースに展開

```
controller# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

### 3-7  Fernet キーの初期化

keystone-manage コマンドで Fernet キーを初期化します。

```
controller# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

<!-- BREAK -->

### 3-8 Apache Webサーバーの設定

+ コントローラーノードの /etc/apache2/apache2.conf のServerNameにコントローラーノードのホスト名を設定します。

```
# Global configuration
#
ServerName controller
...
```
+ コントローラーノードで/etc/apache2/sites-available/wsgi-keystone.confを作成し、次の内容を記述します。

```
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>
```
<!-- BREAK -->

+  keystone サービスのVirtualHostを有効化します。

```
controller# ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

### 3-9 サービスの再起動と不要DBの削除

+ Apache Webサーバーを再起動します。

```
controller# service apache2 restart
```

+ パッケージのインストール時に作成される不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/keystone/keystone.db
```

<!-- BREAK -->

### 3-10 サービスとAPIエンドポイントの作成

以下コマンドでサービスとAPIエンドポイントを設定します。

+ 環境変数の設定

```
controller# export OS_TOKEN=45742a05a541f26ddee8  ← 追記(3-5で設定したキーを入力)
controller# export OS_URL=http://controller:35357/v3
controller# export OS_IDENTITY_API_VERSION=3
```

+ サービスの作成

```
controller# openstack service create \
 --name keystone --description "OpenStack Identity" identity

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | e56875264dad4553bcd5682f9ecc15b9 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```

+ API エンドポイントの作成

```
controller# openstack endpoint create --region RegionOne \
  identity public http://controller:5000/v3

controller# openstack endpoint create --region RegionOne \
  identity internal http://controller:5000/v3

controller#openstack endpoint create --region RegionOne \
  identity admin http://controller:35357/v3
```

<!-- BREAK -->

### 3-11 プロジェクトとユーザー、ロールの作成

以下コマンドで認証情報（プロジェクト・ユーザー・ロール）を設定します。

+ default ドメインの作成

```
controller# openstack domain create --description "Default Domain" default
```

+ admin プロジェクトの作成

```
controller# openstack project create --domain default \
  --description "Admin Project" admin
```

+ admin ユーザーの作成

```
controller# openstack user create --domain default --password-prompt admin
User Password: password  #adminユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | 59f4de8f7a7044d1b307dd3a89091bd6 |
| name      | admin                            |
+-----------+----------------------------------+
```

+ admin ロールの作成

```
controller# openstack role create admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | e63b506dc78e45a6ab282dd68c7fc53f |
| name  | admin                            |
+-------+----------------------------------+
```

+ adminロールを adminプロジェクトと adminユーザーに追加します。

```
controller# openstack role add --project admin --user admin admin
``` 
<!-- BREAK -->

+ サービスプロジェクトの作成

```
controller# openstack project create --domain default \
  --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | c12fc3119bb046a09ba53e0566004f76 |
| enabled     | True                             |
| id          | a85f5704eb5e485d8de664132a3954bd |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | c12fc3119bb046a09ba53e0566004f76 |
+-------------+----------------------------------+
```

+ demo プロジェクトの作成

```
controller# openstack project create --domain default \
 --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | c12fc3119bb046a09ba53e0566004f76 |
| enabled     | True                             |
| id          | 4e00692779864ed185356843ebeb0e2f |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | c12fc3119bb046a09ba53e0566004f76 |
+-------------+----------------------------------+
```

+ demo ユーザーの作成

```
controller# openstack user create --domain default \
 --password-prompt demo
User Password: password  #demoユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | a5fab4bc7a0645cd85e68af50c02d2fe |
| name      | demo                             |
+-----------+----------------------------------+
```

<!-- BREAK -->

+ user ロールの作成

```
controller# openstack role create user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 57c726c5060648898c5dd693a9661aa2 |
| name  | user                             |
+-------+----------------------------------+
```

+ user ロールを demo プロジェクトと demo ユーザーに追加します。

```
controller# openstack role add --project demo --user demo user
```

### 3-12 Keystoneの動作確認

他のサービスをインストールする前に Keystone が正しく構築、設定されたか動作を検証します。

+ セキュリティを確保するため、一時認証トークンメカニズムを無効化します。
  + /etc/keystone/keystone-paste.iniを開き、[pipeline:public_api]と[pipeline:admin_api]と[pipeline:api_v3]セクション（訳者注:..のpipeline行）から、admin_token_authを削除します。

```
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension user_crud_extension public_service
...
[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension crud_extension admin_service
...
[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
```

+ 一時的に設定した環境変数 OS_TOKEN  と OS_URL を解除します。

```
controller# unset OS_TOKEN OS_URL
```

<!-- BREAK -->

動作確認のためadminおよびdemoテナントに対し認証トークンを要求します。
admin、demoユーザーのパスワードを入力します。

+ admin ユーザーとして認証トークンを要求します。

```
controller# openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2016-05-17T02:08:06.019234Z      |
| id         | gAAAAABXOm72ayy8PZc2gSZi5CdO...  |
| project_id | 360855029dbe44649a5dc862c5a3caf1 |
| user_id    | 59f4de8f7a7044d1b307dd3a89091bd6 |
+------------+----------------------------------+
```

正常な応答だと、/var/log/apache2/keystone_access.log に「"POST /v3/auth/tokens HTTP/1.1" 201」が記録されます。
例えば故意に認証用パスワードを間違えた場合は「"POST /v3/auth/tokens HTTP/1.1" 401」が記録されます。
正常な応答がない場合は /var/log/apache2/keystone_error.log を確認しましょう。

```
...
10.0.0.101 - - [16/May/2016:15:54:49 +0900] "GET /v3 HTTP/1.1" 200 556 "-" "python-openstackclient keystoneauth1/2.4.0 python-requests/2.9.1 CPython/2.7.6"
10.0.0.101 - - [16/May/2016:15:54:49 +0900] "POST /v3/auth/tokens HTTP/1.1" 201 1608 "-" "python-openstackclient keystoneauth1/2.4.0 python-requests/2.9.1 CPython/2.7.6"
```

+ demoユーザーとして管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2016-05-17T02:35:04.100816Z      |
| id         | gAAAAABXOnVIT0AYCEj0J_uWhI86I... |
| project_id | 4e00692779864ed185356843ebeb0e2f |
| user_id    | a5fab4bc7a0645cd85e68af50c02d2fe |
+------------+----------------------------------+
```
<!-- BREAK -->

## 4. Glanceのインストールと設定

### 4-1 データベースを作成

MariaDBに glance で使用するデータベースを作成し、アクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
 IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 4-2 データベースの確認

MariaDBに glance ユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u glance -p
Enter password: ← MariaDBのglanceパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
2 rows in set (0.00 sec)
```

<!-- BREAK -->

### 4-3 ユーザー、サービス、APIエンドポイントの作成

以下のコマンドで認証情報を読み込み、ImageサービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

admin-openrc.shを読み込むと次のようにプロンプトが変化します。

```
controller# source admin-openrc.sh
controller ~(admin)#
```

+ glanceユーザーの作成

```
controller# openstack user create --domain default --password-prompt glance
User Password: password  #glanceユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | 76ef779bd772488ab6e8d2984d184f69 |
| name      | glance                           |
+-----------+----------------------------------+
```

+ adminロールをglanceユーザーとserviceプロジェクトに追加

```
controller# openstack role add --project service --user glance admin
```
<!-- BREAK -->

+ glanceサービスの作成

```
controller# openstack service create --name glance \
--description "OpenStack Image service" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image service          |
| enabled     | True                             |
| id          | 746ddec3462a4075845dfc76fb05f09f |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

+ APIエンドポイントの作成

```
controller# openstack endpoint create --region RegionOne \
  image public http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image internal http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```
<!-- BREAK -->

### 4-4 Glanceのインストール

apt-getコマンドでglance パッケージをインストールします。

```
controller# apt-get install glance
```

### 4-5 Glanceの設定変更

Glanceの設定を行います。glance-api.conf、glance-registry.confともに、[keystone_authtoken]に追記した設定以外のパラメーターはコメントアウトします。

```
controller# vi /etc/glance/glance-api.conf
...

[database]
#sqlite_db = /var/lib/glance/glance.sqlite         ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記
...
[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/  ← 追記
...
[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password  ← glanceユーザーのパスワード
...
[paste_deploy]
...
flavor = keystone          ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-api.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

```
controller# vi /etc/glance/glance-registry.conf

[DEFAULT]
...
[database]
#sqlite_db = /var/lib/glance/glance.sqlite             ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@controller/glance  ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password      ← glanceユーザーのパスワード

...
[paste_deploy]
flavor = keystone                ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-registry.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 4-6 データベースに展開

次のコマンドでglanceデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "glance-manage db_sync" glance
```

※ 廃止予定に関するメッセージが出力されますが全て無視してください

<!-- BREAK -->

### 4-7 Glanceサービスの再起動

設定を反映させるためGlanceサービスを再起動します。

```
controller# service glance-registry restart && service glance-api restart
```

### 4-8 ログの確認と使用しないデータベースファイルの削除

サービスの再起動後、ログを参照しGlance RegistryとGlance APIサービスでエラーが起きていないことを確認します。

```
controller# tailf /var/log/glance/glance-api.log
controller# tailf /var/log/glance/glance-registry.log
```

Glanceのインストール直後はGlanceのログに「glance.store.swift.Storeを読みこむことができなかった」といったエラーが出ることがありますが、本書の例に従った場合はGlanceのバックエンドとしてSwiftストレージは使わないため、無視して構いません。正しく設定が行われるとエラー出力はされなくなります。

```
ERROR stevedore.extension [-] Could not load 'glance.store.swift.Store': No module named swiftclient
ERROR stevedore.extension [-] No module named swiftclient
```

インストール直後は作られていない場合が多いですが、コマンドを実行してglance.sqliteを削除します。

```
controller# rm /var/lib/glance/glance.sqlite
```

### 4-9 イメージの取得と登録

Glanceへインスタンス用の仮想マシンイメージを登録します。ここでは、OpenStackのテスト環境に役立つ軽量なLinuxイメージ CirrOS を登録します。

#### 4-9-1 イメージの取得

CirrOSのWebサイトより仮想マシンイメージをダウンロードします。

```
controller# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

<!-- BREAK -->

#### 4-9-2 イメージを登録

ダウンロードした仮想マシンイメージをGlanceに登録します。

```
controller# glance image-create --name "cirros-0.3.4-x86_64" \
 --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 \
 --container-format bare \
 --visibility public --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2016-05-17T04:35:24Z                 |
| disk_format      | qcow2                                |
| id               | 12d67808-c7d2-44b4-8bc6-b0ced1878f06 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64                  |
| owner            | 360855029dbe44649a5dc862c5a3caf1     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2016-05-17T04:35:25Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

#### 4-9-3 イメージの登録を確認

仮想マシンイメージが正しく登録されたか確認します。

```
controller# openstack image list
+--------------------------------------+---------------------+--------+
| ID                                   | Name                | Status |
+--------------------------------------+---------------------+--------+
| 12d67808-c7d2-44b4-8bc6-b0ced1878f06 | cirros-0.3.4-x86_64 | active |
+--------------------------------------+---------------------+--------+
```

<!-- BREAK -->

## 5. Novaのインストールと設定（コントローラーノード）

### 5-1 データベースを作成

MariaDBにデータベースnovaを作成します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE nova_api;
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password:           ← MariaDBのrootパスワードpasswordを入力
```

### 5-2 データベースの確認

MariaDBにnovaユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u nova -p
Enter password: ← MariaDBのnovaパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
| nova_api           |
+--------------------+
3 rows in set (0.00 sec)

```

<!-- BREAK -->

### 5-3 ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc.sh
```

+ novaユーザーの作成

```
controller# openstack user create --domain default --password-prompt nova
User Password: password  #novaユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | e14bbac223e546559251f7e607693316 |
| name      | nova                             |
+-----------+----------------------------------+
```

+ novaユーザーをadminロールに追加

```
controller# openstack role add --project service --user nova admin
```

+ novaサービスの作成

```
controller# openstack service create --name nova \
  --description "OpenStack Compute" compute

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 87d0a9fec6244c37902694b2ae3143a4 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

<!-- BREAK -->

+ ComputeサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```

### 5-4 パッケージのインストール

apt-getコマンドでNova関連のパッケージをインストールします。

```
controller# apt-get install nova-api nova-conductor nova-consoleauth \
 nova-novncproxy nova-scheduler
```

<!-- BREAK -->

### 5-5 Novaの設定変更

nova.confに下記の設定を追記します。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
#lock_path=/var/lock/nova     ← コメントアウト
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
#enabled_apis=ec2,osapi_compute,metadata   ←コメントアウト
enabled_apis = osapi_compute,metadata        ←以下追記
rpc_backend = rabbit
auth_strategy = keystone

# コントローラーノードのIPアドレス:10.0.0.101
my_ip = 10.0.0.101                          ←追記

use_neutron = True

firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
vncserver_listen = 10.0.0.101               ←追記
vncserver_proxyclient_address = 10.0.0.101  ←自ホストを指定

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/nova/nova.confの続き）
...
(↓これ以下追記↓)

[api_database]
connection = mysql+pymysql://nova:password@controller/nova_api

[database]
connection = mysql+pymysql://nova:password@controller/nova

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password  ←novaユーザーのパスワード

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 5-6 データベースに展開

次のコマンドでnovaデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "nova-manage api_db sync" nova
controller# su -s /bin/sh -c "nova-manage db sync" nova
```

### 5-7 Novaサービスの再起動

設定を反映させるため、Novaのサービスを再起動します。

```
controller# service nova-api restart && service nova-consoleauth restart && service nova-scheduler restart && \
service nova-conductor restart && service nova-novncproxy restart
```

### 5-8 不要なデータベースファイルの削除

データベースはMariaDBを使用するため、不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/nova/nova.sqlite
```

### 5-9 Glanceとの通信確認

NovaのコマンドラインインターフェースでGlanceと通信してGlanceと相互に通信できているかを確認します。

```
controller# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| 12d67808-c7d2-44b4-8bc6-b0ced1878f06 | cirros-0.3.4-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```

※Glanceに登録したCirrOSイメージが表示できれば問題ありません。

<!-- BREAK -->

## 6. Nova Computeのインストールと設定（コンピュートノード）

ここまでコントローラーノードの環境構築を行ってきましたが、ここでコンピュートノードに切り替えて設定を行います。

### 6-1 パッケージのインストール

```
compute# apt-get install -y nova-compute
```

### 6-2 Novaの設定を変更

novaの設定ファイルを変更します。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
#lock_path=/var/lock/nova   ←コメントアウト
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone

my_ip = 10.0.0.102  ←IPアドレスで指定

use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/nova/nova.confの続き）
[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 10.0.0.102  ←自ホストを指定
novncproxy_base_url = http://10.0.0.101:6080/vnc_auto.html ←novncホストを指定
vnc_keymap = ja                             ←日本語キーボードの設定

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password  ←novaユーザーのパスワード

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```
<!-- BREAK -->

まず次のコマンドを実行し、コンピュートノードでLinux KVMが動作可能であることを確認します。コマンド結果が1以上の場合は、CPU仮想化支援機能がサポートされています。
もしこのコマンド結果が0の場合は仮想化支援機能がサポートされていないか、設定が有効化されていないので、libvirtでKVMの代わりにQEMUを使用します。後述の/etc/nova/nova-compute.confの設定でvirt_type = qemu を設定します。

```
compute# cat /proc/cpuinfo |egrep 'vmx|svm'|wc -l
4
```
VMXもしくはSVM対応CPUの場合はvirt_type = kvmと設定することにより、仮想化部分のパフォーマンスが向上します。

```
compute# vi /etc/nova/nova-compute.conf

[libvirt]
...
virt_type = kvm
```

### 6-3 Novaコンピュートサービスの再起動

設定を反映させるため、Nova-Computeサービスを再起動します。

```
compute# service nova-compute restart
```

<!-- BREAK -->

### 6-4 コントローラーノードとの疎通確認

疎通確認はコントローラーノード上にて、admin環境変数設定ファイルを読み込んで行います。

```
controller# source admin-openrc.sh
```

#### 6-4-1 ホストリストの確認

コントローラーノードとコンピュートノードが相互に接続できているか確認します。もし、StateがXXXになっているサービスがある場合は、該当のサービスのログを確認して対処してください。

```
controller# openstack compute service list -c Binary -c Host -c State
+------------------+----------------+-------+
| Binary           | Host           | State |
+------------------+----------------+-------+
| nova-consoleauth | controller     | up    |
| nova-scheduler   | controller     | up    | ←Novaのステータスを確認
| nova-conductor   | controller     | up    |
| nova-compute     | compute        | up    |
+------------------+----------------+-------+
```
※一覧にcomputeが表示されていれば問題ありません。Stateがupでないサービスがある場合は-cオプションを外して確認します。


#### 6-4-2 ハイパーバイザの確認

コントローラーノードよりコンピュートノードのハイパーバイザが取得可能か確認します。

```
controller# openstack hypervisor list
+----+---------------------+
| ID | Hypervisor Hostname |
+----+---------------------+
|  1 | compute             |
+----+---------------------+
```

※Hypervisor一覧にcomputeが表示されていれば問題ありません。

<!-- BREAK -->

## 7. Neutronのインストール・設定（コントローラーノード）

### 7-1 データベースを作成

MariaDBにデータベースneutronを作成し、アクセス権を付与します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ←MariaDBのrootパスワードpasswordを入力
```

### 7-2 データベースの確認

MariaDBにneutronユーザーでログインし、データベースの閲覧が可能か確認します。

```
controller# mysql -u neutron -p
Enter password: ←MariaDBのneutronパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
2 rows in set (0.00 sec)
```

※ユーザーneutronでログイン可能でデータベースが閲覧可能なら問題ありません。

<!-- BREAK -->

### 7-3 neutronユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、neutronサービスの作成とAPIエンドポイントを設定します。

+ 環境変数ファイルの読み込み

```
controller# source admin-openrc.sh
```

+ neutronユーザーの作成

```
controller# openstack user create --domain default --password-prompt neutron
User Password: password  #neutronユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | 3ff553c90945412fbbdc5cc43110b308 |
| name      | neutron                          |
+-----------+----------------------------------+

```

+ neutronユーザーをadminロールに追加

```
controller# openstack role add --project service --user neutron admin
```

<!-- BREAK -->

+ neutronサービスの作成

```
controller# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 45e2fd18f80f465fabaf5dfa52742253 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

```

+ neutronサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  network public http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network internal http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

### 7-4 パッケージのインストール

本書ではネットワークの構成は公式マニュアルの「[Networking Option 2: Self-service networks](http://docs.openstack.org/mitaka/install-guide-ubuntu/neutron-controller-install-option2.html)」の方法で構築する例を示します。

```
controller# apt-get install neutron-server neutron-plugin-ml2 \
 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
 neutron-metadata-agent
```

<!-- BREAK -->

### 7-5 Neutronコンポーネントの設定

+ Neutronサーバーの設定

```
controller# vi /etc/neutron/neutron.conf 

[DEFAULT]
...
core_plugin = ml2             ←確認
service_plugins = router      ←追記
allow_overlapping_ips = True  ←追記

rpc_backend = rabbit          ←コメントをはずす
auth_strategy = keystone      ←コメントをはずす
notify_nova_on_port_status_changes = True   ←コメントをはずす
notify_nova_on_port_data_changes = True     ←コメントをはずす

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite  ←既存設定をコメントアウト
connection = mysql+pymysql://neutron:password@controller/neutron  ←追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password  ←neutronユーザーのパスワード

[nova]（以下末尾に追記）
...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password  ←novaユーザーのパスワード

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/neutron/neutron.confの続き）

[oslo_messaging_rabbit]（以下追記）
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password
```

[keystone_authtoken]セクションは追記した設定以外は取り除くかコメントアウトしてください。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

+ ML2プラグインの設定

```
controller# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,vxlan           ←追記
tenant_network_types = vxlan             ←追記
mechanism_drivers = linuxbridge,l2population   ←追記 ※
extension_drivers = port_security              ←追記

[ml2_type_flat]
...
flat_networks = provider                   ←追記

[ml2_type_vxlan]
...
vni_ranges = 1:1000                      ←追記

[securitygroup]
...                                                     
enable_ipset = True                      ←コメントをはずす
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ Linuxブリッジエージェントの設定

パブリックネットワークに接続している側のNICを指定します。本書ではeth0を指定します。

```
controller# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:eth0 ←追記
```

local_ipは、先にphysical_interface_mappingに設定したNIC側のIPアドレスを設定します。

```
[vxlan]
enable_vxlan = True                        ←コメントをはずす
local_ip = 10.0.0.101                      ←追記
l2_population = True                       ←追記 ※
```

エージェントとセキュリティグループの設定を行います。

```
[securitygroup]
...
enable_security_group = True        ←コメントをはずす
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver 
↑ 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```
<!-- BREAK -->

#### ※ ML2プラグインのl2populationについて
OpenStack Mitakaの公式手順書では、ML2プラグインのmechanism_driversとしてlinuxbridgeとl2populationが指定されています。l2populationが有効だとこれまでの動きと異なり、インスタンスが起動してもネットワークが有効化されるまで通信ができません。つまりNeutronネットワークを構築してルーターのパブリック側のIPアドレス宛にPingコマンドを実行して確認できても疎通ができません。このネットワーク有効化の有無についてメッセージキューサービスが監視して制御しています。 <br />
従ってOpenStack Mitakaでは、これまでのバージョン以上にメッセージキューサービス（本例や公式手順ではしばしばRabbitMQが使用されます）が確実に動作している必要があります。このような仕組みが導入されたのは、不要なパケットがネットワーク内に流れ続けないようにするためです。<br />
ただし、弊社で構築した環境においては起動時間が経過するとArpリクエストを返さなくなり、結果Neutronネットワーク内と外との接続に支障が出ることが判明したため、公式のインストールガイドではl2populationを利用していますがこれをオフにして対処しています。

+ controllerの/etc/neutron/plugins/ml2/ml2_conf.iniの設定変更

```
[ml2]
...
mechanism_drivers = linuxbridge,l2population
↓
mechanism_drivers = linuxbridge
```

+ controller & computeの/etc/neutron/plugins/ml2/linuxbridge_agent.iniの設定変更

```
[vxlan]
...
l2_population = true
↓
l2_population = false
```
<!-- BREAK -->

+ Layer-3エージェントの設定

external_network_bridgeは単一のエージェントで複数の外部ネットワークを有効にするには値を指定してはならないため、値を空白にします。

```
controller# vi /etc/neutron/l3_agent.ini

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
```

+ DHCPエージェントの設定

```
controller# vi /etc/neutron/dhcp_agent.ini 

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

+ dnsmasqの設定

一般的にデフォルトのイーサネットのMTUは1500に設定されています。通常のEthernet フレームにVXLANヘッダが加算されるため、VXLANを使う場合は少なくとも50バイト多い、1550バイト以上のMTUが設定されていないと通信が不安定になったり、通信が不可能になる場合があります。<br />
これらはジャンボフレームを設定することで約9000バイトまでのMTUをサポートできるようになり対応可能ですが、ジャンボフレーム非対応のネットワーク機器を使う場合や、ネットワーク機器の設定を変更できない場合はVXLANの50バイトのオーバーヘッドを考慮して1450バイト以内のMTUに設定する必要があります。<br />
これらの制約事項はOpenStack環境でも同様で、インスタンスを起動する際にMTU 1450を設定することで、この問題を回避可能です。<br />
この設定はインスタンス起動毎にUserDataを使って設定することも可能ですが、次のように設定しておくと仮想DHCPサーバーでMTUの自動設定を行うことができるので便利です。

+ DHCPエージェントにdnsmasqの設定を追記

```
controller# vi /etc/neutron/dhcp_agent.ini 

[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf  ←追記
```

<!-- BREAK -->

+ DHCPオプションの26番(MTU)を定義

```
controller# vi /etc/neutron/dnsmasq-neutron.conf
dhcp-option-force=26,1450
```

+ Metadataエージェントの設定

インスタンスのメタデータサービスを提供するMetadataエージェントを設定します。

```
controller# vi /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_ip = controller  ←Metadataホストを指定
metadata_proxy_shared_secret = METADATA_SECRET
```

Metadataエージェントの`metadata_proxy_shared_secret`に指定する値と、次の手順でNovaに設定する`metadata_proxy_shared_secret`が同じ値になるように設定します。任意の値を設定すれば良いですが、思いつかない場合は次のように実行して生成した乱数を使うことも可能です。

```
controller# openssl rand -hex 10
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/metadata_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 7-6 Novaの設定を変更

Novaの設定ファイルにNeutronの設定を追記します。

```
controller# vi /etc/nova/nova.conf
...
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password   ←neutronユーザーのパスワード
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

METADATA_SECRETはMetadataエージェントで指定した値に置き換えます。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 7-7 データベースに展開

コマンドを実行して、エラーが発生せずに完了することを確認します。

```
controller# su -s /bin/sh -c "neutron-db-manage \
 --config-file /etc/neutron/neutron.conf \
 --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

No handlers could be found for logger "neutron.quota"
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL. 
...
INFO  [alembic.runtime.migration] Running upgrade kilo -> c40fbb377ad, Initial Liberty no-op script.
INFO  [alembic.runtime.migration] Running upgrade c40fbb377ad -> 4b47ea298795, add reject rule
  OK
```

<!-- BREAK -->

### 7-8 コントローラーノードのNeutronと関連サービスの再起動

設定を反映させるため、コントローラーノードの関連サービスを再起動します。
まずNova APIサービスを再起動します。

```
controller# service nova-api restart
```

次にNeutron関連サービスを再起動します。

```
controller# service neutron-server restart && \
 service neutron-linuxbridge-agent restart && \
 service neutron-dhcp-agent restart && \
 service neutron-metadata-agent restart && \
 service neutron-l3-agent restart
```

### 7-9 ログの確認

ログを確認して、エラーが出力されていないことを確認します。

```
controller# tailf /var/log/nova/nova-api.log
controller# tailf /var/log/neutron/neutron-server.log
controller# tailf /var/log/neutron/neutron-metadata-agent.log
controller# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

### 7-10 使用しないデータベースファイルを削除

```
controller# rm /var/lib/neutron/neutron.sqlite
```

<!-- BREAK -->

## 8. Neutronのインストール・設定（コンピュートノード）

次にコンピュートノードの設定を行います。

### 8-1 パッケージのインストール

```
compute# apt-get install neutron-linuxbridge-agent
```

### 8-2 設定の変更

+ Neutronの設定

```
compute# vi /etc/neutron/neutron.conf

[DEFAULT]
...
rpc_backend = rabbit                  ←コメントをはずす
auth_strategy = keystone              ←コメントをはずす

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password  ←neutronユーザーのパスワード

[database]
...
# connection = sqlite:////var/lib/neutron/neutron.sqlite   ← コメントアウト

[oslo_messaging_rabbit]
...
rabbit_host = controller           ←追記
rabbit_userid = openstack          ←追記
rabbit_password = password         ←追記
```

<!-- BREAK -->

本書の構成では、コンピュートノードのNeutron.confにはデータベースの指定は不要です。
次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

+ Linuxブリッジエージェントの設定

physical_interface_mappingsにはパブリック側のネットワークに接続しているインターフェイスを指定します。本書ではeth0を指定します。
local_ipにはパブリック側に接続しているNICに設定しているIPアドレスを指定します。

追記と書かれていない項目は設定があればコメントをはずして設定を変更、なければ追記してください。

```
compute# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:eth0

[vxlan]
enable_vxlan = True
local_ip = 10.0.0.102
l2_population = True  (※1)

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
↑追記
```

※1 ML2プラグインの設定 /etc/neutron/plugins/ml2/ml2_conf.iniでl2populationを使用しない場合はl2_population = False を設定し無効化します。

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 8-3 コンピュートノードのネットワーク設定

Novaの設定ファイルの内容をNeutronを利用するように変更します。

```
compute# vi /etc/nova/nova.conf
...
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password  ←neutronユーザーのパスワード
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 8-4 コンピュートノードのNeutronと関連サービスを再起動

ネットワーク設定を反映させるため、コンピュートノードのNeutronと関連のサービスを再起動します。

```
compute# service nova-compute restart 
compute# service neutron-linuxbridge-agent restart
```

### 8-5 ログの確認

エラーが出ていないかログを確認します。

```
compute# tailf /var/log/nova/nova-compute.log
compute# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

### 8-6 Neutronサービスの動作を確認

`neutron agent-list`コマンドを実行してNeutronエージェントが正しく認識されており、稼働していることを確認します。

```
controller# source admin-openrc.sh
controller# neutron agent-list -c host -c alive -c binary
+------------+-------+---------------------------+
| host       | alive | binary                    |
+------------+-------+---------------------------+
| controller | :-)   | neutron-metadata-agent    |
| controller | :-)   | neutron-linuxbridge-agent |
| controller | :-)   | neutron-dhcp-agent        |
| controller | :-)   | neutron-l3-agent          |
| compute    | :-)   | neutron-linuxbridge-agent |
+------------+-------+---------------------------+

```

 ※コントローラーとコンピュートで追加され、neutron-linuxbridge-agentが正常に稼働していることが確認できれば問題ありません。念のためログも確認してください。

<!-- BREAK -->

## 9. 仮想ネットワーク設定（コントローラーノード）

OpenStack Neutron環境ができたので、OpenStack内で利用するネットワークを作成します。ネットワークは外部ネットワークと接続するためのパブリックネットワークと、インスタンス間やルーター、内部DHCPサーバー間の通信に利用するインスタンス用ネットワークの二つを作成します。

パブリックネットワークは既存のネットワークから一定の範囲のネットワークをOpenStackに割り当てます。ネットワークのゲートウェイIPアドレス、IPアドレスのセグメントと割り当てるIPアドレスの範囲を決めておく必要があります。例えば192.168.1.0/24というネットワークであればゲートウェイIPアドレスは192.168.1.1か192.168.1.254がよく使われ、Windowsならipconfigコマンド、LinuxやUnixではifconfigコマンドで確認できます。パブリックネットワーク用に割り当てるIPアドレスの範囲については、そのネットワークでDHCPサーバーが動いている場合はDHCPサーバーが配る**IPアドレスの範囲を除いた**ネットワークを切り出して利用するようにしてください。

インスタンスにはインスタンス用ネットワークの範囲のIPアドレスがDHCPサーバーからDHCP Agentを介して割り当てられます。このインスタンスにパブリックネットワークの範囲からFloating IPアドレスを割り当てることで、NAT接続でインスタンスが外部ネットワークとやり取りができるようになります。

### 9-1 パブリックネットワークの設定

#### 9-1-1 admin環境変数ファイルの読み込み

まずはFloating IPアドレス割当用のネットワークである、パブリックネットワークをadmin権限で作成するためにadmin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```
<!-- BREAK -->

#### 9-1-2 パブリックネットワークの作成

ext-netと言う名前でパブリックネットワークを作成します。`provider:physical_network`で指定する設定は`/etc/neutron/plugins/ml2/linuxbridge_agent.ini`の`physical_interface_mappings`に指定した値を設定します。例えばprovider:eth0と設定した場合はproviderを指定します。

```
controller(admin)# neutron net-create ext-net --router:external \
 --provider:physical_network provider --provider:network_type flat
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| created_at                | 2016-05-18T06:41:05                  |
| description               |                                      |
| id                        | 6d6bee75-55ec-41c4-b738-8409ee051149 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| is_default                | False                                |
| mtu                       | 1500                                 |
| name                      | ext-net                              |
| port_security_enabled     | True                                 |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | 360855029dbe44649a5dc862c5a3caf1     |
| updated_at                | 2016-05-18T06:41:05                  |
+---------------------------+--------------------------------------+

```

<!-- BREAK -->

#### 9-1-3 パブリックネットワーク用サブネットの作成

ext-subnetという名前でパブリックネットワーク用サブネットを作成します。`allocation-pool`にはFloatingIPアドレスとして利用するネットワークの範囲、`gateway`には指定した範囲のネットワークのゲートウェイIPアドレスとネットワークセグメントを指定します。

```
controller(admin)# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=10.0.0.200,end=10.0.0.250 \
  --disable-dhcp --gateway 10.0.0.1 10.0.0.0/24

Created a new subnet:
+-------------------+----------------------------------------------+
| Field             | Value                                        |
+-------------------+----------------------------------------------+
| allocation_pools  | {"start": "10.0.0.200", "end": "10.0.0.250"} |
| cidr              | 10.0.0.0/24                                  |
| created_at        | 2016-05-18T06:51:01                          |
| description       |                                              |
| dns_nameservers   |                                              |
| enable_dhcp       | False                                        |
| gateway_ip        | 10.0.0.1                                     |
| host_routes       |                                              |
| id                | 6c74fe33-47ac-4c28-b396-15b38b1b602f         |
| ip_version        | 4                                            |
| ipv6_address_mode |                                              |
| ipv6_ra_mode      |                                              |
| name              | ext-subnet                                   |
| network_id        | 6d6bee75-55ec-41c4-b738-8409ee051149         |
| subnetpool_id     |                                              |
| tenant_id         | 360855029dbe44649a5dc862c5a3caf1             |
| updated_at        | 2016-05-18T06:51:01                          |
+-------------------+----------------------------------------------+

```

### 9-2 インスタンス用ネットワークの設定

#### 9-2-1 demoユーザーと環境変数ファイルの読み込み

次にインスタンス用ネットワークを作成します。インスタンス用ネットワークを作成するためにdemo環境変数を読み込みます。

```
controller# source demo-openrc.sh
```

<!-- BREAK -->

#### 9-2-2 インスタンス用ネットワークの作成

demo-netという名前でインスタンス用ネットワークを作成します。

```
controller(demo)# neutron net-create demo-net
Created a new network:
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | True                                 |
| created_at              | 2016-05-18T06:52:08                  |
| description             |                                      |
| id                      | 8717f502-f07e-4b2f-84fe-b51833db18fb |
| ipv4_address_scope      |                                      |
| ipv6_address_scope      |                                      |
| mtu                     | 1450                                 |
| name                    | demo-net                             |
| port_security_enabled   | True                                 |
| router:external         | False                                |
| shared                  | False                                |
| status                  | ACTIVE                               |
| subnets                 |                                      |
| tags                    |                                      |
| tenant_id               | 4e00692779864ed185356843ebeb0e2f     |
| updated_at              | 2016-05-18T06:52:08                  |
+-------------------------+--------------------------------------+

```
<!-- BREAK -->

#### 9-2-3 インスタンス用ネットワークのサブネットを作成

demo-subnetという名前でインスタンス用ネットワークサブネットを作成します。

`gateway`には指定したインスタンス用ネットワークのサブネットの範囲から任意のIPアドレスを指定します。第4オクテットとして1を指定したIPアドレスを設定するのが一般的です。ここでは192.168.0.0/24のネットワークをインスタンス用ネットワークとして定義し、ゲートウェイIPとして192.168.0.1を設定しています。

`dns-nameserver`には外部ネットワークに接続する場合に名前引きするためのDNSサーバーを指定します。ここではGoogle Public DNSの`8.8.8.8`を指定していますが、外部の名前解決ができるDNSサーバーであれば何を指定しても構いません。

インスタンス用ネットワーク内でDHCPサーバーが稼働し、インスタンスが起動した時にそのDHCPサーバーが`dns-nameserver`に指定したDNSサーバーと192.168.0.0/24の範囲からIPアドレスを確保してインスタンスに割り当てます。

```
controller(demo)# neutron subnet-create demo-net 192.168.0.0/24 \
--name demo-subnet --gateway 192.168.0.1 --dns-nameserver 8.8.8.8
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr              | 192.168.0.0/24                                   |
| created_at        | 2016-05-18T06:52:57                              |
| description       |                                                  |
| dns_nameservers   | 8.8.8.8                                          |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.0.1                                      |
| host_routes       |                                                  |
| id                | f5e3f465-38c7-4915-abc5-6f966412c330             |
| ip_version        | 4                                                |
| name              | demo-subnet                                      |
| network_id        | 8717f502-f07e-4b2f-84fe-b51833db18fb             |
| subnetpool_id     |                                                  |
| tenant_id         | 4e00692779864ed185356843ebeb0e2f                 |
| updated_at        | 2016-05-18T06:52:57                              |
+-------------------+--------------------------------------------------+

```

<!-- BREAK -->

### 9-3 仮想ネットワークルーターの設定

仮想ネットワークルーターを作成して外部接続用ネットワークとインスタンス用ネットワークをルーターに接続し、双方でデータのやり取りを行えるようにします。

#### 9-3-1 demo-routerを作成

仮想ネットワークルーターを作成します。

```
controller(demo)# neutron router-create demo-router
Created a new router:
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | True                                 |
| description             |                                      |
| external_gateway_info   |                                      |
| id                      | 92a5c675-94e2-4894-b377-d7ccea194fca |
| name                    | demo-router                          |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tenant_id               | 4e00692779864ed185356843ebeb0e2f     |
+-------------------------+--------------------------------------+

```

#### 9-3-2 demo-routerにサブネットを追加

仮想ネットワークルーターにインスタンス用ネットワークを接続します。

```
controller(demo)# neutron router-interface-add demo-router demo-subnet
Added interface 5886789d-5093-4c56-a396-2012dba2381c to router demo-router.
```

#### 9-3-3 demo-routerにゲートウェイを追加

仮想ネットワークルーターに外部ネットワークを接続します。

```
controller(demo)# neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router
```

<!-- BREAK -->

### 9-4 ネットワークの確認

- 仮想ルーターのゲートウェイIPアドレスの確認

`neutron router-port-list`コマンドを実行すると、仮想ルーターのそれぞれのポートに割り当てられたIPアドレスを確認することができます。Set gateway for router demo-router
コマンドの実行結果から192.168.0.1がインスタンスネットワーク側ゲートウェイIPアドレス、10.0.0.200がパブリックネットワーク側ゲートウェイIPアドレスであることがわかります。

作成したネットワークの確認のために、外部PCからパブリックネットワーク側ゲートウェイIPアドレスにpingを飛ばしてみましょう。問題なければ仮想ルーターと外部ネットワークとの接続ができていると判断することができます。

```
controller(admin)# source admin-openrc.sh
controller(admin)# neutron router-port-list demo-router -c fixed_ips --max-width 30
+--------------------------------+
| fixed_ips                      |
+--------------------------------+
| {"subnet_id": "f5e3f465-38c7-4 |
| 915-abc5-6f966412c330",        |
| "ip_address": "192.168.0.1"}   |
| {"subnet_id": "6c74fe33-47ac-  |
| 4c28-b396-15b38b1b602f",       |
| "ip_address": "10.0.0.200"}    |
+--------------------------------+

# ping -c3 10.0.0.200|grep "packet loss"
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
(ルーターゲートウェイ宛に各ノードからpingコマンドの実行)
```

※応答が返ってくれば問題ありません。

<!-- BREAK -->

コントローラーノードで次のようにコマンドを実行すると、従来のOpen vSwitchのようにlinuxbridgeの中をのぞくことができます。仮想ルーターと仮想DHCPサーバーの状態を確認できます。接続がうまくいかない場合におためしください。

```
controller# ip netns
qrouter-92a5c675-94e2-4894-b377-d7ccea194fca
qdhcp-8717f502-f07e-4b2f-84fe-b51833db18fb

(仮想ルーターと仮想DHCPサーバーを確認)
...
controller# ip netns exec `ip netns|grep qrouter*` bash
(qrouterにログイン)
...
controller# ip -f inet addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: qr-5886789d-50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000

    inet 192.168.0.1/24 brd 192.168.0.255 scope global qr-5886789d-50

       valid_lft forever preferred_lft forever
3: qg-c39b9d87-8e: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000

    inet 10.0.0.200/24 brd 10.0.0.255 scope global qg-c39b9d87-8e

       valid_lft forever preferred_lft forever
(ネットワークデバイスとIPアドレスを表示)
...
controller# ping 8.8.8.8 -I qg-c39b9d87-8e
(Public側から外にアクセスできることを確認)
 
```
<!-- BREAK -->

### 9-5 インスタンスの起動を確認

起動イメージ、コンピュート、NeutronネットワークといったOpenStackの最低限の構成ができあがったので、ここでOpenStack環境がうまく動作しているか確認しましょう。
まずはコマンドを使ってインスタンスを起動するために必要な情報を集める所から始めます。環境設定ファイルを読み込んで、各コマンドを実行し、情報を集めてください。

```
controller# source demo-openrc.sh
controller(demo)# openstack image list
(起動イメージ一覧を表示)
+--------------------------------------+---------------------+--------+
| ID                                   | Name                | Status |
+--------------------------------------+---------------------+--------+
| 12d67808-c7d2-44b4-8bc6-b0ced1878f06 | cirros-0.3.4-x86_64 | active |
+--------------------------------------+---------------------+--------+

controller(demo)# openstack network list -c ID -c Name
(ネットワーク一覧を表示)
+--------------------------------------+----------+
| ID                                   | Name     |
+--------------------------------------+----------+
| 6d6bee75-55ec-41c4-b738-8409ee051149 | ext-net  |
| 8717f502-f07e-4b2f-84fe-b51833db18fb | demo-net |
+--------------------------------------+----------+

controller(demo)# openstack security group list -c ID -c Name
(セキュリティグループ一覧を表示)
+--------------------------------------+---------+
| ID                                   | Name    |
+--------------------------------------+---------+
| 77c03d2e-0898-4e5a-b93a-8bf0c5f26ee2 | default |
+--------------------------------------+---------+

controller(demo)# openstack flavor list -c Name -c Disk
(フレーバー一覧を表示)
+-----------+------+
| Name      | Disk |
+-----------+------+
| m1.tiny   |    1 |
| m1.small  |   20 |
| m1.medium |   40 |
| m1.large  |   80 |
| m1.xlarge |  160 |
+-----------+------+

```

nova bootコマンドを使って、インスタンスを起動します。正常に起動したらnova deleteコマンドでインスタンスを削除してください。

```
controller(demo)# nova boot --flavor m1.tiny --image "cirros-0.3.4-x86_64" \
 --nic net-id=8717f502-f07e-4b2f-84fe-b51833db18fb \
 --security-group 77c03d2e-0898-4e5a-b93a-8bf0c5f26ee2 vm1

(インスタンスを起動)

controller(demo)# watch nova list
(インスタンス一覧を表示)
+--------------------------------------+------+--------+------------+-------------+----------------------+
| ID                                   | Name | Status | Task State | Power State | Networks             |
+--------------------------------------+------+--------+------------+-------------+----------------------+
| 2bc42415-c6a7-45a2-b5ee-b865a26bf101 | vm1  | ACTIVE | -          | Running     | demo-net=192.168.0.3 |
+--------------------------------------+------+--------+------------+-------------+----------------------+

# grep "ERROR\|WARNING" /var/log/rabbitmq/*.log
# grep "ERROR\|WARNING" /var/log/neutron/*
# grep "ERROR\|WARNING" /var/log/nova/*
(各ノードの関連サービスでエラーが出ていないことを確認)

controller(demo)# nova delete vm1
Request to delete server vm1 has been accepted.
(起動したインスタンスを削除)
```

<!-- BREAK -->

## 10. Cinderのインストール（コントローラーノード）

### 10-1 データベースを作成

MariaDBのデータベースにCinderのデータベースを作成します。

```
controller# mysql -u root -p << EOF
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ←MariaDBのrootパスワードpasswordを入力
```

### 10-2 データベースの確認

MariaDBにCinderのデータベースが登録されたか確認します。

```
controller# mysql -u cinder -p
Enter password: ← MariaDBのcinderパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| cinder             |
+--------------------+
2 rows in set (0.00 sec)
```

※ユーザーcinderでログイン可能でデータベースの閲覧が可能なら問題ありません。

<!-- BREAK -->

### 10-3 Cinderサービスなどの作成

+ admin環境変数の読み込み

```
controller# source admin-openrc.sh
```

+ cinderユーザーの作成

```
controller# openstack user create --domain default --password-prompt cinder
User Password: password  ←cinderユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | c12fc3119bb046a09ba53e0566004f76 |
| enabled   | True                             |
| id        | 4b798308a76041d1a0dc86464ac1208a |
| name      | cinder                           |
+-----------+----------------------------------+
```

+ adminロールをcinderユーザーに追加

```
controller# openstack role add --project service --user cinder admin
```

<!-- BREAK -->

+ cinder および cinder v2サービスの作成

```
controller# openstack service create --name cinder \
--description "OpenStack Block Storage" volume
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | f04a5af43a774b33887e4e42a2478340 |
| name        | cinder                           |
| type        | volume                           |
+-------------+----------------------------------+

controller# openstack service create --name cinderv2 \
--description "OpenStack Block Storage" volumev2
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | be8bebb2f37345b3a1f14512193d86f6 |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+
```
<!-- BREAK -->

+ Block StorageサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
 volume public http://controller:8776/v1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
 volume internal http://controller:8776/v1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
 volume admin http://controller:8776/v1/%\(tenant_id\)s
```

``` 
controller# openstack endpoint create --region RegionOne \
   volumev2 public http://controller:8776/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
   volumev2 internal http://controller:8776/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
   volumev2 admin http://controller:8776/v2/%\(tenant_id\)s
```

### 10-4 パッケージのインストール

本書ではBlock StorageコントローラーとBlock Storageボリュームコンポーネントを一台のマシンで構築するため、両方の役割をインストールします。

```
controller# apt-get install -y lvm2 cinder-api cinder-scheduler cinder-volume python-mysqldb python-cinderclient 
```

<!-- BREAK -->

### 10-5 Cinderの設定を変更

```
controller# vi /etc/cinder/cinder.conf

[DEFAULT]
...
auth_strategy = keystone      ←確認
#lock_path = /var/lock/cinder ←コメントアウト

↓↓ 以下追記 ↓↓

rpc_backend = rabbit

my_ip = 10.0.0.101   #コントローラーノード
enabled_backends = lvm

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[database]
connection = mysql+pymysql://cinder:password@controller/cinder

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password       ←cinderユーザーのパスワード

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm
```
<!-- BREAK -->

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/cinder/cinder.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 10-6 データベースに展開

```
controller# su -s /bin/sh -c "cinder-manage db sync" cinder
```

### 10-7 使用しないデータベースファイルを削除

```
controller# rm /var/lib/cinder/cinder.sqlite
```

### 10-8 nova-api の設定変更

controllerノードの /etc/nova/nova.conf に以下を追記します。

```
controller# vi /etc/nova/nova.conf

(以下追記)

[cinder]
os_region_name = RegionOne

```

### 10-9 サービスの再起動

設定を反映させるために、Cinderのサービスを再起動します。

```
controller# service cinder-scheduler restart && service cinder-api restart && service nova-api restart
```

<!-- BREAK -->

### 10-10 イメージ格納用ボリュームの作成

イメージ格納用ボリュームを設定するために物理ボリュームの設定、ボリュームの作成を行います。

#### 10-11-1 物理ボリュームを追加

本書ではコントローラーノードにハードディスクを追加して、そのボリュームをCinder用ボリュームとして使います。コントローラーノードを一旦シャットダウンしてからハードディスクを増設し、再起動してください。新しい増設したディスクはdmesgコマンドなどを使って確認できます。

```
controller# dmesg |grep sd|grep "logical blocks"
[    1.361779] sd 2:0:0:0: [sda] 62914560 512-byte logical blocks: (32.2 GB/30.0 GiB)  ←システムディスク
[    1.362105] sd 2:0:1:0: [sdb] 33554432 512-byte logical blocks: (17.1 GB/16.0 GiB)  ←追加ディスク
```

仮想マシンにハードディスクを増設した場合は/dev/vdbなどのようにデバイス名が異なる場合があります。

#### 10-11-2 物理ボリュームを設定

以下コマンドで物理ボリュームを作成します。

+ LVM物理ボリュームの作成

```
controller# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
```

+ LVMボリュームグループの作成

```
controller# vgcreate cinder-volumes /dev/sdb
  Volume group "cinder-volumes" successfully created
```

+ /etc/lvm/lvm.confにデバイスを指定

```
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

<!-- BREAK -->

#### 10-11-3 Cinder-Volumeサービスの再起動

Cinderストレージの設定を反映させるために、Cinder-Volumeのサービスを再起動します。

```
controller# service cinder-volume restart && service tgt restart
```

#### 10-11-4 admin環境変数設定ファイルを読み込み

adminのみ実行可能なコマンドを実行するために、admin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```

#### 10-11-5 Cinderサービスの確認

以下コマンドでCinderサービスの一覧を表示し、正常に動作していることを確認します。

```
controller# cinder service-list

+------------------+----------------+------+---------+-------+------------------+
|      Binary      |      Host      | Zone |  Status | State |    Updated_at    |
+------------------+----------------+------+---------+--------------------------+
| cinder-scheduler |   controller   | nova | enabled |   up  | 2016-05-19T02:39 |
|  cinder-volume   | controller@lvm | nova | enabled |   up  | 2016-05-19T02:39 |
+------------------+----------------+------+---------+-------+------------------+

```
<!-- BREAK -->

#### 10-11-6 Cinderボリュームの作成

以下コマンドでCinderボリュームを作成し、正常にCinderが動作していることを確認します。

```
controller# openstack volume create --size 1 volume
(1GBのストレージを作成)
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2016-05-27T06:26:22.501800           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | cb0979fa-ea2e-4c52-a26a-c815c53a22ee |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volume                               |
| properties          |                                      |
| replication_status  | disabled                             |
| size                | 1                                    | ← 作成できた
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | None                                 |
| updated_at          | None                                 |
| user_id             | 197e779ea83e43d98197b5abe23de80c     |
+---------------------+--------------------------------------+
```

<!-- BREAK -->

## 11. Dashboardのインストールと確認（コントローラーノード）

クライアントマシンからブラウザーでOpenStack環境を操作可能なWebインターフェイスをインストールします。

### 11-1 パッケージのインストール

コントローラーノードにDashboardをインストールします。

```
controller# apt-get install openstack-dashboard
```

### 11-2 Dashboardの設定を変更

インストールしたDashboardの設定を変更します。

```
controller# vi /etc/openstack-dashboard/local_settings.py 

(以下変更)

...

OPENSTACK_HOST = "controller"    ← 変更
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST   ← 変更
ALLOWED_HOSTS = ['*', ] 

SESSION_ENGINE = ‘django.contrib.sessions.backends.cache'   ←追記
CACHES = {										←CACHESパラメーターのコメントを外す
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'controller:11211',	← 変更
    },
}

OPENSTACK_API_VERSIONS = {		←OPENSTACK_API_VERSIONSパラメーターのコメントを外す
    "identity": 3,	← 変更
    "volume": 2,		← 変更
    "compute": 2,		← 変更
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'  ← コメントを外す
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"  ← 変更
TIME_ZONE = "Asia/Tokyo"  ← 変更
```
<!-- BREAK -->

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/openstack-dashboard/local_settings.py  | grep -v "^\s*$" | grep -v "^\s*#"
```

念のため、リダイレクトするように設定しておきます（数字は待ち時間）。

```
controller# vi /var/www/html/index.html
...
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="refresh" content="3; url=/horizon" />    ← 追記
```

変更した変更を反映させるため、Apacheとセッションストレージサービスを再起動します。

```
controller# service apache2 restart
```

### 11-3 Dashboardにアクセス

コントローラーノードとネットワーク的に接続されているマシンからブラウザで以下URLに接続してOpenStackのログイン画面が表示されるか確認します。

※ブラウザで接続するマシンは予めDNSもしくは/etc/hostsにコントローラーノードのIPを記述しておく等コンピュートノードの名前解決を行っておく必要があります。

```
http://controller/horizon/
```

※上記URLにアクセスしてログイン画面が表示され、ユーザーadminとdemoでログイン（パスワード:password）でログインできれば問題ありません。

<!-- BREAK -->

### 11-4 セキュリティグループの設定

OpenStackの上で動かすインスタンスのファイアウォール設定は、セキュリティグループで行います。ログイン後、次の手順でセキュリティグループを設定できます。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」を選択<br>
3.「ルールの管理」ボタンをクリック<br>
4.「ルールの追加」で許可するルールを定義<br>
5.「追加」ボタンをクリック<br>

インスタンスに対してPingを実行したい場合はルールとしてすべてのICMPを、インスタンスにSSH接続したい場合はSSHをルールとしてセキュリティグループに追加してください。

セキュリティーグループは複数作成できます。作成したセキュリティーグループをインスタンスを起動する際に選択することで、セキュリティグループで定義したポートを解放したり、拒否したり、接続できるクライアントを制限することができます。

### 11-5 キーペアの作成

OpenStackではインスタンスへのアクセスはデフォルトで公開鍵認証方式で行います。次の手順でキーペアを作成できます。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」をクリック<br>
3.「キーペア」タブをクリック<br>
4.「キーペアの作成」ボタンをクリック<br>
5.キーペア名を入力<br>
6.「キーペアの作成」ボタンをクリック<br>
7.キーペア（拡張子:pem）ファイルをダウンロード<br>

<!-- BREAK -->

### 11-6 インスタンスの起動

前の手順でGlanceにCirrOSイメージを登録していますので、早速構築したOpenStack環境上でインスタンスを起動してみましょう。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→イメージ」をクリック<br>
3.イメージ一覧から起動するOSイメージを選び、「インスタンスの起動」ボタンをクリック<br>
4.「インスタンスの起動」詳細タブで起動するインスタンス名、フレーバー、インスタンス数を設定<br>
5.アクセスとセキュリティタブで割り当てるキーペア、セキュリティーグループを設定<br>
6.ネットワークタブで割り当てるネットワークを設定<br>
7.作成後タブで必要に応じてユーザーデータの入力（オプション）<br>
8.高度な設定タブでパーティションなどの構成を設定（オプション）<br>
9.右下の「起動」ボタンをクリック<br>

### 11-7 Floating IPの設定

起動したインスタンスにFloating IPアドレスを設定することで、Dashboardのコンソール以外からインスタンスにアクセスできるようになります。インスタンスにFloating IPを割り当てるには次の手順で行います。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→インスタンス」をクリック<br>
3.インスタンスの一覧から割り当てるインスタンスをクリック<br>
4.アクションメニューから「Floating IPの割り当て」をクリック<br>
5.「Floating IP割り当ての管理」画面のIPアドレスで「+」ボタンをクリック<br>
6.右下の「IPの確保」ボタンをクリック<br>
7.割り当てるIPアドレスとインスタンスを選択して右下の「割り当て」ボタンをクリック<br>

<!-- BREAK -->

### 11-8 インスタンスへのアクセス

Floating IPを割り当てて、かつセキュリティグループの設定を適切に行っていれば、リモートアクセスできるようになります。セキュリティーグループでSSHを許可した場合、端末からSSH接続が可能になります（下記は実行例）。

```
client$ ssh -i mykey.pem cloud-user@instance-floating-ip 
```

その他、適切なポートを開放してインスタンスへのPingを許可したり、インスタンスでWebサーバーを起動して外部PCからアクセスしてみましょう。

### 11-9 Ubuntuテーマの削除

Ubuntu ServerベースでOpenStack環境を構築した場合、OpenStack DashboardはUbuntuのテーマが設定されています。このテーマはOpenStack標準テーマと比べてフォントが大きく見やすくて良いのですが、一部のボタンなどでボタンラベルが表示されない問題が発生することがあります。

![MacDown logo](./images/ubuntu-theme.png)

次のように実行するとUbuntuのカスタムテーマを削除できます。必要に応じて対応してください。

```
$ sudo apt-get remove --auto-remove openstack-dashboard-ubuntu-theme
```
<!-- BREAK -->

# Part.2 OpenStack 監視編 (近日公開予定)

近日公開予定
