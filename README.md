# ros-websocket

## TL; DR

[ROS](http://wiki.ros.org) がインストールされた何らかのデバイスからクラウド上の WebSocket サーバーに対してデータを送信するサンプル。

全体のアーキテクチャは以下のとおり。

![全体アーキテクチャ](images/rosbridge_suite-WebSocket.svg "全体アーキテクチャ")

採用テクノロジーは以下のとおり。

* サーバー
  * Google Cloud Platform
    * Cloud DNS
    * Cloud Load Balancing
    * Compute Engine
    * Container Registry
  * Ruby v2.4.4
    * ~~[EM-WebSocket](https://github.com/igrigorik/em-websocket) v0.5.1~~ [faye-websocket](https://github.com/faye/faye-websocket-ruby) v0.10.7
* クライアント
  * [ROS Indigo](http://wiki.ros.org/indigo)
  * [rosbridge_suite package](http://wiki.ros.org/rosbridge_suite)
  * Python [websocket-client](https://github.com/websocket-client/websocket-client)


なお、以下の点は、まだ対応されていない。

* WebSocket クライアントを構成する [ROS パッケージ](http://wiki.ros.org/ROS/Tutorials/CreatingPackage)
  * ROS および WebSocket を検証するため、[rosbridge_suite package](http://wiki.ros.org/rosbridge_suite) を利用しているが、本来的には WebSocket クライアントとして機能する ROS パッケージを作成すべき（詳細は後述）
  * [こちら](https://qiita.com/ryskiwt/items/bf33dae63561feac0f5d) が参考になるか
* ~~WebSocket の SSL 化~~（対応済み）
  * ~~[Let's Encrypt](https://letsencrypt.org/) 導入予定~~
* CI/CD
* ~~Ruby の WebSocket 実装におけるライブラリ変更~~（対応済み）
  * ~~[EM-WebSocket は更新が停止している模様](https://github.com/igrigorik/em-websocket/releases)~~
  * ~~[WebSocket Ruby](https://github.com/imanel/websocket-ruby) や [
faye-websocket](https://github.com/faye/faye-websocket-node) が代替候補~~

最終的には、以下のアーキテクチャになるはず。

![全体アーキテクチャ（最終系）](images/ROS-WebSocket.svg "全体アーキテクチャ（最終系）")


## WebSocket Server on GCP

WebSocket サーバーを GCP 上に構築する。

### WebSocket サーバーアプリケーション

WebSocket サーバーを実装したアプリケーションは、[こちら](https://github.com/chronushadow/ruby-websocket)を参照。

### アプリケーションの稼働環境

GCP のコンピューティングサービスにおける WebSocket サポートは以下のとおり。

|コンピューティングサービス|WebSocket サポート|参考情報|
|:--------------|:--:|--------|
|Compute Engine|○||
|Kubernetes Engine|○||
|App Engine|×|リクエストの処理方法 -サポート対象外の機能-https://cloud.google.com/appengine/docs/flexible/ruby/how-requests-are-handled|
|Cloud Funtions|×|FaaS であり、 WebSocket のような long-lived connections が必要なアプリには不適 |

今回は、簡易な構成であるため `Compute Engine` を選択。

なお、WebSocket サーバーアプリケーション以外に、多くのアプリケーションを稼働させる場合は、本番用途に耐えるコンテナオーケストレーションを実現すべく、 `Kubernetes Engine` の採用を検討した方が良いであろう。 

### WebSocket 通信の暗号化（wss）

WebSocket サーバーに対する通信をセキュアにするため、SSL暗号化を実施する。

`Cloud Load Balancing` には、負荷分散として以下の3種類がある。

|負荷分散の種類|WebSocket サポート|SSL Termination|Backend|
|:---------|:---------------:|:-------------:|:-----------|
|[HTTP(S) Load Balancing](https://cloud.google.com/load-balancing/docs/https/)|○|○|Instance Group|
|[TCP Proxy Load Balancing](https://cloud.google.com/load-balancing/docs/tcp/)|○|×|VM Instance or Instance Group|
|UDP Proxy Load Balancing|×|×|VM Instance|

通常は、Google も推奨している `HTTP(S) LB` を採用することが一般的である。
`HTTP(S) LB` で SSL 認証（`SSL Termination`）を行うかは、以下の点を検討する必要がある。

* SSL 認証を LB に集約するか
* LB からバックエンドへの通信を SSL 暗号化するか

`HTTP(S) LB` で SSL 認証を行わない場合、必然的にバックエンドで SSL 認証が必要となり、個々に証明書の更新や管理を対応することになる。

一方、 `HTTP(S) LB` で SSL 認証を行う場合においても、LB からバックエンドへの通信をセキュアに保つためには、再度、通信を SSL 暗号化する必要がある。 [Google も以下のように言及している](https://cloud.google.com/load-balancing/docs/ssl/)。

**Figure: Google Cloud Load Balancing with SSL termination**
> You can separately decide if you want SSL between the proxy and your backends or not. We recommend using SSL.

なお、今回は以下の理由および状況を鑑み、`TCP Proxy LB` を採用している。

* バックエンドまで通信をセキュアにするため
* バックエンドが VM Instance (not Instance Group)

また、SSL 認証は、WebSocket サーバー内に [Let's Encrypt](https://letsencrypt.org/) を導入して実現している。


#### 参考情報

1. `HTTP(S) LB` における WebSocket サポートは、[2017年4月末以降](https://cloud.google.com/compute/docs/load-balancing/http/#websocket_proxy_support) から\
以前は、[こちら](https://www.slideshare.net/fumihikoshiroyama/gcp-http) で言及されているように WebSocket はサポートされておらず、選択肢は `TCP Proxy LB` のみであった
2. [マイクロサービスで構成する場合、内部通信もセキュアにするため暗号化することが望ましい](https://www.infoq.com/jp/news/2016/12/microservices-security) とされており、`Let's Encrypt` は手段の一つ
3. マイクロサービスに必要な機能「サービスメッシュ」を実現する「Istio」においても、[内部通信は基本的に暗号化](https://istio.io/docs/concepts/security/mutual-tls/)

### 冗長構成

可用性および耐障害性の観点から、[GCE の マネージドインスタンスグループの自動スケーリング機能](https://cloud.google.com/compute/docs/autoscaler/) を利用して、サーバーを冗長構成にすることが望ましい。

冗長化に際しては、Redis の Pub/Sub 機能を利用して、ステートを外部管理する必要がある。詳細は[こちら](https://github.com/chronushadow/ruby-websocket)を参照。

### アプリケーションの管理

通常は CI/CD により、アプリケーションのビルドおよびデプロイを自動化する。その際、アプリケーションはコンテナ化され、コンテナレジストリに保存、実行環境へデプロイされることが一般的である。

そこで今回は、 WebSocket サーバーアプリケーションをコンテナ化し、`Container Registry` に格納し、`Compute Engine` から起動するコンテナとして設定し、アプリケーションを起動している。

## WebSocket Client at ROS

### WebSocket クライアントアプリケーション

WebSocket クライアントを実装したアプリケーションは、[こちら](https://github.com/chronushadow/websocket-client)を参照。

本来は、ROS パッケージとして、ROS において、サーバーへ送信するデータの生成元となるデバイス用ノードと WebSocket クライアント用ノードで Pub/Sub を構成し、メッセージ（データ）を受信したタイミングで、WebSocket サーバーに対して送信するようにする必要がある。

### Rosbridge_suite

ROS における WebSocket を検証するため、[Rosbrige_suite](http://wiki.ros.org/rosbridge_suite) を試行・導入。

[Rosbrige_suite をコンテナ化：chronushadow/rosbridge](https://github.com/chronushadow/rosbridge)

`Rosbrige_suite` のコンポーネント `Rosbridge server` は、WebSocket サーバーあるいは TCP サーバーの機能を提供する。
[roslibjs](http://wiki.ros.org/roslibjs) 等のクライアントから Web ベースで ROS を制御することが可能になる。