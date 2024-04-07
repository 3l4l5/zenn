---
title: "Prometheusでweb serverを監視してみた(with Docker Compose)"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Prometheus, docker, dockercompose]
published: true
---

## 本記事ざっくりと

docker compose を用いて APIサーバーを Prometheus で監視してみました。
この記事では、PrometheusでどのようにAPIサーバーを監視しているのかについて説明した後、docker-composeを用いてPrometheusを立ち上げてメトリクスを収集する方法について説明します。
最後には、実際にAPIに負荷をかけてみて、メトリクスが変化することを確かめます。

今回サンプルとして挙げているコードなどは以下のリポジトリにあります。もしよろしければご自分の環境で試してみてください！
https://github.com/3l4l5/test_prometheus

## 目次

* 環境
* Prometheusについて
* Prometheusが監視対象のメトリクスを取得する仕組み
* Web API + Prometheusをdocker composeで構築する手順
* 挙動チェック
* 最後に

## 環境

* PC: MacBookPro(Apple silicon)
* Docker: v.24.0.6

## Prometheusについて

Prometheusとは、OSSのコンピュータリソース監視ツールです。監視対象のサーバーのCPUやメモリ、IOについてなどのメトリクスを取り、ブラウザで表示することができます。
メトリクスの表示のみならず、監視対象が規定値に達した際の通知機能なども持ち合わせます。
PrometheusはPull型の監視システムで、監視対象のサーバーに対してリクエストを投げ情報を収取する方式をとっています。

![プロメテウスの概要図](/images/5e0e0bfbfefdeb/img4.png)

しかし、特に何も設定していないサーバーは、リクエストを投げてもCPU使用率などの情報を返してくれるはずがありません。 Prometheusはどのようにメトリクスを収集しているのでしょうか。

## Prometheusが監視対象のメトリクスを取得する仕組み

Prometheus（というより前述のPull型のシステム監視ツール）では、監視対象のサーバーにリクエストを投げた時に、コンピュータのリソースに関するメトリクスを返すサービスを常駐させておくことよって情報を取得します。このサービスのことをエージェントと呼びます。

![プロメテウスの概要図](/images/5e0e0bfbfefdeb/img5.png)

Prometheusのエージェントとしてよく使われるのが、node_exporterと呼ばれるもので、今回はこれを用いてメトリクスを取得できるようにしていきます。

## Web API + Prometheusをdocker composeで構築する手順

作業としては以下の順を追って行いました。今回は、最終的なcompose.yamlの中身にを用いて説明していきます。

1. 適当な web api と dockerfile を作成する
2. node_exporterを用いてweb apiのメトリクスを取得できるようにする
3. node_exporterに対してPrometheusがリクエストを投げるよう設定する

### 実際のcomposeファイル

実際に使用したcomposeファイルは以下のようになります。

```yml:compose.yml
services:
  server:
    build:
      dockerfile: backend.Dockerfile
    ports:
      - 8888:8888
  node_exporter:
    image: prom/node-exporter
    ports:
      - 9100:9100
    pid: "service:server" # service:serverのサイドカーコンテナとして配置する
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

### server

serverはシンプルなwebサーバーです。```/helloWorld```にリクエストを投げると```{"message": "Hello World!"}```を返します

### node_exporter

こちらが、先ほど説明した、Prometheusに監視対象のサーバーの情報を渡すエージェントであるnode_exporterです。
9100ポートを開放しており、9100ポートに対してリクエストを投げることによって、メトリクスの情報を返してくれるようになります。

コンテナイメージは原則プロセスを一つしか持たないようにするのが良いとされているため、serverコンテナにnode_exporterを配置するのではなく、別コンテナで建てています。

しかし、それではserverのリソースを取得することができなくなってしまいます。
そのため、```pid: "service:server"```とすることで、node_exporterのコンテナをwebサーバーの**サイドカーコンテナ**として建てるようにしました。
サイドカーコンテナとは、メインとなるコンテナのリソースを共有するコンテナであり、メインのコンテナに対して機能を付加する目的で作られます。このようにすることで、serverのリソースを参照できるようになります。

なぜ原則1プロセスなのかについては以下の記事で詳しく説明していただいているため、紹介させていただきます。
[1コンテナ複数プロセスはやめておいた方が良い話](https://qiita.com/kazurego7/items/57f5fb80b4783b7633a1)

### prometheus

これが Prometheusのコンテナです。ポイントとしては、ルートディレクトリに存在する Prometheusの設定ファイルである```prometheus.yml```をマウントしている点です。
prometheus.ymlは以下の内容になっています。

```yml:./prometheus.yml
global:
  scrape_interval: 15s
  external_labels:
    monitor: 'codelab-monitor'
scrape_configs:
  - job_name: 'go api'               #①
    scrape_interval: 1s              #②
    static_configs:
      - targets:
        - 'node_exporter:9100'       #③

```

参考：https://prometheus.io/docs/prometheus/latest/getting_started/

設定していることは、

* 監視先の名前を```go api```とすること ①
* メトリクスの取得間隔を1秒にすること ②
* ```compose.yaml```で設定したnode_exporterを監視先にしていること ③

です。
docker composeのルーティング機能を用いることで、```compose.yml```で指定したサービス名でリクエスをを投げることができます。


### 構成のイメージ

以下に上で設定したdocker-composeの構成を示します

![構成イメージ](/images/5e0e0bfbfefdeb/img6.png)

## 挙動チェック

実際にコンテナを立ちあげ、各サービスに接続してみましょう。

```bash
$ docker compose build
$ docker compose up -d
```

### server

まずweb apiです
以下のコマンドを投げてみましょう

```bash
$ curl localhost:8888/helloWorld
> {"message":"Hello World!"}
```

ちゃんとAPIが立ち上がっていますね。いい感じ

### node_exporter

次に、node_exporterにリクエストを投げてみましょう

```bash
$ curl localhost:9100/metrics

> # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
> # TYPE go_gc_duration_seconds summary
> go_gc_duration_seconds{quantile="0"} 4.7416e-05
> go_gc_duration_seconds{quantile="0.25"} 0.000170917
> go_gc_duration_seconds{quantile="0.5"} 0.000441626
> go_gc_duration_seconds{quantile="0.75"} 0.001321833
> go_gc_duration_seconds{quantile="1"} 0.142523791
> ......
```

このように大きなレスポンスが帰ってきました。Prometheusはこの情報を定期的(今回は1秒ごと)保存していくことで、メトリクスをとっていきます。

### prometheus

次に、ブラウザから```localhost:9090```に接続します

ヘッダーからStatus>Targetsに進むことで、管理対象を確認することができます。
以下のようにnode_exporterが監視対象になっていることが確認できればセッティングが完了しています！

![接続確認](/images/5e0e0bfbfefdeb/img1.png)

## 実際にメトリクスをとってみる

では、実際にメトリクスを取得してみましょう！
ヘッダーから、「Graph」をクリックし、「Add panel」します。

Prometheusでは、この虫眼鏡マークのExpressionの欄にPromQLという形式のクエリを投げることでメトリクスを取得することができます。
実際に、現在時刻から直近のcpu使用時間の10秒平均を求めるクエリを入れてみましょう

```promql
rate(node_cpu_seconds_total{mode!="idle", mode!="softirq"}[10s])
```

![結果確認](/images/5e0e0bfbfefdeb/img2.png)

各コアの使用時間が見えました！いい感じですね！

では、ちゃんと動いているのかを確かめるために、実際にweb APIに負荷をかけてみましょう。
ここではabコマンドというAPIの負荷テスト用のコマンドラインツールを使用しています。Macの場合は標準でインストールされています。
このコマンドは、APIサーバーに対して5並列で30秒間リクエストを投げるというコマンドです。

``` sh
$ ab -c 5 -t 30 http://localhost:8888/helloworld
```

負荷テストが終了したら、もう一度確認してみましょう！

![結果確認](/images/5e0e0bfbfefdeb/img3.png)

ちゃんとピークが出ました！node_exporterがちゃんとApi serverのメトリクスを取得してくれていますね！

## 最後に

今回は、Prometheusというコンピューターリソースのモニタリングツールを用いて、実際のwebアプリに対してモニタリングを行ってみました。
本番環境ではPrometheusとwebアプリのコンテナが同じdocker composeで管理されていることなどはないとは思いますが、どのようなアーキテクチャで構築したとしても、エージェントを用意してPrometheusが情報をpullするという構成は変わりません。
また、Prometheusは今回のようなCPU時間の他にもネットワークについてや、ディスク、メモリについてなど、様々なメトリクスを取ることができたり、あらかじめ設定した通知先(slackなど)にルールに基づいた通知を行ったりなどを行うことができます。ぜひ、いろいろ調べてみて他のメトリクスを集計してみてください！


## 参考にさせていただいたもの

* [1コンテナ複数プロセスはやめておいた方が良い話| Qiita](https://qiita.com/kazurego7/items/57f5fb80b4783b7633a1)
* [サイドカーコンテナをdocker-compose.ymlで指定する方法| Qiita](https://qiita.com/ynott/items/d5994a7a06e0d10c8e68)
* [達人が教える Web パフォーマンスチューニング| 書籍](https://gihyo.jp/book/2022/978-4-297-12846-3)
