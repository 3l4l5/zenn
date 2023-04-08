---
title: "mariadb with TrueNAS + kubernetes の構築"
emoji: "🍙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [kubernetes, TrueNAS, MariaDB]
published: false
---
## 本記事ざっくりと

以下の要件で我が家のおうちk8sにmariaDB鯖を建てたました

* レプリケーションなどはしない単一のmariadbコンテナを作成
* 実際のデータはクラスタ外のTrueNASに保存する

このような要件で考えたときに、定期的にBatchでNASにデータをバックアップするなどのことも考えられましたが、今回はNASをマウントする形でデータを保持するような構成で作成しました。

:::message
本記事で紹介するのは、最小構成のDB鯖です。もし、本番環境で使用するような耐障害性の高いDBシステムを構築する必要がある場合は、[Kubernetes Operators for MariaDB](https://mariadb.com/kb/en/kubernetes-operators-for-mariadb/) などを使用してください。
:::

## 私の環境

* k8s version: 
* master node: 
* woker node: 
* TrueNAS: 

## TrueNASの設定

### TrueNASとは

[TrueNAS](https://www.truenas.com/)（旧: FreeNAS）とは、無料で使用できるNAS用のOSです。家に余っているパーツなどを用いて小規模のNASを構築したい場合などにうってつけです。無料とは思えないほど、様々な機能が充実しており、一般的なNASの用途（写真の保存など）はもちろんのこと、VMを建てられたり、ブロックストレージを払い出したり、RAIDを組むことができたり、、と様々な用途に使用することができます。

詳しいインストール手順などは本記事では触れませんが、インストール方法などについては、色々な方が説明してくださっているので、簡単に構築することができます。興味ある方はぜひ。

### TrueNASにISCSI共有ボリュームを作成する

## k8sの設定

### mariadb のベースイメージからマニフェストを作成

### pod からISCSIボリュームをマウントする

## ちゃんと作成できているか確認

## 参考にさせていただいた記事等