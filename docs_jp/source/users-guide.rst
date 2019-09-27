Fabric CA User's Guide
======================

Hyperledger Fabric CAは、Hyperledger Fabricの認証局（CA）です。

次のような機能を提供します。

  * IDの登録、またはユーザーレジストリとしてのLDAPへの接続
  * 登録証明書（ECerts）の発行
  * 証明書の更新と失効
  * Hyperledger Fabric CAは、このドキュメントで後述するように、サーバーコンポーネントとクライアントコンポーネントの両方で構成されています。

Hyperledger Fabric CAへの貢献に関心のある開発者については、`Fabric CA repository <https://github.com/hyperledger/fabric-ca>`__で詳細をご覧ください。

.. _Back to Top:

Table of Contents
-----------------

1. `Overview`_

2. `Getting Started`_

   1. `Prerequisites`_
   2. `Install`_
   3. `Explore the Fabric CA CLI`_

3. `Configuration Settings`_

   1. `A word on file paths`_

4. `Fabric CA Server`_

   1. `Initializing the server`_
   2. `Starting the server`_
   3. `Configuring the database`_
   4. `Configuring LDAP`_
   5. `Setting up a cluster`_
   6. `Setting up multiple CAs`_
   7. `Enrolling an intermediate CA`_
   8. `Upgrading the server`_
   9. `Operations Service`_

5. `Fabric CA Client`_

   1. `Enrolling the bootstrap identity`_
   2. `Registering a new identity`_
   3. `Enrolling a peer identity`_
   4. `Getting Identity Mixer credential`_
   5. `Getting Idemix CRI (Certificate Revocation Information)`_
   6. `Reenrolling an identity`_
   7. `Revoking a certificate or identity`_
   8. `Generating a CRL (Certificate Revocation List)`_
   9. `Attribute-Based Access Control`_
   10. `Dynamic Server Configuration Update`_
   11. `Enabling TLS`_
   12. `Contact specific CA instance`_

6. `HSM`_

   1. `Configuring Fabric CA server to use softhsm2`_

7. `File Formats`_

   1. `Fabric CA server's configuration file format`_
   2. `Fabric CA client's configuration file format`_

8. `Troubleshooting`_


Overview
--------

次の図は、Hyperledger Fabric CAサーバーがHyperledger Fabricアーキテクチャ全体にどのようにフィットしているかを示しています。

.. image:: ./images/fabric-ca.png

Hyperledger Fabric CAサーバーと対話する方法は2つあります。
HyperledgerFabric CAクライアントを介するやり方と、いくつか存在するFabric SDKのいずれかを介するやりかたです。
Hyperledger Fabric CAサーバーへのすべての通信は、REST APIを介して行われます。
これらのREST APIのswaggerドキュメントについては、 `fabric-ca/swagger/swagger-fabric-ca.json` を参照してください。
このドキュメントは、http://editor2.swagger.io オンラインエディターで表示できます。

Hyperledger Fabric CAクライアントまたはSDKは、Hyperledger Fabric CAサーバーのクラスターにあるサーバーに接続できます。
これは、図の右上のセクションに示されています。
クライアントはHAプロキシエンドポイントにルーティングします。HAプロキシエンドポイントは、ファブリックCAサーバークラスターメンバーの1つにトラフィックを負荷分散します。

クラスター内のすべてのHyperledger Fabric CAサーバーは、アイデンティティと証明書を追跡するために同じデータベースを共有します。
LDAPが構成されている場合、アイデンティティ情報はデータベースではなくLDAPに保持されます。

サーバーには複数のCAが含まれる場合があります。
各CAは、ルートCAまたは中間CAのいずれかです。
各中間CAには、ルートCAまたは別の中間CAである親CAがあります。

Getting Started
---------------

Prerequisites
~~~~~~~~~~~~~~~

前提条件

-  Go 1.10以降のインストール
-  ``GOPATH`` 環境変数が正しく設定されている
-  libtoolおよびlibtdhl-devパッケージがインストールされています

以下は、Ubuntuにlibtoolの依存関係をインストールします。

.. code:: bash

   sudo apt install libtool libltdl-dev

以下は、MacOSXにlibtoolの依存関係をインストールします。

.. code:: bash

   brew install libtool

.. note:: libtldl-dev is not necessary on MacOSX if you instal
          libtool via Homebrew

libtoolの詳細については、以下を参照してください。
https://www.gnu.org/software/libtool

libltdl-devの詳細については、以下を参照してください。
https://www.gnu.org/software/libtool/manual/html_node/Using-libltdl.html

Install
~~~~~~~

インストール

以下は、`fabric-ca-server` と `fabric-ca-client`の両方のバイナリを $GOPATH/bin にインストールします。

.. code:: bash

    go get -u github.com/hyperledger/fabric-ca/cmd/...

注：fabric-caリポジトリのクローンをすでに作成している場合は、上記の「go get」コマンドを実行する前にmasterブランチにいることを確認してください。
そうしないと、次のエラーが表示される場合があります。

::

    <gopath>/src/github.com/hyperledger/fabric-ca; git pull --ff-only
    There is no tracking information for the current branch.
    Please specify which branch you want to merge with.
    See git-pull(1) for details.

        git pull <remote> <branch>

    If you wish to set tracking information for this branch you can do so with:

        git branch --set-upstream-to=<remote>/<branch> tlsdoc

    package github.com/hyperledger/fabric-ca/cmd/fabric-ca-client: exit status 1

Start Server Natively
~~~~~~~~~~~~~~~~~~~~~

サーバーをネイティブで起動

以下は、デフォルト設定でfabric-ca-serverを開始します。

.. code:: bash

    fabric-ca-server start -b admin:adminpw

`-b` オプションは、ブートストラップ管理者の登録ID (enrollment ID) とシークレットを提供します。 
これは、LDAPが「ldap.enabled」設定で有効になっていない場合に必要です。

`fabric-ca-server-config.yaml`という名前のデフォルト設定ファイルが、ローカルディレクトリに作成され、これはカスタマイズできます。

Start Server via Docker
~~~~~~~~~~~~~~~~~~~~~~~

サーバーをDockerで起動

Docker Hub
^^^^^^^^^^^^

以下にアクセスします。
https://hub.docker.com/r/hyperledger/fabric-ca/tags/

pullするfabric CAのアーキテクチャとバージョンに一致するタグを見つけます。

`$GOPATH/src/github.com/hyperledger/fabric-ca/docker/server`に移動し、
エディターで docker-compose.yml を開きます。

docker-compose.yml の、`image` の行に、バージョンのtagを反映します。 
ベータ版のx86アーキテクチャでは、ファイルは次のようになります。

.. code:: yaml

    fabric-ca-server:
      image: hyperledger/fabric-ca:x86_64-1.0.0-beta
      container_name: fabric-ca-server
      ports:
        - "7054:7054"
      environment:
        - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      volumes:
        - "./fabric-ca-server:/etc/hyperledger/fabric-ca-server"
      command: sh -c 'fabric-ca-server start -b admin:adminpw'

docker-compose.yml ファイルと同じディレクトリでターミナルを開き、次を実行します。

.. code:: bash

    # docker-compose up -d

これにより、構成ファイルに指定された fabric-ca イメージがまだ存在しない場合は pull され、
fabric-ca サーバーのインスタンスが開始されます。

Building Your Own Docker image
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

自分用のDocker Imageをビルドする

以下に示すように、docker-composeを介してサーバーをビルドおよび起動できます。

.. code:: bash

    cd $GOPATH/src/github.com/hyperledger/fabric-ca
    make docker
    cd docker/server
    docker-compose up -d

hyperledger/fabric-ca の docker image には、fabric-ca-server と fabric-ca-client の両方が含まれています。

.. code:: bash

    # cd $GOPATH/src/github.com/hyperledger/fabric-ca
    # FABRIC_CA_DYNAMIC_LINK=true make docker
    # cd docker/server
    # docker-compose up -d

Explore the Fabric CA CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Fabric CA CLIを調べる

このセクションでは、Fabric CA サーバーとクライアントのシンプルなusage messageを便宜上提供します。
さらなる使用法については、次のセクションで説明します。

次のリンクは、:doc:`Server Command Line <servercli>`と:doc:`Client Command Line <clientcli>`を示しています。

.. note:: 文字列スライス（リスト）であるコマンドラインオプションは、コンマ区切りのリスト要素でオプションを指定するか、
          リストを構成する文字列値でオプションを複数回指定することで指定できます。
          たとえば、``csr.hosts`` オプションに ``host1`` と ``host2`` を指定するには、
          ``-csr.hosts 'host1, host2'`` または ``--csr.hosts host1 --csr.hosts host2`` を渡すことができます。
          1つめの形式を使用する場合は、コンマの前後にスペースがないことを確認してください。

`Back to Top`_

Configuration Settings
~~~~~~~~~~~~~~~~~~~~~~

構成設定

Fabric CAは、Fabric CAサーバーとクライアントの設定を構成する3つの方法を提供します。 
優先順位は次のとおりです。

  1. CLIフラグ
  2. 環境変数
  3. 構成ファイル

このドキュメントの残りの部分では、構成ファイルに変更を加えることに言及します。
ただし、構成ファイルの変更は、環境変数またはCLIフラグによってオーバーライドできます。

たとえば、クライアント構成ファイルに次のものがある場合：

.. code:: yaml

    tls:
      # Enable TLS (default: false)
      enabled: false

      # TLS for the client's listenting port (default: false)
      certfiles:
      client:
        certfile: cert.pem
        keyfile:

次の環境変数を使用して、構成ファイルの ``cert.pem`` 設定をオーバーライドできます。

.. code:: bash

  export FABRIC_CA_CLIENT_TLS_CLIENT_CERTFILE=cert2.pem

環境変数と構成ファイルの両方をオーバーライドする場合は、コマンドラインフラグを使用できます。

.. code:: bash

  fabric-ca-client enroll --tls.client.certfile cert3.pem

同じアプローチがfabric-ca-serverに適用されますが、環境変数のプレフィックスとして
``FABIRC_CA_CLIENT``を使用する代わりに、``FABRIC_CA_SERVER``が使用されます。

.. _server:

A word on file paths
^^^^^^^^^^^^^^^^^^^^^

ファイルパスに関する一言

ファイル名を指定する Fabric CA サーバーおよびクライアント構成ファイルのすべてのプロパティは、
相対パスと絶対パスの両方をサポートします。
相対パスは、構成ファイルが置かれている config ディレクトリーに対する相対パスです。
たとえば、config ディレクトリが ``~/config`` で、tlsセクションが以下のようになっている場合、
Fabric CAサーバーまたはクライアントは以下の通り検索します。

``~/config`` ディレクトリの ``root.pem`` ファイル
``~/config/certs`` ディレクトリの ``cert.pem`` ファイル
``/abs/path`` ディレクトリの ``key.pem`` ファイル

.. code:: yaml

    tls:
      enabled: true
      certfiles:
        - root.pem
      client:
        certfile: certs/cert.pem
        keyfile: /abs/path/key.pem

`Back to Top`_



Fabric CA Server
----------------

このセクションでは、Fabric CA serverについて説明します。

Fabric CA serverを開始する前に初期化できます。 
これにより、サーバーを起動する前に確認およびカスタマイズできるデフォルトの構成ファイルを生成できます。

Fabric CA server のホームディレクトリは、次のように決定されます。
  - –homeコマンドラインオプションが設定されている場合は、その値を使用します
  - それ以外の場合、 ``FABRIC_CA_SERVER_HOME`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合、 ``FABRIC_CA_HOME`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合、 ``CA_CFG_PATH`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合は、現在の作業ディレクトリを使用します

このサーバーセクションの残りの部分では、 ``FABRIC_CA_HOME`` 環境変数を
以下のように設定していることを前提としています。
``$HOME/fabric-ca/server``

以下の手順は、サーバー構成ファイルがサーバーのホームディレクトリに存在することを前提としています。

.. _initialize:

Initializing the server
~~~~~~~~~~~~~~~~~~~~~~~

サーバーの初期化

以下のようにFabric CA serverを初期化します。

.. code:: bash

    fabric-ca-server init -b admin:adminpw

LDAPが無効になっている場合、初期化には ``-b`` (bootstrap identity：ブートストラップID) オプションが必要です。 
Fabric CA serverを起動するには、少なくとも1つの ブートストラップID が必要です。 このIDはサーバー管理者です。

サーバー構成ファイルには、構成可能な証明書署名要求（CSR:Certificate Signing Request）セクションが含まれています。 
以下はCSRのサンプルです。

.. _csr-fields:

.. code:: yaml

   cn: fabric-ca-server
   names:
      - C: US
        ST: "North Carolina"
        L:
        O: Hyperledger
        OU: Fabric
   hosts:
     - host1.example.com
     - localhost
   ca:
      expiry: 131400h
      pathlength: 1

上記のすべてのフィールドは、 ``fabric-ca-server init`` によって生成されるX.509署名鍵と証明書に関係しています。 
これは、サーバーの構成ファイルの ``ca.certfile`` および ``ca.keyfile`` ファイルに対応します。 
フィールドは次のとおりです。

  -  **cn** は一般名（Common Name）です
  -  **O** は組織名（organization name）です
  -  **OU** は組織単位(organizational unit）です
  -  **L** は場所または都市（location or city）です
  -  **ST** は状態（state）です
  -  **C** は国（country）です

CSRのカスタム値が必要な場合は、構成ファイルをカスタマイズし、
 ``ca.certfile`` および ``ca.keyfile`` 構成アイテムで指定されたファイルを削除してから、
 ``fabric-ca-server init -b admin：adminpw`` コマンドを再度実行します。

``fabric-ca-server init`` コマンドは、 ``-u <parent-fabric-ca-server-URL>`` オプションが指定されていない限り、
自己署名CA証明書を生成します。
``-u`` が指定されている場合、サーバーのCA証明書は親Fabric CA server によって署名されます。
親Fabric CA serverへの認証を行うには、URLは ``<scheme>：// <enrollmentID>：<secret> @ <host>：<port>`` の形式である必要があります。
ここで言及されている、<enrollmentID> および <secret> は 'hf.IntermediateCA' 属性値が「true」であるアイデンティティに対応するものになります。
``fabric-ca-server init`` コマンドは、サーバーのホームディレクトリに
**fabric-ca-server-config.yaml** という名前のデフォルト構成ファイルも生成します。

提供するCA署名証明書とキーファイルをFabric CA server で使用する場合は、
``ca.certfile`` と ``ca.keyfile`` でそれぞれ参照される場所にファイルを配置する必要があります。
両方のファイルはPEMエンコードされている必要があり、暗号化されていてはなりません。
より具体的には、CA証明書ファイルの内容は ``----- BEGIN CERTIFICATE -----`` で始まり、
キーファイルの内容は ``----- BEGIN PRIVATE KEY -----`` で始まる必要があります。
``-----BEGIN ENCRYPTED PRIVATE KEY-----`` ではありません。

アルゴリズムとキーサイズ

CSRは、楕円曲線（ECDSA）をサポートするX.509証明書とキーを生成するようにカスタマイズできます。 
次の設定は、曲線 ``prime256v1`` および署名アルゴリズム ``ecdsa-with-SHA256`` を使用した楕円曲線デジタル署名アルゴリズム（ECDSA）の実装例です。

.. code:: yaml

    key:
       algo: ecdsa
       size: 256

アルゴリズムとキーサイズの選択は、セキュリティのニーズに基づいています。

楕円曲線（ECDSA）は、次のキーサイズオプションを提供します。

+--------+--------------+-----------------------+
| size   | ASN1 OID     | Signature Algorithm   |
+========+==============+=======================+
| 256    | prime256v1   | ecdsa-with-SHA256     |
+--------+--------------+-----------------------+
| 384    | secp384r1    | ecdsa-with-SHA384     |
+--------+--------------+-----------------------+
| 521    | secp521r1    | ecdsa-with-SHA512     |
+--------+--------------+-----------------------+

Starting the server
~~~~~~~~~~~~~~~~~~~

サーバーを起動する

次のようにFabric CAサーバーを起動します。

.. code:: bash

    fabric-ca-server start -b <admin>:<adminpw>

サーバーがこれまで初期化されていない場合は、初めて起動するときにサーバー自体によって初期化されます。
この初期化中に、サーバーは ca-cert.pem および ca-key.pem ファイルがまだ存在しない場合は生成し、存在しない場合はデフォルトの構成ファイルも作成します。
Fabric `CAサーバーの初期化 <#initialize>`__セクションを参照してください。

Fabric CA server が LDAP を使用するように構成されていない限り、
少なくとも1つの事前登録されたブートストラップIDで構成して、他のアイデンティティを登録、および登録できるようにする必要があります。
``-b`` オプションは、ブートストラップIDの名前とパスワードを指定します。

Fabric CAサーバーが ``http`` ではなく ``https`` でリッスンするようにするには、 ``tls.enabled`` を ``true`` に設定します。

セキュリティ警告：Fabric CA server は、TLSを有効にして（ ``tls.enabled`` を ``true`` に設定して）常に起動する必要があります。
そうしないと、サーバーがネットワークトラフィックへのアクセス権を持つ攻撃者に対して脆弱になります。

同じシークレット（またはパスワード）を登録に使用できる回数を制限するには、構成ファイルの ``registry.maxenrollments`` を適切な値に設定します。
値を1に設定すると、Fabric CA server は、特定の登録IDに対してパスワードを1回だけ使用することを許可します。
値を-1に設定すると、Fabric CA server は、登録のためにシークレットを再利用できる回数に制限を設けません。
デフォルト値は-1です。
値を0に設定すると、Fabric CAサーバーはすべてのアイデンティティの登録を無効にし、アイデンティティの登録は許可されません。

これで、Fabric CAサーバーはポート7054でリッスンするはずです。

Fabric CAサーバーをクラスターで実行したり、LDAPを使用したりしないように設定する場合は、
`Fabric CA Client <#fabric-ca-client>`__ セクションにスキップできます。

Configuring the database
~~~~~~~~~~~~~~~~~~~~~~~~

データベースの設定

このセクションでは、PostgreSQLまたはMySQLデータベースに接続するようにFabric CAサーバーを構成する方法について説明します。
デフォルトのデータベースはSQLiteで、デフォルトのデータベースファイルはFabric CAサーバーのホームディレクトリにある ``fabric-ca-server.db`` です。

クラスタでFabric CAサーバーを実行する必要がない場合は、このセクションをスキップできます。 
それ以外の場合は、以下で説明するようにPostgreSQLまたはMySQLを構成する必要があります。
ファブリックCAは、クラスターセットアップで次のデータベースバージョンをサポートします。

- PostgreSQL: 9.5.5 or later
- MySQL: 5.7 or later

PostgreSQL
^^^^^^^^^^

PostgreSQLデータベースに接続するために、サーバーの構成ファイルに次のサンプルを追加できます。
さまざまな値を適切にカスタマイズしてください。
データベース名に使用できる文字には制限があります。
詳細については、次のPostgresのドキュメントを参照してください。
https://www.postgresql.org/docs/current/static/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS

.. code:: yaml

    db:
      type: postgres
      datasource: host=localhost port=5432 user=Username password=Password dbname=fabric_ca sslmode=verify-full

*sslmode* を指定すると、SSL認証のタイプが構成されます。 
sslmodeの有効な値は次のとおりです。

|

+----------------+----------------+
| Mode           | Description    |
+================+================+
| disable        | No SSL         |
+----------------+----------------+
| require        | Always SSL     |
|                | (skip          |
|                | verification)  |
+----------------+----------------+
| verify-ca      | Always SSL     |
|                | (verify that   |
|                | the            |
|                | certificate    |
|                | presented by   |
|                | the server was |
|                | signed by a    |
|                | trusted CA)    |
+----------------+----------------+
| verify-full    | Same as        |
|                | verify-ca AND  |
|                | verify that    |
|                | the            |
|                | certificate    |
|                | presented by   |
|                | the server was |
|                | signed by a    |
|                | trusted CA and |
|                | the server     |
|                | hostname       |
|                | matches the    |
|                | one in the     |
|                | certificate    |
+----------------+----------------+

|

TLSを使用する場合は、Fabric CAサーバー構成ファイルの ``db.tls`` セクションを指定する必要があります。
PostgreSQLサーバーでSSLクライアント認証が有効になっている場合は、 ``db.tls.client`` セクションでクライアント証明書と
キーファイルも指定する必要があります。
以下は、 ``db.tls`` セクションの例です。

.. code:: yaml

    db:
      ...
      tls:
          enabled: true
          certfiles:
            - db-server-cert.pem
          client:
                certfile: db-client-cert.pem
                keyfile: db-client-key.pem

| **certfiles** - PEMエンコードされた信頼されたルート証明書ファイルのリスト。
| **certfile** and **keyfile** - PEMエンコードされた証明書とキーファイルで、PostgreSQLサーバーと安全に通信するためにFabric CAサーバーが使用するもの。

PostgreSQL SSL Configuration
"""""""""""""""""""""""""""""

**PostgreSQLサーバーでSSLを構成するための基本的な手順：**

1. postgresql.confで、SSLのコメントを外し、「on」に設定します（SSL=on）

2. PostgreSQLデータディレクトリに、証明書とキーファイルを配置します。

自己署名証明書を生成する手順は以下を参照してください。
https://www.postgresql.org/docs/9.5/static/ssl-tcp.html

注：自己署名証明書はテスト用であり、実稼働環境では使用しないでください

**PostgreSQLサーバー - クライアント証明書を必要とする**

1. 信頼できる認証局（CA）の証明書を、PostgreSQLデータディレクトリのファイル root.crt に配置します。

2. postgresql.conf で、「ssl_ca_file」がクライアントのルート証明書（CA証明書）を指すように設定します

3. pg_hba.conf の適切な hostssl の行で clientcert パラメーターを1に設定します。

PostgreSQLサーバーでSSLを構成する方法の詳細については、以下のPostgreSQLドキュメントを参照してください。https：//www.postgresql.org/docs/9.4/static/libpq-ssl.html

MySQL
^^^^^^^

MySQLデータベースに接続するために、次のサンプルをFabric CA server構成ファイルに追加できます。
さまざまな値を適切にカスタマイズしてください。
データベース名に使用できる文字には制限があります。
詳細については、以下のMySQLドキュメントを参照してください。
https://dev.mysql.com/doc/refman/5.7/en/identifiers.html

MySQL 5.7.Xでは、特定のモードについて、サーバーが「0000-00-00」を有効な日付として許可するかどうかに影響します。
MySQLサーバーが使用するモードを緩和する必要がある場合があります。
ですので、サーバーがゼロの日付値を受け入れることができるようにしましょう。

my.cnfで、構成オプション *sql_mode* を見つけ、NO_ZERO_DATEが存在する場合は削除します。 
この変更を行った後、MySQLサーバーを再起動します。

使用可能なさまざまなモードに関する次のMySQLドキュメントを参照し、使用されている特定のバージョンのMySQLに適切な設定を選択してください。

https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html

.. code:: yaml

    db:
      type: mysql
      datasource: root:rootpw@tcp(localhost:3306)/fabric_ca?parseTime=true&tls=custom

TLS経由でMySQLサーバーに接続する場合、上記の **PostgreSQL** セクションで説明されているように、
``db.tls.client`` セクションも必要です。

MySQL SSL Configuration
""""""""""""""""""""""""

**MySQLサーバーでSSLを構成するための基本的な手順：**

1. サーバーのmy.cnfファイルを開くか作成します。
   [mysqld]セクションで以下の行を追加またはコメント解除します。
   これらは、サーバーのキーと証明書、およびルートCA証明書を指している必要があります。

   サーバーおよびクライアント側の証明書の作成手順は以下の通りです。
   http://dev.mysql.com/doc/refman/5.7/en/creating-ssl-files-using-openssl.html

   [mysqld] ssl-ca = ca-cert.pem ssl-cert = server-cert.pem ssl-key = server-key.pem

   次のクエリを実行して、SSLが有効になっていることを確認できます。

   mysql> SHOW GLOBAL VARIABLES LIKE 'have\_%ssl';

   実行結果は以下のようになるでしょう：

   +----------------+----------------+
   | Variable_name  | Value          |
   +================+================+
   | have_openssl   | YES            |
   +----------------+----------------+
   | have_ssl       | YES            |
   +----------------+----------------+

2. サーバー側のSSL設定が完了したら、次のステップとして、SSL経由でMySQLサーバーにアクセスする権限を持つユーザーを作成します。
   そのためには、MySQLサーバーにログインし、次のように入力します。

   mysql> GRANT ALL PRIVILEGES ON . TO ‘ssluser’@’%’ IDENTIFIED BY ‘password’ REQUIRE SSL; 
   mysql> FLUSH PRIVILEGES;

   ユーザーがサーバーにアクセスする特定のIPアドレスを指定する場合は、「%」を特定のIPアドレスに変更します。

**MySQLサーバー - クライアント証明書を必要とする**

セキュア接続のオプションは、サーバー側で使用されるオプションと似ています。

-  ssl-ca は、認証局（CA）証明書を識別します。このオプションを使用する場合、サーバーが使用する証明書と同じ証明書を指定する必要があります。
-  ssl-cert は、MySQLサーバーの証明書を識別します。
-  ssl-key は、MySQLサーバーの秘密鍵を識別します。

特別な暗号化要件のないアカウント、または REQUIRE SSL オプションを含む GRANT ステートメントを使用し、作成されたアカウントで接続するとします。
推奨されるセキュア接続オプションのセットとして、少なくとも -ssl-cert および -ssl-key オプションを使用してMySQLサーバーを起動します。
次に、サーバー構成ファイルで ``db.tls.certfiles`` プロパティーを設定し、Fabric CA serverを開始します。

クライアント証明書も指定するように要求するには、REQUIRE X509 オプションを使用してアカウントを作成します。
次に、クライアントは適切なクライアントキーと証明書ファイルも指定する必要があります。 
そうしないと、MySQLサーバーは接続を拒否します。
Fabric CA server のクライアントキーと証明書ファイルを指定するには、 
``db.tls.client.certfile`` および ``db.tls.client.keyfile`` 構成プロパティを設定します。

Configuring LDAP
~~~~~~~~~~~~~~~~

Fabric CAサーバーは、LDAPサーバーから読み取るように構成できます。

特に、Fabric CAサーバーはLDAPサーバーに接続して次のことを実行できます。

-  登録前にアイデンティティを認証する
-  認証に使用されるアイデンティティの属性値を取得します。

Fabric CAサーバーの構成ファイルのLDAPセクションを変更して、LDAPサーバーに接続するようにサーバーを構成します。

.. code:: yaml

    ldap:
       # Enables or disables the LDAP client (default: false)
       enabled: false
       # The URL of the LDAP server
       url: <scheme>://<adminDN>:<adminPassword>@<host>:<port>/<base>
       userfilter: <filter>
       attribute:
          # 'names' is an array of strings that identify the specific attributes
          # which are requested from the LDAP server.
          names: <LDAPAttrs>
          # The 'converters' section is used to convert LDAP attribute values
          # to fabric CA attribute values.
          #
          # For example, the following converts an LDAP 'uid' attribute
          # whose value begins with 'revoker' to a fabric CA attribute
          # named "hf.Revoker" with a value of "true" (because the expression
          # evaluates to true).
          #    converters:
          #       - name: hf.Revoker
          #         value: attr("uid") =~ "revoker*"
          #
          # As another example, assume a user has an LDAP attribute named
          # 'member' which has multiple values of "dn1", "dn2", and "dn3".
          # Further assume the following configuration.
          #    converters:
          #       - name: myAttr
          #         value: map(attr("member"),"groups")
          #    maps:
          #       groups:
          #          - name: dn1
          #            value: client
          #          - name: dn2
          #            value: peer
          # The value of the user's 'myAttr' attribute is then computed to be
          # "client,peer,dn3".  This is because the value of 'attr("member")' is
          # "dn1,dn2,dn3", and the call to 'map' with a 2nd argument of
          # "group" replaces "dn1" with "client" and "dn2" with "peer".
          converters:
            - name: <fcaAttrName>
              value: <fcaExpr>
          maps:
            <mapName>:
                - name: <from>
                  value: <to>

Where:

  * ``scheme`` *ldap* もしくは *ldaps*
  * ``adminDN`` adminユーザーの識別名です。
  * ``pass`` adminユーザーのパスワードです。
  * ``host`` LDAPサーバーのホスト名またはIPアドレスです。
  * ``port`` オプションのポート番号です。デフォルトでは、*ldap* の場合は389、*ldaps* の場合は636です。
  * ``base`` 検索に使用するLDAPツリーのオプションのルートです。
  * ``filter`` ログインユーザー名を識別名に変換するために検索するときに使用するフィルターです。
    たとえば、値 ``(uid=%s)`` は、値がログインユーザー名である ``uid`` 属性の値を持つLDAPエントリを検索します。
    同様に、 ``(email=%s)`` を使用して電子メールアドレスでログインできます。 
  * ``LDAPAttrs`` ユーザーに代わってLDAPサーバーから要求するLDAP属性名の配列です。
  * attribute.convertersセクションは、LDAP属性をファブリックCA属性に変換するために使用されます。
    * ``fcaAttrName`` ファブリックCA属性の名前です。
    * ``fcaExpr`` 評価値がファブリックCA属性に割り当てられる式です。
    たとえば、<LDAPAttrs> が ["uid"]、<fcaAttrName> が 'hf.Revoker' 、<fcaExpr> が 'attr("uid") =~ "revoker*"' であるとします。
    これは、ユーザーに代わってLDAPサーバーから "uid" という名前の属性が要求されることを意味します。
    ユーザーの「uid」LDAP属性の値が「revoker」で始まる場合、ユーザーには「hf.Revoker」属性の「true」の値が与えられます。
    それ以外の場合、ユーザーには「hf.Revoker」属性に「false」の値が与えられます。
  * the attribute.maps セクションは、LDAP応答値をマップするために使用されます。
    典型的な使用例は、LDAPグループに関連付けられた識別名をID種別にマップすることです。


LDAP expression language は、以下で説明されているgovaluateパッケージを使用します。
https：//github.com/Knetic/govaluate/blob/master/MANUAL.md
これは、 "=~" などの演算子と、 "revoker*" などのリテラルを定義します。
これは正規表現です。
govaluate langage を拡張するLDAP固有の変数と関数は次のとおりです。

  * ``DN`` ユーザーの所属に等しい変数です。
  * ``affiliation`` ユーザーの識別名に等しい変数です。
  * ``attr`` 1つまたは2つの引数を取る関数です。 
    最初の引数はLDAP属性名です。 
    2番目の引数は、複数の値を1つの文字列に結合するために使用される区切り文字列です。 
    デフォルトの区切り文字列は "," です。 
    ``attr`` 関数は、常に 'string' 型の値を返します。
  * ``map`` mapは2つの引数を取る関数です。 
    最初の引数は任意の文字列です。 
    2番目の引数はマップの名前であり、1番目の引数からの文字列の文字列置換を実行するために使用されます。
  * ``if`` 最初の引数がboolean値に解決される必要がある3つの引数を取る関数です。 
    trueと評価されると、2番目の引数が返されます。 
    それ以外の場合、3番目の引数が返されます。

たとえば、ユーザーが "O=org1,C=US" で終わる識別名を持っている場合、
またはユーザーが "org1.dept2." で始まる所属を持ち、 "admin" 属性が "true" の場合、次の式はtrueと評価されます。

  **DN =~ "*O=org1,C=US" || (affiliation =~ "org1.dept2.*" && attr('admin') = 'true')**

注： ``attr`` 関数は常に「string」型の値を返すため、数値演算子を使用して式を作成することはできません。 
たとえば、次は有効な式ではありません。

.. code:: yaml

     value: attr("gidNumber) >= 10000 && attr("gidNumber) < 10006

または、次のように引用符で囲まれた正規表現を使用して、同等の結果を返すこともできます。

.. code:: yaml

     value: attr("gidNumber") =~ "1000[0-5]$" || attr("mail") == "root@example.com"

以下は、Dockerイメージが以下のURLにあるOpenLDAPサーバーのデフォルト設定のサンプル構成セクションです。
``https://github.com/osixia/docker-openldap``

.. code:: yaml

    ldap:
       enabled: true
       url: ldap://cn=admin,dc=example,dc=org:admin@localhost:10389/dc=example,dc=org
       userfilter: (uid=%s)

``FABRIC_CA/scripts/run-ldap-tests`` を参照すると、
OpenLDAPドッカーイメージを起動し、構成し、 ``FABRIC_CA/cli/server/ldap/ldap_test.go`` でLDAPテストを実行し、
OpenLDAPサーバーを停止するスクリプトについて理解できます。

LDAPを構成すると、登録は次のように機能します。

-  Fabric CAクライアントまたはクライアントSDKは、basic認証ヘッダーを含む登録要求を送信します。
-  Fabric CAサーバーは登録要求を受信し、認証ヘッダーのID名とパスワードをデコードし、
   構成ファイルの “userfilter”  を使用してID名に関連付けられたDN（Distinguish Name : 識別名）を検索し、IDのパスワードでLDAPバインドを試行します。
   LDAPバインドが成功すると、登録処理が許可され、続行できます。


Setting up a cluster
~~~~~~~~~~~~~~~~~~~~

クラスターのセットアップ

任意のIPスプレイヤーを使用して、ファブリックCAサーバーのクラスターに負荷を分散できます。
このセクションでは、ファブリックCAサーバークラスターにルーティングするように Haproxy をセットアップする方法の例を示します。
ファブリックCAサーバーの設定を反映するために、ホスト名とポートを必ず変更してください。

haproxy.conf

.. code::

    global
          maxconn 4096
          daemon

    defaults
          mode http
          maxconn 2000
          timeout connect 5000
          timeout client 50000
          timeout server 50000

    listen http-in
          bind *:7054
          balance roundrobin
          server server1 hostname1:port
          server server2 hostname2:port
          server server3 hostname3:port


注：TLSを使用する場合は、 ``mode tcp`` を使用する必要があります。


Setting up multiple CAs
~~~~~~~~~~~~~~~~~~~~~~~

複数のCAのセットアップ

デフォルトでは、fabric-ca サーバーは単一のデフォルトCAで構成されています。
ただし、 `cafiles` または `cacount` 構成オプションを使用して、追加のCAを単一のサーバーに追加できます。
追加の各CAには、独自のホームディレクトリがあります。

cacount:
^^^^^^^^

`cacount` は、X個のデフォルトの追加CAをすばやく開始する方法を提供します。
ホームディレクトリは、サーバーディレクトリに相対的です。
このオプションを使用すると、ディレクトリ構造は次のようになります。

.. code:: yaml

    --<Server Home>
      |--ca
        |--ca1
        |--ca2

追加の各CAは、そのホームディレクトリに生成されたデフォルトの構成ファイルを取得します。
構成ファイル内には、一意のCA名が含まれます。

たとえば、次のコマンドは2つのデフォルトCAインスタンスを起動します。

.. code:: bash

   fabric-ca-server start -b admin:adminpw --cacount 2

cafiles:
^^^^^^^^

cafiles構成オプションを使用するときに絶対パスが指定されていない場合、CAホームディレクトリはサーバーディレクトリに対して相対的になります。

このオプションを使用するには、開始するCAごとにCA構成ファイルが既に生成および構成されている必要があります。
各構成ファイルには一意のCA名と共通名（CN）が必要です。
そうでない場合、これらの名前は一意である必要があるため、サーバーの起動に失敗します。
CA構成ファイルは、デフォルトのCA構成をオーバーライドし、CA構成ファイルで欠落しているオプションは、デフォルトのCAの値に置き換えられます。

優先順位は次のとおりです。

   1. CA設定ファイル
   2. デフォルトのCA CLIフラグ
   3. デフォルトのCA環境変数
   4. デフォルトのCA構成ファイル

CA構成ファイルには、少なくとも次のものが含まれている必要があります。

.. code:: yaml

    ca:
    # Name of this CA
    name: <CANAME>

    csr:
      cn: <COMMONNAME>

次のようにディレクトリ構造を構成できます。

.. code:: yaml

    --<Server Home>
      |--ca
        |--ca1
          |-- fabric-ca-config.yaml
        |--ca2
          |-- fabric-ca-config.yaml

たとえば、次のコマンドは2つのカスタマイズされたCAインスタンスを起動します。

.. code:: bash

    fabric-ca-server start -b admin:adminpw --cafiles ca/ca1/fabric-ca-config.yaml
    --cafiles ca/ca2/fabric-ca-config.yaml


Enrolling an intermediate CA
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

中間CAの登録

中間CAのCA署名証明書を作成するには、fabric-ca-clientがCAに登録するのと同じ方法で、中間CAが親CAに登録する必要があります。
これは、以下に示すように、-u オプションを使用して、親CAのURLと登録IDおよびシークレットを指定することにより行われます。
この登録IDに関連付けられたアイデンティティには、属性として "hf.IntermediateCA" が "true" となっているものが必要となります。
中間CAがCN値を明示的に指定しようとすると、エラーが発生します。

.. code:: bash

    fabric-ca-server start -b admin:adminpw -u http://<enrollmentID>:<secret>@<parentserver>:<parentport>

他の中間CAフラグについては、`Fabric CA server's configuration file format`_ をご覧ください。


Upgrading the server
~~~~~~~~~~~~~~~~~~~~

サーバーのアップグレード

Fabric CAクライアントをアップグレードする前に、Fabric CAサーバーをアップグレードする必要があります。 
アップグレードの前に、現在のデータベースをバックアップすることをお勧めします。

- sqlite3を使用している場合、現在のデータベースファイル（デフォルトではfabric-ca-server.dbという名前）をバックアップします。
- 他のデータベースタイプの場合は、適切なバックアップ/レプリケーションメカニズムを使用します。

Fabric CAサーバーの単一インスタンスをアップグレードするには：

1. fabric-ca-server プロセスを停止します。
2. 現在のデータベースがバックアップされていることを確認します。
3. 以前の fabric-ca-server バイナリをアップグレードされたバージョンに置き換えます。
4. fabric-ca-server プロセスを起動します。
5. 次のコマンドを使用して、fabric-ca-serverプロセスが使用可能であることを確認します。<host> は、サーバーが起動されたホスト名です。

      fabric-ca-client getcainfo -u http://<host>:7054

Upgrading a cluster:
^^^^^^^^^^^^^^^^^^^^

クラスターのアップグレード：

MySQL または Postgres データベースを使用して fabric-ca-server インスタンスのクラスターをアップグレードするには、次の手順を実行します。
haproxyを使用して、それぞれ host1 と host2 の2つの fabric-ca-server クラスターメンバーに負荷分散し、両方ともポート7054でリッスンしていると仮定します。
この手順の後、両方ともポート7054でリッスンしている host3 と host4 のアップグレードされた fabric-ca-server クラスターメンバーの負荷分散を行います。

haproxy統計を使用して変更を監視するには、統計収集を有効にします。 
haproxy設定ファイルのグローバルセクションに次の行を追加します。

::

    stats socket /var/run/haproxy.sock mode 666 level operator
    stats timeout 2m

haproxyを再起動して、変更を有効にします。

    # haproxy -f <configfile> -st $(pgrep haproxy)

haproxy "show stat"コマンドからの要約情報を表示するには、次の関数から返された大量のCSVデータが解析に役立つことがあります。

.. code:: bash

    haProxyShowStats() {
       echo "show stat" | nc -U /var/run/haproxy.sock |sed '1s/^# *//'|
          awk -F',' -v fmt="%4s %12s %10s %6s %6s %4s %4s\n" '
             { if (NR==1) for (i=1;i<=NF;i++) f[tolower($i)]=i }
             { printf fmt, $f["sid"],$f["pxname"],$f["svname"],$f["status"],
                           $f["weight"],$f["act"],$f["bck"] }'
    }


1) 最初に、haproxy設定ファイルは次のようになります。

      server server1 host1:7054 check
      server server2 host2:7054 check
   
   この構成を次のように変更します。

      server server1 host1:7054 check backup
      server server2 host2:7054 check backup
      server server3 host3:7054 check
      server server4 host4:7054 check

2) 次のように、新しい構成でHAプロキシを再起動します。


      haproxy -f <configfile> -st $(pgrep haproxy)

   ``"haProxyShowStats"`` は、2つのアクティブな古いバージョンのバックアップサーバーと
   2つの（まだ開始されていない）アップグレードされたサーバーで、変更された構成を反映します。

      sid   pxname      svname  status  weig  act  bck
        1   fabric-cas  server3   DOWN     1    1    0
        2   fabric-cas  server4   DOWN     1    1    0
        3   fabric-cas  server1     UP     1    0    1
        4   fabric-cas  server2     UP     1    0    1

3) host3およびhost4にfabric-ca-serverのアップグレードされたバイナリをインストールします。 
   host3およびhost4の新しいアップグレードされたサーバーは、host1およびhost2の古い対応サーバーと同じデータベースを使用するように構成する必要があります。 
   アップグレードされたサーバーを起動すると、データベースは自動的に移行されます。 
   haproxyは、すべての新しいトラフィックがバックアップサーバーとして構成されていないため、アップグレードされたサーバーにすべてのトラフィックを転送します。 
   先に進む前に、 ``"fabric-ca-client getcainfo"`` コマンドを使用して、クラスターがなお適切に機能していることを確認してください。 
   また、``"haProxyShowStats"`` は、次のように、すべてのサーバーがアクティブであることを反映する必要があります。

      sid   pxname      svname  status  weig  act  bck
        1   fabric-cas  server3    UP     1    1    0
        2   fabric-cas  server4    UP     1    1    0
        3   fabric-cas  server1    UP     1    0    1
        4   fabric-cas  server2    UP     1    0    1

4) host1およびhost2の古いサーバーを停止します。
   先に進む前に、 ``"fabric-ca-client getcainfo"`` コマンドを使用して、新しいクラスターが適切に機能していることを確認してください。
   次に、古いサーバーバックアップ構成をhaproxy構成ファイルから削除します。これにより、次のようになります。

      server server3 host3:7054 check
      server server4 host4:7054 check

5) 次のように、新しい構成でHAプロキシを再起動します。

      haproxy -f <configfile> -st $(pgrep haproxy)

   ``"haProxyShowStats"`` は変更された構成を反映し、新しいバージョンにアップグレードされた2つのアクティブなサーバーを確認できます。

      sid   pxname      svname  status  weig  act  bck
        1   fabric-cas  server3   UP       1    1    0
        2   fabric-cas  server4   UP       1    1    0


`Back to Top`_


Operations Service
~~~~~~~~~~~~~~~~~~~~

運用サービス

CA Serverは、RESTfulな「操作」APIを提供するHTTPサーバーをホストします。 
このAPIは、ネットワークの管理者や「ユーザー」ではなく、運用担当者が使用することを目的としています。

APIは次の機能を公開します。

    運用メトリックスにおけるPrometheusのターゲット（構成されている場合）
    （訳者注：Prometheusは監視ツール）

Configuring the Operations Service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

運用サービスの構成

運用サービスには、2つの基本的な構成が必要です。

    リッスンする **アドレス** と **ポート** 。
    認証と暗号化に使用する **TLS証明書** と **鍵** 。
    **注意：これらの証明書は、個別の専用CAによって生成される必要がある。**
    どのチャネルのどの組織にも、証明書を生成したCAを使用しないでください。

CAサーバーは、サーバーの構成ファイルの ``operations`` セクションで構成できます。

.. code:: yaml

  operations:
    # host and port for the operations server
    listenAddress: 127.0.0.1:9443

    # TLS configuration for the operations endpoint
    tls:
      # TLS enabled
      enabled: true

      # path to PEM encoded server certificate for the operations server
      cert:
        file: tls/server.crt

      # path to PEM encoded server key for the operations server
      key:
        file: tls/server.key

      # require client certificate authentication to access all resources
      clientAuthRequired: false

      # paths to PEM encoded ca certificates to trust for client authentication
      clientRootCAs:
        files: []

``listenAddress`` キーは、オペレーションサーバーがリッスンするホストとポートを定義します。
サーバーがすべてのアドレスをリッスンする必要がある場合、ホスト部分は省略できます。

``tls`` セクションは、運用サービスでTLSが有効になっているかどうか、サービスの証明書と秘密鍵の場所、
およびクライアント認証で信頼される認証局ルート証明書の場所を示すために使用されます。
``clientAuthRequired`` が ``true`` の場合、クライアントは認証用の証明書を提供する必要があります。

Operations Security
^^^^^^^^^^^^^^^^^^^^^^

運用セキュリティ

運用サービスは運用に焦点を合わせており、意図的にFabricネットワークとは無関係であるため、アクセス制御にMembership Services Providerを使用しません。
代わりに、オペレーションサービスは、クライアント証明書による認証を使用した、Mutal TLSに完全に依存しています。

（訳者注：接続先のサーバーが信頼できることを保証するためにHTTPSを使い、接続してくるクライアントが信頼できることを保証するためTLSクライアント認証がある。これを同時に行うことをMutual TLS(SSL)と呼びます。）

実稼働環境で ``clientAuthRequired`` の値を ``true`` に設定して、Mutal TLSを有効にすることを強くお勧めします。
この構成では、クライアントは認証に有効な証明書を提供する必要があります。
クライアントが証明書を提供しない場合、またはサービスがクライアントの証明書を検証できない場合、要求は拒否されます。
 ``clientAuthRequired`` が ``false`` に設定されている場合、クライアントは証明書を提供する必要がないことに注意してください。 
 ただし、証明書が提供され、サービスが検証できない場合、要求は拒否されます。

TLSが無効になっている場合、認承はバイパスされ、オペレーションエンドポイントに接続できるすべてのクライアントがAPIを使用できるようになります。

Metrics
^^^^^^^^^

メトリクス

Fabric CAは、システムの動作に関する内部情報を提供できるメトリクスを公開します。 
オペレーターと管理者は、この情報を使用して、システムが長期にわたってどのように機能しているかをよりよく理解できます。


Configuring Metrics
^^^^^^^^^^^^^^^^^^^^

メトリクスの設定

ファブリックCAは、メトリックを公開する2つの方法を提供します。
Prometheusに基づく **プルモデル** とStatsDに基づく **プッシュモデル** です。

Prometheus
^^^^^^^^^^^

典型的なプロメテウスの展開では、計測されたターゲットによって公開されたHTTPエンドポイントから
メトリクスを要求・スクレイピングします。
Prometheusはメトリクスのリクエストを担当するため、プルシステムと見なされます。

設定する際、Fabric CA Serverは、運用サービスに ``/metrics`` リソースを提示します。 
Prometheusを有効にするには、サーバーの構成ファイルでプロバイダーの値を ``prometheus`` に設定します。

.. code:: yaml

  metrics:
    provider: prometheus

StatsD
^^^^^^^

StatsDは、単純な統計集約デーモンです。 メトリクスは ``statsd`` デーモンに送信され、
そこで収集され、集計され、視覚化とアラートのためにバックエンドにプッシュされます。 
このモデルでは、メトリクスデータを StatsD に送信するために計測されたプロセスが必要であるため、これはプッシュシステムと見なされます。

サーバーの構成ファイルの ``metrics`` セクションでメトリクスプロバイダーを ``statsd`` に設定することで、
CA Serverがメトリックを StatsD に送信するように構成できます。
``statsd`` サブセクションは、StatsD デーモンのアドレス、使用するネットワークタイプ（ ``tcp`` または ``udp`` ）、
およびメトリックの送信頻度で構成する必要もあります。
オプションの ``prefix`` を指定して、メトリクスのソースを区別できます（たとえば、別々のサーバーからのメトリクスを区別する）。
これは、生成されたすべてのメトリクスの先頭に追加されます。

.. code:: yaml

  metrics:
    provider: statsd
    statsd:
      network: udp
      address: 127.0.0.1:8125
      writeInterval: 10s
      prefix: server-0

For a look at the different metrics that are generated, check out
:doc:`metrics_reference`.

`Back to Top`_

.. _client:

Fabric CA Client
----------------

Fabric CA クライアント

このセクションでは、fabric-ca-clientコマンドの使用方法について説明します。

Fabric CAクライアントのホームディレクトリは、次のように決定されます。
  - –-home コマンドラインオプションが設定されている場合は、その値を使用します
  - それ以外の場合、 ``FABRIC_CA_CLIENT_HOME`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合、 ``FABRIC_CA_HOME`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合、 ``CA_CFG_PATH`` 環境変数が設定されている場合は、その値を使用します
  - それ以外の場合は、 ``$HOME/.fabric-ca-client`` を使用します

以下の手順では、クライアント構成ファイルがクライアントのホームディレクトリに存在することを前提としています。

Enrolling the bootstrap identity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ブートストラップIDの登録

最初に、必要に応じて、クライアント構成ファイルの証明書署名要求（CSR : Certificate Signing Request）セクションをカスタマイズします。
``csr.cn`` フィールドは、ブートストラップIDとして設定する必要があることに注意してください。 
デフォルトのCSR値は次のとおりです。

.. code:: yaml

    csr:
      cn: <<enrollment ID>>
      key:
        algo: ecdsa
        size: 256
      names:
        - C: US
          ST: North Carolina
          L:
          O: Hyperledger Fabric
          OU: Fabric CA
      hosts:
       - <<hostname of the fabric-ca-client>>
      ca:
        pathlen:
        pathlenzero:
        expiry:

フィールドの説明については、 `CSRフィールド<#csr-fields>`__ を参照してください。

次に、 ``fabric-ca-client enroll`` コマンドを実行してアイデンティティを登録します。 
たとえば、次のコマンドは、7054ポートでローカルに実行されているFabric CAサーバーを呼び出すことにより、
IDが **admin** でパスワードが **adminpw** のアイデンティティを登録します。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client enroll -u http://admin:adminpw@localhost:7054

enrollコマンドは、登録証明書（ECert）、対応する秘密鍵、およびCA証明書チェーンPEMファイルを
Fabric CAクライアントの ``msp`` ディレクトリのサブディレクトリに保存します。 
PEMファイルの保存場所を示すメッセージが表示されます。

Registering a new identity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

新しいアイデンティティを登録する

登録要求を実行するアイデンティティは現在登録されている必要があり、登録するアイデンティティのタイプに対する適切な権限も持っている必要があります。

特に、登録中にFabric CAサーバーによって3つの認証チェックが行われます。

1. レジストラ（つまり、呼び出し側）は、値の1つが登録されているアイデンティティのタイプと等しい値のコンマ区切りリストを持つ
   "hf.Registrar.Roles" 属性を持っている必要があります。 
   たとえば、レジストラに値 "peer" の "hf.Registrar.Roles" 属性がある場合、レジストラはタイプ "peer" のアイデンティティを登録できますが、
   クライアント、管理者、またはオーダラーは登録できません。

2. レジストラの affiliation は、登録されているアイデンティティの affiliation と同じか、その接頭辞でなければなりません。
   たとえば、 "a.b" の affiliation を持つレジストラは、 "a.b.c" の affiliation を持つアイデンティティを登録できますが、
   "a.c" の affiliation を持つアイデンティティは登録できません。
   アイデンティティに root affiliation が必要な場合、affiliation リクエストはドット（"."）である必要があり、
   レジストラも root affiliation を持っている必要があります。
   登録要求に affiliation が指定されていない場合、登録されるアイデンティティにはレジストラの affiliation が付与されます。
   
3. レジストラは、次のすべての条件が満たされている場合、アイデンティティと属性を登録できます。
   - レジストラは、条件を満たせば、プレフィックス 'hf.' を持つFabric CA予約属性を登録できます。
     これは、レジストラが属性を所有し、それが 'hf.Registrar.Attributes' 属性の値の一部である場合に限ります。
     さらに、属性がリスト型の場合、登録される属性の値は、レジストラが持っている値と等しいか、値のサブセットである必要があります。
     属性がブール型の場合、レジストラは、属性の値が 'true' である場合にのみ、属性を登録できます。
   - カスタム属性（名前が 'hf.' で始まらない属性）を登録するには、登録する属性またはパターンの値が登録されている
     'hf.Registar.Attributes' 属性がレジストラに必要です。 
     サポートされている唯一のパターンは、末尾に '*' が付いた文字列です。
     たとえば、'a.b.*' は、 'a.b.' で始まるすべての属性名に一致するパターンです。 
     たとえば、レジストラに hf.Registrar.Attributes=orgAdmin がある場合、
     レジストラがアイデンティティに対して追加または削除できる唯一の属性は 'orgAdmin' 属性です。
   - 要求された属性名が 'hf.Registrar.Attributes' の場合、この属性の要求された値が 'hf.Registrar.Attributes' の
     レジストラの値と等しいかサブセットであるかどうかを確認するため、追加のチェックが実行されます。 
     これが真であるためには、リクエストされた各値が 'hf.Registrar.Attributes' 属性のレジストラの値と一致する必要があります。 
     たとえば、 'hf.Registrar.Attributes' のレジストラの値が 'a.b.*, x.y.z' であり、要求された属性値が 'a.b.c, x.y.z' である場合、
      'a.b.c' は 'a.b.*' と一致し、 'x.y.z' は、レジストラの 'x.y.z' 値と一致します。
     
例:
   有効なシナリオ:
      1. レジストラが属性 'hf.Registrar.Attributes = a.b.*, x.y.z' を持ち、属性 'a.b.c' を登録しようとしている場合、
         'a.b.c' は 'a.b.*' と一致するため有効です。
      2. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、属性 'x.y.z' を登録しようとしている場合、
         'x.y.z' はレジストラの 'x.y.z' と一致するため有効です。
      3. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、要求された属性値が 'a.b.c, x.y.z' の場合、
         'a.b.*' はレジストラの 'a.b.c' 、 'x.y.z' はレジストラの 'x.y.z' と一致するため有効です。
      4. レジストラに属性  'hf.Registrar.Roles = peer,client,admin,orderer' があり、
         要求された属性値が 'peer' 、 'peer,client,admin,orderer' または 'client,admin'である場合、 
         要求された値は、レジストラの値と等しいか、レジストラの値のサブセットであるため、有効です。

   無効なシナリオ:
      1. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、
         属性「hf.Registar.Attributes = abc、xy *」を登録しようとしている場合、
         要求された属性 'x.y.*' はレジストラが所有しているパターンではないため無効です。
         値 'x.y.*' は 'x.y.z' の上位集合です。
      2. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、
         属性 'hf.Registar.Attributes = a.b.c, x.y.z, attr1' を登録しようとしている場合、
         レジストラの 'hf.Registrar.Attributes' の属性には 'attr1' が含まれないため無効です。
      3. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、
         属性「a.b」を登録しようとしている場合、 'a.b' は 'a.b.*' に含まれていないため無効です。
      4. レジストラに属性 'hf.Registrar.Attributes = a.b.*, x.y.z' があり、
         属性 'x.y' を登録しようとしている場合、 'x.y' は 'x.y' に含まれていないため無効です。
      5. レジストラに属性 'hf.Registrar.Roles = peer' があり、要求された属性値が 'peer,client' である場合、
         レジストラには hf.Registrar.Roles 属性の値にclientロールがないため無効です。
      6. レジストラに属性 'hf.Revoker = false' があり、要求された属性値が 'true' の場合、
         hf.Revoker 属性はブール属性であり、属性のレジストラの値は 'true' ではないため無効です。

次の表に、アイデンティティに登録できるすべての属性を示します。 
属性の名前では大文字と小文字が区別されます。

+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| Name                        | Type       | Description                                                                                                |
+=============================+============+============================================================================================================+
| hf.Registrar.Roles          | List       | List of roles that the registrar is allowed to manage                                                      |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.Registrar.DelegateRoles  | List       | List of roles that the registrar is allowed to give to a registree for its 'hf.Registrar.Roles' attribute  |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.Registrar.Attributes     | List       | List of attributes that registrar is allowed to register                                                   |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.GenCRL                   | Boolean    | Identity is able to generate CRL if attribute value is true                                                |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.Revoker                  | Boolean    | Identity is able to revoke an identity and/or certificates if attribute value is true                           |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.AffiliationMgr           | Boolean    | Identity is able to manage affiliations if attribute value is true                                         |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+
| hf.IntermediateCA           | Boolean    | Identity is able to enroll as an intermediate CA if attribute value is true                                |
+-----------------------------+------------+------------------------------------------------------------------------------------------------------------+

注：アイデンティティを登録するときは、属性の名前と値の配列を指定します。 
配列が同じ名前の複数の配列要素を指定する場合、最後の要素のみが現在使用されています。 
つまり、複数の値を持つ属性は現在サポートされていません。

次のコマンドは、 **管理者ID** の資格情報を使用して、
「admin2」というアイデンティティを新規登録します。
属性情報としては、所属は「org1.department1」、「hf.Revoker」属性の値が「true」、「admin」属性の値が「true」です。 
「:ecert」サフィックスは、デフォルトで「admin」属性のデフォルト値を意味します。
その値はアイデンティティの登録証明書に挿入され、アクセス制御の決定に使用できます。


.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name admin2 --id.affiliation org1.department1 --id.attrs 'hf.Revoker=true,admin=true:ecert'

登録シークレットとも呼ばれるパスワードが表示されます。 
このパスワードは、アイデンティティを登録するために必要です。 
これにより、管理者はアイデンティティを登録し、アイデンティティを登録するための登録IDと秘密鍵を他の誰かに渡すことができます。

複数の属性を --id.attrs フラグの一部として指定できます。  
各属性はコンマで区切る必要があります。 
コンマを含む属性値の場合、属性はダブルクオーテーションで囲う必要があります。 
以下の例を参照してください。

.. code:: bash

    fabric-ca-client register -d --id.name admin2 --id.affiliation org1.department1 --id.attrs '"hf.Registrar.Roles=peer,client",hf.Revoker=true'

または

.. code:: bash

    fabric-ca-client register -d --id.name admin2 --id.affiliation org1.department1 --id.attrs '"hf.Registrar.Roles=peer,client"' --id.attrs hf.Revoker=true

クライアントの構成ファイルを編集することにより、registerコマンドで使用される任意のフィールドにデフォルト値を設定できます。 
たとえば、以下のような構成ファイルだったとします。  

.. code:: yaml

    id:
      name:
      type: client
      affiliation: org1.department1
      maxenrollments: -1
      attributes:
        - name: hf.Revoker
          value: true
        - name: anotherAttrName
          value: anotherAttrValue

次のコマンドは、コマンドラインから取得する「admin3」の登録IDで新しいアイデンティティを登録し、
残りは構成ファイルから取得されます
ID種別：「client」、所属：「org1.department1」 、および2つの属性：「hf.Revoker」および「anotherAttrName」。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name admin3

複数の属性を持つアイデンティティを登録するには、上記のように構成ファイルにすべての属性名と値を指定する必要があります。

`maxenrollments` を 0 に設定するか、構成から除外すると、CAの最大登録数を使用するようにアイデンティティが登録されます。 
さらに、登録されるアイデンティティの最大登録数は、CAの最大登録数を超えることはできません。 
たとえば、CAの最大登録値が 5 の場合、新しいアイデンティティの値は 5 以下である必要があります。
また、この値は -1 （無制限）に設定することはできません。

次のセクションではピアの登録に使用されるピアIDを登録しましょう。 
次のコマンドは、**peer1** ピアIDを登録します。 
サーバーにパスワードを生成させるのではなく、独自のパスワード（または秘密鍵）を指定することに注意してください。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name peer1 --id.type peer --id.affiliation org1.department1 --id.secret peer1pw

サーバー構成ファイルで指定されているleaf以外のaffliationでは、affiliationは大文字と小文字が区別されることに注意してください。
leaf affiliationは常に小文字で保存されます。 
たとえば、サーバー構成ファイルの所属セクションが次のようになっている場合：

.. code:: bash

    affiliations:
      BU1:
        Department1:
          - Team1
      BU2:
        - Department2
        - Department3

`BU1`, `Department1`, `BU2` は小文字で保存されます。 これは、Fabric CAが
Viper（訳者注：Golangの設定ファイル導入支援ライブラリ）を使用して構成を読み取るためです。
Viperはマップキーを大文字と小文字を区別せずに扱い、常に小文字の値を返します。 
アイデンティティを`Team1` affiliation で登録するには、以下に示すように、 `--id.affiliation` フラグで、`bu1.department1.Team1` を指定する必要があります。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client register --id.name client1 --id.type client --id.affiliation bu1.department1.Team1

Enrolling a peer identity
~~~~~~~~~~~~~~~~~~~~~~~~~

ピアIDの登録

ピアIDが正常に登録されたので、登録IDとシークレット（つまり、前のセクションのパスワード）を指定してピアを登録できます。 
これは、ブートストラップIDの登録に似ていますが、「-M」オプションを使用してHyperledger Fabric MSP（Membership Service Provider）のディレクトリ構造を設定する方法も示します。

次のコマンドは、 peer1 を登録します。
「-M」オプションの値を、ピアの core.yaml ファイルの「mspConfigPath」設定の内容を、ピアのMSPディレクトリへのパスに置き換えてください。
FABRIC_CA_CLIENT_HOME をピアのホームディレクトリに設定することもできます。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/peer1
    fabric-ca-client enroll -u http://peer1:peer1pw@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/msp

オーダラーの登録も同様です。
MSPディレクトリへのパスが、オーダラーの orderer.yaml ファイルの「LocalMSPDir」設定であることに注意してください。

fabric-ca-server によって発行されたすべての登録証明書には、次のような組織単位（Organization Unit 略して「OU」）があります。

1. OU階層のルートは、ID種別と等しい
2. ID の 所属 (affiliation) の各コンポーネントに、OUが追加されます

たとえば、ID種別が「peer」で、所属が `department1.team1` の場合、
アイデンティティのOU階層（枝葉からルートまで）は `OU=team1, OU=department1, OU=peer` です。

Getting Identity Mixer credential
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Identity Mixer資格情報の取得

Identity Mixer（Idemix）は、プライバシーを保護する認証および認証された属性の転送のための暗号化プロトコルスイートです。
Idemixを使用すると、クライアントは発行者（CA）の関与なしに検証者で認証でき、検証者が必要とする属性のみを選択的に開示できます。

ファブリックCAサーバーは、X509証明書に加えてIdemix資格情報を発行できます。 Idemix認証情報は、 ``/api/v1/idemix/credential`` APIエンドポイントに要求を送信することで要求できます。
これおよび他のFabric CAサーバーAPIエンドポイントの詳細については、`swagger-fabric-ca.json <https://github.com/hyperledger/fabric-ca/blob/master/swagger/swagger-fabric-ca.json>`_ を参照してください。

Idemixクレデンシャルの発行は2段階のプロセスです。 
最初に、空のボディを含むリクエストを ``/api/v1/idemix/credential`` APIエンドポイントに送信して、ナンスとCAのIdemix公開キーを取得します。 
次に、nonceとCAのIdemix公開鍵を使用して認証情報要求を作成し、認証情報要求を本文に含む別の要求を ``/api/v1/idemix/credential`` APIエンドポイントに送信します。
それにより、Idemix認証情報、認証情報失効情報（CRI:Credential Revocation Information）、 および属性の名前と値を得ます。 
現在、次の3つの属性のみがサポートされています。

- **OU** - アイデンティティの組織単位（Organization Unit）。 この属性の値は、IDの所属に設定されます。 たとえば、IDの所属が `dept1.unit1` の場合、OU属性は  `dept1.unit1` に設定されます。
- **IsAdmin** - アイデンティティが管理者であるかどうか。 この属性の値は、isAdmin 登録属性の値に設定されます。
- **EnrollmentID** - アイデンティティの登録ID。

Idemixクレデンシャルを取得するための2ステッププロセスのリファレンス実装については、
https://github.com/hyperledger/fabric-ca/blob/master/lib/client.go の `handleIdemixEnroll` 関数を参照できます。

``/api/v1/idemix/credential`` APIエンドポイントは、basic認証ヘッダーとトークン認証ヘッダーの両方を受け入れます。 
basic認証ヘッダーには、ユーザーの登録IDとパスワードが含まれている必要があります。
アイデンティティにすでにX509登録証明書がある場合、トークン認証ヘッダーの作成にも使用できます。

Hyperledger Fabricは、X509 と Idemix の両方の資格情報を使用してトランザクションに署名するクライアントをサポートしますが、ピアID と オーダラーID の X509 資格情報のみをサポートすることに注意してください。
前と同様に、アプリケーションは Fabric SDK を使用して、Fabric CA サーバーにリクエストを送信できます。 
SDK は、認証ヘッダーとリクエストペイロードの作成、および応答の処理に関連する複雑さを隠します

Getting Idemix CRI (Certificate Revocation Information)
-------------------------------------------------------

Idemix CRI（証明書失効情報）の取得

Idemix CRI（Credential Revocation Information : 資格情報失効情報）は、
目的が X509 CRL（Credential Revocation List : 証明書失効リスト）と似ており、以前に発行されたものを失効させます。 
ただし、いくつかの違いがあります。

X509では、発行者はエンドユーザーの証明書を失効させ、そのIDはCRLに含まれます。
検証者は、ユーザーの証明書がCRLに含まれているかどうかを確認し、含まれている場合は認証エラーを返します。
エンドユーザーは、検証者から認証エラーを受信する以外、この失効に関するプロセスには関与しません。

Idemixでは、エンドユーザーが関与します。
発行者は、X509にと同様にエンドユーザーの資格情報を失効させ、この失効の証拠がCRIに記録されます。
CRIはエンドユーザー（別名「証明者(prover)」）に与えられます。
次に、エンドユーザーは、CRIに従って資格情報が取り消されていないことの証明を生成します。
エンドユーザーは、CRIに従って証明を検証する検証者にこの証明を提供します。
検証を成功させるには、エンドユーザーと検証者が使用するCRIのバージョン（「エポック」と呼ばれる）が同じでなければなりません。
最新のCRIは、 ``/api/v1/idemix/cri`` APIエンドポイントにリクエストを送信することでリクエストできます。

登録要求がfabric-ca-serverによって受信され、失効ハンドルプールに失効ハンドルが残っていない場合、CRIのバージョンがインクリメントされます。
この場合、fabric-ca-serverは、CRIのエポックをインクリメントする失効ハンドルの新しいプールを生成する必要があります。
失効ハンドルプール内の失効ハンドルの数は、「idemix.rhpoolsize」サーバー設定プロパティを使用して設定できます。

Reenrolling an identity
~~~~~~~~~~~~~~~~~~~~~~~

アイデンティティの再登録

Suppose your enrollment certificate is about to expire or has been compromised.
You can issue the reenroll command to renew your enrollment certificate as follows.

登録証明書の有効期限が間もなく切れる、または侵害されたとします。
次のように、再登録コマンドを発行して、登録証明書を更新できます。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/peer1
    fabric-ca-client reenroll

Revoking a certificate or identity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

証明書またはアイデンティティを失効させる

An identity or a certificate can be revoked. Revoking an identity will revoke all
the certificates owned by the identity and will also prevent the identity from getting
any new certificates. Revoking a certificate will invalidate a single certificate.

アイデンティティまたは証明書を失効させることができます。
アイデンティティを失効させると、アイデンティティが所有するすべての証明書が取り消され、アイデンティティが新しい証明書を取得できなくなります。
証明書を失効させると、単一の証明書が無効になります。

In order to revoke a certificate or an identity, the calling identity must have
the ``hf.Revoker`` and ``hf.Registrar.Roles`` attribute. The revoking identity
can only revoke a certificate or an identity that has an affiliation that is
equal to or prefixed by the revoking identity's affiliation. Furthermore, the
revoker can only revoke identities with types that are listed in the revoker's
``hf.Registrar.Roles`` attribute.

証明書またはIDを取り消すには、呼び出しID (calling identity) に ``hf.Revoker`` および ``hf.Registrar.Roles`` 属性が必要です。
失効IDは、失効IDの所属と同等または接頭辞が付いた所属を持つ証明書またはIDのみを失効できます。
さらに、リボーカー (revoker : 取り消し者) は、リボーカーの ``hf.Registrar.Roles`` 属性にリストされているタイプのIDのみを取り消すことができます。

For example, a revoker with affiliation **orgs.org1** and 'hf.Registrar.Roles=peer,client'
attribute can revoke either a **peer** or **client** type identity affiliated with
**orgs.org1** or **orgs.org1.department1** but can't revoke an identity affiliated with
**orgs.org2** or of any other type.

たとえば、所属 **orgs.org1** および 'hf.Registrar.Roles=peer,client' 属性を持つリボーカーは、
**orgs.org1** または **orgs.org1.department1** に関連付けられている **ピア** または **クライアント** タイプのアイデンティティを失効させることができますが、
**orgs.org2** または、その他関連会社のIDを失効させることはできません。

The following command disables an identity and revokes all of the certificates
associated with the identity. All future requests received by the Fabric CA server
from this identity will be rejected.

次のコマンドは、アイデンティティを無効にし、そのアイデンティティに関連付けられているすべての証明書を失効させます。
このアイデンティティからFabric CAサーバーが受信する、今後のリクエストはすべて拒否されます。

.. code:: bash

    fabric-ca-client revoke -e <enrollment_id> -r <reason>

The following are the supported reasons that can be specified using ``-r`` flag:

以下は、 ``-r`` フラグを使用して指定できるサポートされている理由です。

  1. 不特定 (unspecified)
  2. 鍵の漏洩 (keycompromise)
  3. CA弱体化 (cacompromise)
  4. 所属変更 (affiliationchange)
  5. 破棄 (superseded)
  6. 運用停止 (cessationofoperation)
  7. 証明書保留 (certificatehold)
  8. CRL からの削除 (removefromcrl)
  9. 属性証明書の特権が剥奪されたことを示す (privilegewithdrawn)
  10. AA において信頼性が失われる事象が生じたことを示す (aacompromise)  

（訳者注：表 4-6 証明書失効理由と同等。 https://www.ipa.go.jp/security/pki/042.html）

For example, the bootstrap admin who is associated with root of the affiliation tree
can revoke **peer1**'s identity as follows:

たとえば、所属ツリーのルートに関連付けられているブートストラップ管理者は、
次のように **peer1** のアイデンティティを取り消すことができます。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=$HOME/fabric-ca/clients/admin
    fabric-ca-client revoke -e peer1

An enrollment certificate that belongs to an identity can be revoked by
specifying its AKI (Authority Key Identifier) and serial number as follows:

.. code:: bash

    fabric-ca-client revoke -a xxx -s yyy -r <reason>

For example, you can get the AKI and the serial number of a certificate using the openssl command
and pass them to the ``revoke`` command to revoke the said certificate as follows:

IDに属する登録証明書は、次のように、AKI（Authority Key Identifier）とシリアル番号を指定することにより失効できます。

.. code:: bash

   serial=$(openssl x509 -in userecert.pem -serial -noout | cut -d "=" -f 2)
   aki=$(openssl x509 -in userecert.pem -text | awk '/keyid/ {gsub(/ *keyid:|:/,"",$1);print tolower($0)}')
   fabric-ca-client revoke -s $serial -a $aki -r affiliationchange

The `--gencrl` flag can be used to generate a CRL (Certificate Revocation List) that contains all the revoked
certificates. For example, following command will revoke the identity **peer1**, generates a CRL and stores
it in the **<msp folder>/crls/crl.pem** file.

`--gencrl` フラグを使用して、すべての失効した証明書を含む CRL （証明書失効リスト） を生成できます。
たとえば、次のコマンドはID **peer1** のアイデンティティを失効させ、CRLを生成して **<msp folder>/crls/crl.pem** ファイルに保存します。

.. code:: bash

    fabric-ca-client revoke -e peer1 --gencrl

A CRL can also be generated using the `gencrl` command. Refer to the `Generating a CRL (Certificate Revocation List)`_
section for more information on the `gencrl` command.

CRLは、 `gencrl`コマンドを使用して生成することもできます。
`gencrl`コマンドの詳細については、 `Generating a CRL (Certificate Revocation List)`_ セクションを参照してください。

Generating a CRL (Certificate Revocation List)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CRL（証明書失効リスト）の生成

After a certificate is revoked in the Fabric CA server, the appropriate MSPs in Hyperledger Fabric must also be updated.
This includes both local MSPs of the peers as well as MSPs in the appropriate channel configuration blocks.
To do this, PEM encoded CRL (certificate revocation list) file must be placed in the `crls`
folder of the MSP. The ``fabric-ca-client gencrl`` command can be used to generate a CRL. Any identity
with ``hf.GenCRL`` attribute can create a CRL that contains serial numbers of all certificates that were revoked
during a certain period. The created CRL is stored in the `<msp folder>/crls/crl.pem` file.

Fabric CAサーバーで証明書を失効させた後、Hyperledger Fabricの適切なMSPも更新する必要があります。
これには、ピアのローカルMSPと適切なチャネル設定の部分にあるMSPの両方が含まれます。
これを行うには、PEMエンコードCRL（証明書失効リスト）ファイルをMSPの `crls` フォルダーに配置する必要があります。
``fabric-ca-client gencrl`` コマンドを使用してCRLを生成できます。
``hf.GenCRL`` 属性を持つアイデンティティは、特定の期間中に取り消されたすべての証明書のシリアル番号を含むCRLを作成できます。
作成されたCRLは、 `<msp folder>/crls/crl.pem` ファイルに保存されます。

The following command will create a CRL containing all the revoked certficates (expired and unexpired) and
store the CRL in the `~/msp/crls/crl.pem` file.

次のコマンドは、失効したすべての証明書（期限切れおよび期限切れなし）を含むCRLを作成し、CRLを `~/msp/crls/crl.pem` ファイルに保存します。

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=~/clientconfig
    fabric-ca-client gencrl -M ~/msp

The next command will create a CRL containing all certificates (expired and unexpired) that were revoked after
2017-09-13T16:39:57-08:00 (specified by the `--revokedafter` flag) and before 2017-09-21T16:39:57-08:00
(specified by the `--revokedbefore` flag) and store the CRL in the `~/msp/crls/crl.pem` file.

次のコマンドは、すべての失効した証明書（期限切れおよび期限切れなし）のCRLを作成し、`~/msp/crls/crl.pem` にファイルで保存します。
日付の条件は、2017-09-13T16：39：57-08：00 （ `--revokedafter` フラグで指定） 以降、
2017-09-21T16：39：57-08：00 （ `--revokedbefore` フラグで指定） となっています。 

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=~/clientconfig
    fabric-ca-client gencrl --caname "" --revokedafter 2017-09-13T16:39:57-08:00 --revokedbefore 2017-09-21T16:39:57-08:00 -M ~/msp


The `--caname` flag specifies the name of the CA to which this request is sent. In this example, the gencrl request is
sent to the default CA.

`--caname` フラグは、このリクエストが送信されるCAの名前を指定します。 
この例では、 gencrl リクエストは、デフォルトCAに送信されます。

The `--revokedafter` and `--revokedbefore` flags specify the lower and upper boundaries of a time period.
The generated CRL will contain certificates that were revoked in this time period. The values must be UTC
timestamps specified in RFC3339 format. The `--revokedafter` timestamp cannot be greater than the
`--revokedbefore` timestamp.

`--revokedafter` および `--revokedbefore` フラグは、期間の下限と上限を指定します。
生成されたCRLには、この期間に取り消された証明書が含まれます。 値は RFC3339 形式で指定された UTC タイムスタンプでなければなりません。 
`--revokedafter` タイムスタンプは `--revokedbefore` タイムスタンプより大きくすることはできません。

By default, 'Next Update' date of the CRL is set to next day. The `crl.expiry` CA configuration property
can be used to specify a custom value.

デフォルトでは、CRLの 'Next Update' 日付は翌日に設定されます。 
`crl.expiry` CA設定プロパティを使用して、カスタム値を指定できます。

The gencrl command will also accept `--expireafter` and `--expirebefore` flags that can be used to generate a CRL
with revoked certificates that expire during the period specified by these flags. For example, the following command
will generate a CRL that contains certificates that were revoked after 2017-09-13T16:39:57-08:00 and
before 2017-09-21T16:39:57-08:00, and that expire after 2017-09-13T16:39:57-08:00 and before 2018-09-13T16:39:57-08:00

gencrl コマンドは、これらのフラグで指定された期間内に失効する失効した証明書でCRLを生成するために使用できる 
`--expireafter` および `--expirebefore` フラグも受け入れます。
たとえば、次のコマンドは、2017-09-13T16：39：57-08：00 から 2017-09-21T16：39：57-08：00 の間に失効し、
その後、2017-09-13T16：39：57-08：00 から 2018-09-13T16：39：57-08：00 の間に期限切れになる証明書を含むCRLを生成します 

.. code:: bash

    export FABRIC_CA_CLIENT_HOME=~/clientconfig
    fabric-ca-client gencrl --caname "" --expireafter 2017-09-13T16:39:57-08:00 --expirebefore 2018-09-13T16:39:57-08:00  --revokedafter 2017-09-13T16:39:57-08:00 --revokedbefore 2017-09-21T16:39:57-08:00 -M ~/msp

Enabling TLS
~~~~~~~~~~~~

TLSを有効にする

This section describes in more detail how to configure TLS for a Fabric CA client.

The following sections may be configured in the ``fabric-ca-client-config.yaml``.

このセクションでは、Fabric CAクライアントのTLSを構成する方法について詳しく説明します。

次のセクションは ``fabric-ca-client-config.yaml`` で設定できます。

.. code:: yaml

    tls:
      # Enable TLS (default: false)
      enabled: true
      certfiles:
        - root.pem
      client:
        certfile: tls_client-cert.pem
        keyfile: tls_client-key.pem

The **certfiles** option is the set of root certificates trusted by the
client. This will typically just be the root Fabric CA server's
certificate found in the server's home directory in the **ca-cert.pem**
file.

The **client** option is required only if mutual TLS is configured on
the server.

**certfiles** オプションは、クライアントが信頼するルート証明書のセットです。
これは通常、 **ca-cert.pem** ファイル内のサーバーのホームディレクトリにあるルートファブリックCAサーバーの証明書になります。

**client** オプションは、サーバーで mutal TLS が構成されている場合にのみ必要です。

Attribute-Based Access Control
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

属性ベースのアクセス制御

Access control decisions can be made by chaincode (and by the Hyperledger Fabric runtime)
based upon an identity's attributes.  This is called
**Attribute-Based Access Control**, or **ABAC** for short.

アクセス制御の決定は、アイデンティティの属性に基づいてチェーンコード （および Hyperledger Fabric ランタイム） によって行うことができます。
これは、 **属性ベースのアクセス制御 : Attribute-Based Access Control** 、または略して **ABAC** と呼ばれます。

In order to make this possible, an identity's enrollment certificate (ECert)
may contain one or more attribute name and value.  The chaincode then
extracts an attribute's value to make an access control decision.

これを可能にするために、IDの登録証明書 （ECert） に1つ以上の属性名と値が含まれる場合があります。
次に、チェーンコードは属性の値を抽出して、アクセス制御の決定を行います。

For example, suppose that you are developing application *app1* and want a
particular chaincode operation to be accessible only by app1 administrators.
Your chaincode could verify that the caller's certificate (which was issued by
a CA trusted for the channel) contains an attribute named *app1Admin* with a
value of *true*.  Of course the name of the attribute can be anything and the
value need not be a boolean value.

たとえば、アプリケーション *app1* を開発しており、特定のチェーンコード操作に app1 管理者のみがアクセスできるようにするとします。
チェーンコードは、呼び出し元の証明書 （チャネルに対して信頼されている CA によって発行された） に *app1Admin* という名前の値が *true* の属性が含まれていることを確認できます。
もちろん、属性の名前は何でもよく、値はブール値である必要はありません。

So how do you get an enrollment certificate with an attribute?
There are two methods:

属性付きの登録証明書をどのように取得するか、それには2つの方法があります。

1.   When you register an identity, you can specify that an enrollment certificate
     issued for the identity should by default contain an attribute.  This behavior
     can be overridden at enrollment time, but this is useful for establishing
     default behavior and, assuming registration occurs outside of your application,
     does not require any application change.

1.   アイデンティティを登録するときに、アイデンティティに対して発行される登録証明書にデフォルトで属性が含まれるように指定できます。
     この動作は、登録時にオーバーライドできますが、これはデフォルトの動作を確立するのに役立ち、
     アプリケーションの外部で登録が行われる場合、アプリケーションの変更は不要です。

     The following shows how to register *user1* with two attributes:
     *app1Admin* and *email*.
     The ":ecert" suffix causes the *appAdmin* attribute to be inserted into user1's
     enrollment certificate by default, when the user does not explicitly request
     attributes at enrollment time.  The *email* attribute is not added
     to the enrollment certificate by default.

     以下は、 *user1* を、次の2つの属性、 *app1Admin* と  *email* で登録する方法を示しています。
     「：ecert」サフィックスにより、ユーザーが登録時に明示的に属性を要求しない場合、デフォルトで *appAdmin* 属性が user1 の登録証明書に挿入されます。
     デフォルトでは、 *email* 属性は登録証明書に追加されません。

.. code:: bash

     fabric-ca-client register --id.name user1 --id.secret user1pw --id.type client --id.affiliation org1 --id.attrs 'app1Admin=true:ecert,email=user1@gmail.com'

2. When you enroll an identity, you may explicitly request that one or more attributes
   be added to the certificate.
   For each attribute requested, you may specify whether the attribute is
   optional or not.  If it is not requested optionally and the identity does
   not possess the attribute, an error will occur.

2. IDを登録するときに、1つ以上の属性を証明書に追加することを明示的に要求できます。
   要求された属性ごとに、属性がオプションかどうかを指定できます。
   オプションで要求されず、IDに属性がない場合、エラーが発生します。

   The following shows how to enroll *user1* with the *email* attribute,
   without the *app1Admin* attribute, and optionally with the *phone*
   attribute (if the user possesses the *phone* attribute).

   以下に、*user1* を *email* 属性で登録し、*app1Admin* 属性なしで、
   オプションで *phone* 属性を登録する方法を示します（ユーザーが *phone* 属性を所有している場合）。

.. code:: bash

   fabric-ca-client enroll -u http://user1:user1pw@localhost:7054 --enrollment.attrs "email,phone:opt"

The table below shows the three attributes which are automatically registered for every identity.

次の表は、すべてのIDに対して自動的に登録される3つの属性を示しています。

===================================   =====================================
     Attribute Name                               Attribute Value
===================================   =====================================
  hf.EnrollmentID                        アイデンティティの登録ID
  hf.Type                                アイデンティティの種別
  hf.Affiliation                         アイデンティティの所属
===================================   =====================================

To add any of the above attributes **by default** to a certificate, you must
explicitly register the attribute with the ":ecert" specification.
For example, the following registers identity 'user1' so that
the 'hf.Affiliation' attribute will be added to an enrollment certificate if
no specific attributes are requested at enrollment time.  Note that the
value of the affiliation (which is 'org1') must be the same in both the
'--id.affiliation' and the '--id.attrs' flags.

上記の属性のいずれかを **デフォルトで** 証明書に追加するには、「:ecert」仕様で属性を明示的に登録する必要があります。
たとえば、以下は、登録時に特定の属性が要求されない場合に登録証明書に「hf.Affiliation」属性が追加されるように、アイデンティティ「user1」を登録します。
アフィリエーションの値（「org1」）は、「--id.affiliation」フラグと「--id.attrs」フラグの両方で同じでなければならないことに注意してください。

.. code:: bash

    fabric-ca-client register --id.name user1 --id.secret user1pw --id.type client --id.affiliation org1 --id.attrs 'hf.Affiliation=org1:ecert'

For information on the chaincode library API for Attribute-Based Access Control,
see `https://github.com/hyperledger/fabric/blob/release-1.4/core/chaincode/lib/cid/README.md <https://github.com/hyperledger/fabric/blob/release-1.4/core/chaincode/lib/cid/README.md>`_

属性ベースのアクセス制御用のチェーンコードライブラリAPIについては、以下を参照してください  
`https://github.com/hyperledger/fabric/blob/release-1.4/core/chaincode/lib/cid/README.md <https://github.com/hyperledger/fabric/blob/release-1.4/core/chaincode/lib/cid/README.md>`_

Dynamic Server Configuration Update
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

動的サーバー構成の更新

This section describes how to use fabric-ca-client to dynamically update portions
of the fabric-ca-server's configuration without restarting the server.

このセクションでは、 fabric-ca-client を使用して、サーバーを再起動せずに fabric-ca-server の構成の一部を
動的に更新する方法について説明します。

All commands in this section require that you first be enrolled by executing the
`fabric-ca-client enroll` command.

このセクションのすべてのコマンドでは、最初に「fabric-ca-client enroll」コマンドを実行して登録する必要があります。

Dynamically updating identities
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

アイデンティティを動的に更新する

This section describes how to use fabric-ca-client to dynamically update identities.

このセクションでは、 fabric-ca-client を使用してアイデンティティを動的に更新する方法について説明します。

An authorization failure will occur if the client identity does not satisfy all of the following:

クライアントのアイデンティティが次のすべてを満たしていない場合、認証エラーが発生します。

 - The client identity must possess the "hf.Registrar.Roles" attribute with a comma-separated list of
   values where one of the values equals the type of identity being updated; for example, if the client's
   identity has the "hf.Registrar.Roles" attribute with a value of "client", the client can update
   identities of type 'client', but not 'peer'.
   
 - クライアントのアイデンティティには、値の1つが更新されるIDのタイプに等しい値のコンマ区切りリストを持つ「hf.Registrar.Roles」属性が必要です。  
   たとえば、クライアントのアイデンティティに「hf.Registrar.Roles」属性があり、その値が値「client」の場合、
   クライアントは種別が「peer」ではなく「client」のアイデンティティを更新できます。

 - The affiliation of the client's identity must be equal to or a prefix of the affiliation of the identity
   being updated.  For example, a client with an affiliation of "a.b" may update an identity with an affiliation
   of "a.b.c" but may not update an identity with an affiliation of "a.c". If root affiliation is required for an
   identity, then the update request should specify a dot (".") for the affiliation and the client must also have
   root affiliation.

 - クライアントのアイデンティティの所属 (affiliation) は、更新されるアイデンティティの所属と等しいか、プレフィックスである必要があります。  
   たとえば、「a.b」の所属を持つクライアントは、「a.b.c」の所属を持つアイデンティティを更新できますが、「a.c」の所属を持つアイデンティティは更新できません。
   アイデンティティにルート所属が必要な場合、更新要求は所属にドット（"."）を指定する必要があり、クライアントにもルート所属が必要です。

The following shows how to add, modify, and remove an affiliation.

以下に、所属を追加、変更、削除する方法を示しています。

Getting Identity Information
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

アイデンティティ情報の取得

A caller may retrieve information on a identity from the fabric-ca server as long as the caller meets
the authorization requirements highlighted in the section above. The following command shows how to get an
identity.

発信者は、上記のセクションで強調表示されている承認要件を満たしている限り、fabric CA サーバーからアイデンティティに関する情報を取得できます。  
次のコマンドは、アイデンティティを取得する方法を示しています。

.. code:: bash

    fabric-ca-client identity list --id user1

A caller may also request to retrieve information on all identities that it is authorized to see by
issuing the following command.

呼び出し元は、次のコマンドを発行することにより、表示が許可されているすべてのアイデンティティに関する情報の取得を要求することもできます。

.. code:: bash

    fabric-ca-client identity list

Adding an identity
"""""""""""""""""""

アイデンティティを追加する

The following adds a new identity for 'user1'. Adding a new identity performs the same action as registering an
identity via the 'fabric-ca-client register' command. There are two available methods for adding a new identity.
The first method is via the `--json` flag where you describe the identity in a JSON string.

以下は、「user1」の新しいアイデンティティを追加します。 新しいアイデンティティを追加すると、
「fabric-ca-client register」コマンドを使用してアイデンティティを登録するのと同じアクションが実行されます。 
新しいアイデンティティを追加する方法は2つあります。
最初の方法は、JSON文字列でIDを記述する「--json」フラグを使用する方法です。

.. code:: bash

    fabric-ca-client identity add user1 --json '{"secret": "user1pw", "type": "client", "affiliation": "org1", "max_enrollments": 1, "attrs": [{"name": "hf.Revoker", "value": "true"}]}'

The following adds a user with root affiliation. Note that an affiliation name of "." means the root affiliation.

以下は、ルート所属のユーザーを追加します。 所属名が「.」であることに注意してください。  
ルート所属を意味します。

.. code:: bash

    fabric-ca-client identity add user1 --json '{"secret": "user1pw", "type": "client", "affiliation": ".", "max_enrollments": 1, "attrs": [{"name": "hf.Revoker", "value": "true"}]}'

The second method for adding an identity is to use direct flags. See the example below for adding 'user1'.

アイデンティティを追加する2番目の方法は、直接フラグを使用することです。  
「user1」の追加については、以下の例を参照してください。

.. code:: bash

    fabric-ca-client identity add user1 --secret user1pw --type client --affiliation . --maxenrollments 1 --attrs hf.Revoker=true

The table below lists all the fields of an identity and whether they are required or optional, and any default values they might have.

以下の表は、アイデンティティのすべてのフィールド、それらが必須またはオプションであるかどうか、およびそれらがデフォルト値として入る可能性のある値の一覧です。

+----------------+------------+------------------------+
| Fields         | Required   | Default Value          |
+================+============+========================+
| ID             | Yes        |                        |
+----------------+------------+------------------------+
| Secret         | No         |                        |
+----------------+------------+------------------------+
| Affiliation    | No         | Caller's Affiliation   |
+----------------+------------+------------------------+
| Type           | No         | client                 |
+----------------+------------+------------------------+
| Maxenrollments | No         | 0                      |
+----------------+------------+------------------------+
| Attributes     | No         |                        |
+----------------+------------+------------------------+


Modifying an identity
""""""""""""""""""""""

There are two available methods for modifying an existing identity. The first method is via the `--json` flag where you describe
the modifications in to an identity in a JSON string. Multiple modifications can be made in a single request. Any element of an identity that
is not modified will retain its original value.

NOTE: A maxenrollments value of "-2" specifies that the CA's max enrollment setting is to be used.

The command below make multiple modification to an identity using the --json flag.

.. code:: bash

    fabric-ca-client identity modify user1 --json '{"secret": "newPassword", "affiliation": ".", "attrs": [{"name": "hf.Regisrar.Roles", "value": "peer,client"},{"name": "hf.Revoker", "value": "true"}]}'

The commands below make modifications using direct flags. The following updates the enrollment secret (or password) for identity 'user1' to 'newsecret'.

.. code:: bash

    fabric-ca-client identity modify user1 --secret newsecret

The following updates the affiliation of identity 'user1' to 'org2'.

.. code:: bash

    fabric-ca-client identity modify user1 --affiliation org2

The following updates the type of identity 'user1' to 'peer'.

.. code:: bash

    fabric-ca-client identity modify user1 --type peer


The following updates the maxenrollments of identity 'user1' to 5.

.. code:: bash

    fabric-ca-client identity modify user1 --maxenrollments 5

By specifying a maxenrollments value of '-2', the following causes identity 'user1' to use
the CA's max enrollment setting.

.. code:: bash

    fabric-ca-client identity modify user1 --maxenrollments -2

The following sets the value of the 'hf.Revoker' attribute for identity 'user1' to 'false'.
If the identity has other attributes, they are not changed.  If the identity did not previously
possess the 'hf.Revoker' attribute, the attribute is added to the identity. An attribute may
also be removed by specifying no value for the attribute.

.. code:: bash

    fabric-ca-client identity modify user1 --attrs hf.Revoker=false

The following removes the 'hf.Revoker' attribute for user 'user1'.

.. code:: bash

    fabric-ca-client identity modify user1 --attrs hf.Revoker=

The following demonstrates that multiple options may be used in a single `fabric-ca-client identity modify`
command. In this case, both the secret and the type are updated for user 'user1'.

.. code:: bash

    fabric-ca-client identity modify user1 --secret newpass --type peer

Removing an identity
"""""""""""""""""""""

The following removes identity 'user1' and also revokes any certificates associated with the 'user1' identity.

.. code:: bash

    fabric-ca-client identity remove user1

Note: Removal of identities is disabled in the fabric-ca-server by default, but may be enabled
by starting the fabric-ca-server with the `--cfg.identities.allowremove` option.

Dynamically updating affiliations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This section describes how to use fabric-ca-client to dynamically update affiliations. The
following shows how to add, modify, remove, and list an affiliation.

Adding an affiliation
"""""""""""""""""""""""

An authorization failure will occur if the client identity does not satisfy all of the following:

  - The client identity must possess the attribute 'hf.AffiliationMgr' with a value of 'true'.
  - The affiliation of the client identity must be hierarchically above the affiliation being updated.
    For example, if the client's affiliation is "a.b", the client may add affiliation "a.b.c" but not
    "a" or "a.b".

The following adds a new affiliation named ‘org1.dept1’.

.. code:: bash

    fabric-ca-client affiliation add org1.dept1

Modifying an affiliation
"""""""""""""""""""""""""

An authorization failure will occur if the client identity does not satisfy all of the following:

  - The client identity must possess the attribute 'hf.AffiliationMgr' with a value of 'true'.
  - The affiliation of the client identity must be hierarchically above the affiliation being updated.
    For example, if the client's affiliation is "a.b", the client may add affiliation "a.b.c" but not
    "a" or "a.b".
  - If the '--force' option is true and there are identities which must be modified, the client
    identity must also be authorized to modify the identity.

The following renames the 'org2' affiliation to 'org3'.  It also renames any sub affiliations
(e.g. 'org2.department1' is renamed to 'org3.department1').

.. code:: bash

    fabric-ca-client affiliation modify org2 --name org3

If there are identities that are affected by the renaming of an affiliation, it will result in
an error unless the '--force' option is used. Using the '--force' option will update the affiliation
of identities that are affected to use the new affiliation name.

.. code:: bash

    fabric-ca-client affiliation modify org1 --name org2 --force

Removing an affiliation
"""""""""""""""""""""""""

An authorization failure will occur if the client identity does not satisfy all of the following:

  - The client identity must possess the attribute 'hf.AffiliationMgr' with a value of 'true'.
  - The affiliation of the client identity must be hierarchically above the affiliation being updated.
    For example, if the client's affiliation is "a.b", the client may remove affiliation "a.b.c" but not
    "a" or "a.b".
  - If the '--force' option is true and there are identities which must be modified, the client
    identity must also be authorized to modify the identity.

The following removes affiliation 'org2' and also any sub affiliations.
For example, if 'org2.dept1' is an affiliation below 'org2', it is also removed.

.. code:: bash

    fabric-ca-client affiliation remove org2

If there are identities that are affected by the removing of an affiliation, it will result
in an error unless the '--force' option is used. Using the '--force' option will also remove
all identities that are associated with that affiliation, and the certificates associated with
any of these identities.

Note: Removal of affiliations is disabled in the fabric-ca-server by default, but may be enabled
by starting the fabric-ca-server with the `--cfg.affiliations.allowremove` option.

Listing affiliation information
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An authorization failure will occur if the client identity does not satisfy all of the following:

  - The client identity must possess the attribute 'hf.AffiliationMgr' with a value of 'true'.
  - Affiliation of the client identity must be equal to or be hierarchically above the
    affiliation being updated. For example, if the client's affiliation is "a.b",
    the client may get affiliation information on "a.b" or "a.b.c" but not "a" or "a.c".

The following command shows how to get a specific affiliation.

.. code:: bash

    fabric-ca-client affiliation list --affiliation org2.dept1

A caller may also request to retrieve information on all affiliations that it is authorized to see by
issuing the following command.

.. code:: bash

    fabric-ca-client affiliation list

Manage Certificates
~~~~~~~~~~~~~~~~~~~~

This section describes how to use fabric-ca-client to manage certificates.

Listing certificate information
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The certificates that are visible to a caller include:

  - Those certificates which belong to the caller
  - If the caller possesses the ``hf.Registrar.Roles`` attribute or the ``hf.Revoker`` attribute with a value of ``true``,
    all certificates which belong to identities in and below the caller's affiliation. For example, if the client's
    affiliation is ``a.b``, the client may get certificates for identities who's affiliation
    is ``a.b`` or ``a.b.c`` but not ``a`` or ``a.c``.

If executing a list command that requests certificates of more than one identity, only certificates of identities
with an affiliation that is equal to or hierarchically below the caller's affiliation will be listed.

The certificates which will be listed may be filtered based on ID, AKI, serial number, expiration time, revocation time, notrevoked, and notexpired flags.

* ``id``: List certificates for this enrollment ID
* ``serial``: List certificates that have this serial number
* ``aki``: List certificates that have this AKI
* ``expiration``: List certificates that have expiration dates that fall within this expiration time
* ``revocation``: List certificates that were revoked within this revocation time
* ``notrevoked``: List certificates that have not yet been revoked
* ``notexpired``: List certificates that have not yet expired

You can use flags ``notexpired`` and ``notrevoked`` as filters to exclude revoked certificates and/or expired certificates from the result set.
For example, if you only care about certificates that have expired but have not been revoked you can use the ``expiration`` and ``notrevoked`` flags to
get back such results. An example of this case is provided below.

Time should be specified based on RFC3339. For instance, to list certificates that have expirations between
March 1, 2018 at 1:00 PM and June 15, 2018 at 2:00 AM, the input time string would look like 2018-03-01T13:00:00z
and 2018-06-15T02:00:00z. If time is not a concern, and only the dates matter, then the time part can be left
off and then the strings become 2018-03-01 and 2018-06-15.

The string ``now`` may be used to denote the current time and the empty string to denote any time. For example, ``now::`` denotes
a time range from now to any time in the future, and ``::now`` denotes a time range from any time in the past until now.

The following command shows how to list certificates using various filters.

List all certificates:

.. code:: bash

 fabric-ca-client certificate list

List all certificates by id:

.. code:: bash

 fabric-ca-client certificate list --id admin

List certificate by serial and aki:

.. code:: bash

 fabric-ca-client certificate list --serial 1234 --aki 1234

List certificate by id and serial/aki:

.. code:: bash

 fabric-ca-client certificate list --id admin --serial 1234 --aki 1234

List certificates that are neither revoker nor expired by id:

.. code:: bash

 fabric-ca-client certificate list --id admin --notrevoked --notexpired

List all certificates that have not been revoked for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --id admin --notrevoked

List all certificates have not expired for an id (admin):

The "--notexpired" flag is equivalent to "--expiration now::", which means certificates
will expire some time in the future.

.. code:: bash

 fabric-ca-client certificate list --id admin --notexpired

List all certificates that were revoked between a time range for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --id admin --revocation 2018-01-01T01:30:00z::2018-01-30T05:00:00z

List all certificates that were revoked between a time range but have not expired for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --id admin --revocation 2018-01-01::2018-01-30 --notexpired

List all revoked certificates using duration (revoked between 30 days and 15 days ago) for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --id admin --revocation -30d::-15d

List all revoked certificates before a time

.. code:: bash

 fabric-ca-client certificate list --revocation ::2018-01-30

List all revoked certificates after a time

.. code:: bash

 fabric-ca-client certificate list --revocation 2018-01-30::

List all revoked certificates before now and after a certain date

.. code:: bash

 fabric-ca-client certificate list --id admin --revocation 2018-01-30::now

List all certificate that expired between a time range but have not been revoked for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --id admin --expiration 2018-01-01::2018-01-30 --notrevoked

List all expired certificates using duration (expired between 30 days and 15 days ago) for an id (admin):

.. code:: bash

 fabric-ca-client certificate list --expiration -30d::-15d

List all certificates that have expired or will expire before a certain time

.. code:: bash

 fabric-ca-client certificate list --expiration ::2058-01-30

List all certificates that have expired or will expire after a certain time

.. code:: bash

 fabric-ca-client certificate list --expiration 2018-01-30::

List all expired certificates before now and after a certain date

.. code:: bash

 fabric-ca-client certificate list --expiration 2018-01-30::now

List certificates expiring in the next 10 days:

.. code:: bash

 fabric-ca-client certificate list --id admin --expiration ::+10d --notrevoked

The list certificate command can also be used to store certificates on the file
system. This is a convenient way to populate the admins folder in an MSP, The "-store" flag
points to the location on the file system to store the certificates.

Configure an identity to be an admin, by storing certificates for an identity
in the MSP:

.. code:: bash

 export FABRIC_CA_CLIENT_HOME=/tmp/clientHome
 fabric-ca-client certificate list --id admin --store msp/admincerts

Contact specific CA instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When a server is running multiple CA instances, requests can be directed to a
specific CA. By default, if no CA name is specified in the client request the
request will be directed to the default CA on the fabric-ca server. A CA name
can be specified on the command line of a client command using the ``caname``
filter as follows:

.. code:: bash

    fabric-ca-client enroll -u http://admin:adminpw@localhost:7054 --caname <caname>

`Back to Top`_

HSM
---
By default, the Fabric CA server and client store private keys in a PEM-encoded file,
but they can also be configured to store private keys in an HSM (Hardware Security Module)
via PKCS11 APIs. This behavior is configured in the BCCSP (BlockChain Crypto Service Provider)
section of the server’s or client’s configuration file.

Configuring Fabric CA server to use softhsm2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section shows how to configure the Fabric CA server or client to use a software version
of PKCS11 called softhsm (see https://github.com/opendnssec/SoftHSMv2).

After installing softhsm, make sure to set your SOFTHSM2_CONF environment variable to
point to the location where the softhsm2 configuration file is stored. The config file looks like

.. code::

  directories.tokendir = /tmp/
  objectstore.backend = file
  log.level = INFO

You can find example configuration file named softhsm2.conf under testdata directory.

Create a token, label it “ForFabric”, set the pin to ‘98765432’
(refer to softhsm documentation).



You can use both the config file and environment variables to configure BCCSP
For example, set the bccsp section of Fabric CA server configuration file as follows.
Note that the default field’s value is PKCS11.

.. code:: yaml

  #############################################################################
  # BCCSP (BlockChain Crypto Service Provider) section is used to select which
  # crypto library implementation to use
  #############################################################################
  bccsp:
    default: PKCS11
    pkcs11:
      Library: /usr/local/Cellar/softhsm/2.1.0/lib/softhsm/libsofthsm2.so
      Pin: 98765432
      Label: ForFabric
      hash: SHA2
      security: 256
      filekeystore:
        # The directory used for the software file-based keystore
        keystore: msp/keystore

And you can override relevant fields via environment variables as follows:

.. code:: bash

  FABRIC_CA_SERVER_BCCSP_DEFAULT=PKCS11
  FABRIC_CA_SERVER_BCCSP_PKCS11_LIBRARY=/usr/local/Cellar/softhsm/2.1.0/lib/softhsm/libsofthsm2.so
  FABRIC_CA_SERVER_BCCSP_PKCS11_PIN=98765432
  FABRIC_CA_SERVER_BCCSP_PKCS11_LABEL=ForFabric

`Back to Top`_

File Formats
------------

Fabric CA server's configuration file format
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A default configuration file is created in the server's home directory
(see `Fabric CA Server <#server>`__ section for more info). The following
link shows a sample :doc:`Server configuration file <serverconfig>`.

Fabric CA client's configuration file format
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A default configuration file is created in the client's home directory
(see `Fabric CA Client <#client>`__ section for more info). The following
link shows a sample :doc:`Client configuration file <clientconfig>`.

`Back to Top`_

Troubleshooting
---------------

1. If you see a ``Killed: 9`` error on OSX when trying to execute
   ``fabric-ca-client`` or ``fabric-ca-server``, there is a long thread
   describing this problem at https://github.com/golang/go/issues/19734.
   The short answer is that to work around this issue, you can run the
   following command::

    # sudo ln -s /usr/bin/true /usr/local/bin/dsymutil

2. The error ``[ERROR] No certificates found for provided serial and aki`` will occur
   if the following sequence of events occurs:

   a. You issue a `fabric-ca-client enroll` command, creating an enrollment certificate (i.e. an ECert).
      This stores a copy of the ECert in the fabric-ca-server's database.
   b. The fabric-ca-server's database is deleted and recreated, thus losing the ECert from step 'a'.
      For example, this may happen if you stop and restart a docker container hosting the fabric-ca-server,
      but your fabric-ca-server is using the default sqlite database and the database file is not stored
      on a volume and is therefore not persistent.
   c. You issue a `fabric-ca-client register` command or any other command which tries to use the ECert from
      step 'a'.  In this case, since the database no longer contains the ECert, the
      ``[ERROR] No certificates found for provided serial and aki`` will occur.

   To resolve this error, you must enroll again by repeating step 'a'.  This will issue a new ECert
   which will be stored in the current database.

3. When sending multiple parallel requests to a Fabric CA Server cluster that uses shared sqlite3 databases,
   the server occasionally returns a 'database locked' error. This is most probably because the database
   transaction timed out while waiting for database lock (held by another cluster member) to be released.
   This is an invalid configuration because sqlite is an embedded database, which means the Fabric CA server
   cluster must share the same file via a shared file system, which introduces a SPoF (single point of failure),
   which contradicts the purpose of cluster topology. The best practice is to use either Postgres or MySQL
   databases in a cluster topology.

4. Suppose an error similar to
   ``Failed to deserialize creator identity, err The supplied identity is not valid, Verify() returned x509: certificate signed by unknown authority``
   is returned by a peer or orderer when using an enrollment certificate issued by the Fabric CA Server.  This indicates that
   the signing CA certificate used by the Fabric CA Server to issue certificates does not match a certificate in the `cacerts` or `intermediatecerts`
   folder of the MSP used to make authorization checks.

   The MSP which is used to make authorization checks depends on which operation you were performing when the error occurred.
   For example, if you were trying to install chaincode on a peer, the local MSP on the file system of the peer is used;
   otherwise, if you were performing some channel specific operation such as instantiating chaincode on a specific channel,
   the MSP in the genesis block or the most recent configuration block of the channel is used.

   To confirm that this is the problem, compare the AKI (Authority Key Identifier) of the enrollment certificate
   to the SKI (Subject Key Identifier) of the certificate(s) in the `cacerts` and `intermediatecerts` folder of appropriate MSP.
   The command `openssl x509 -in <PEM-file> -noout -text | grep -A1 "Authority Key Identifier"` will display the AKI and
   `openssl x509 -in <PEM-file> -noout -text | grep -A1 "Subject Key Identifier"` will display the SKI.
   If they are not equal, you have confirmed that this is the cause of the error.

   This can happen for multiple reasons including:

   a. You used `cryptogen` to generate your key material but did not start `fabric-ca-server` with the signing key and certificate generated
      by `cryptogen`.

      To resolve (assuming `FABRIC_CA_SERVER_HOME` is set to the home directory of your `fabric-ca-server`):

      1. Stop `fabric-ca-server`.
      2. Copy `crypto-config/peerOrganizations/<orgName>/ca/*pem` to `$FABRIC_CA_SERVER_HOME/ca-cert.pem`.
      3. Copy `crypto-config/peerOrganizations/<orgName>/ca/*_sk` to `$FABRIC_CA_SERVER_HOME/msp/keystore/`.
      4. Start `fabric-ca-server`.
      5. Delete any previously issued enrollment certificates and get new certificates by enrolling again.

   b. You deleted and recreated the CA signing key and certificate used by the Fabric CA Server after generating the genesis block.
      This can happen if the Fabric CA Server is running in a docker container, the container was restarted, and its home directory
      is not on a volume mount.  In this case, the Fabric CA Server will create a new CA signing key and certificate.

      Assuming that you can not recover the original CA signing key, the only way to recover from this scenario is to update the
      certificate in the `cacerts` (or `intermediatecerts`) of the appropriate MSPs to the new CA certificate.

.. Licensed under Creative Commons Attribution 4.0 International License
   https://creativecommons.org/licenses/by/4.0/
ƒ
