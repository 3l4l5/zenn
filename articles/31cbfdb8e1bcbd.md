---
title: "オンプレk8sのpodにSSLで通信をさせるための方法メモ"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 前提条件

- k8s上でingress が使用できること
- k8s上でservice typeでloadbalancerを使用することができること

## 概要

作成したserviceに紐付けたIngressがlet’s encryptを用いてSSL通信ができるように設定されるまでの方法メモ。詳しい原理にはあまり触れず、手順について解説する

## 参考にさせていただいた記事

- [Zenn: cert-manager基礎知識](https://zenn.dev/masaaania/articles/e54119948bbaa2)
    - cert managerについて詳しく書かれている
- [Qiita: Centos7サーバーで1からKubernetes環境を構築しIngressでの外部公開・TLSの設定まで行う](https://qiita.com/bindingpry/items/0299e4e063e6f871e40e#certificate%E3%81%AE%E4%BD%9C%E6%88%90)
    - k8sの構築から、Certmanagerの立て方まで詳しく説明されている。しかし、cert managerのversionが今使っているものと異なっていた。
- [Ingress - HTTPS通信(Cert Manager)](https://developer.mamezou-tech.com/containers/k8s/tutorial/ingress/https/)
    - tls通信をさせるために自己証明書でのcert-managerの設定方法とそれを発展させたLet’s encryptでのcert-managerの設定方法を解説している。チャレンジの方法がdnsではないため、僕の環境では認証できなかった。

## 設定までの流れ

1. Cert-managerをインストールする
2.  Issuerを作成する
3. Certificateをデプロイする
4. Ingressを作成し、SSL証明書を指定する

## 1. Cert-managerをapplyする

cert-managerとは、k8sで稼働する証明書の自動管理などを行なってくれるやつ。

以下コマンドでapplyできます。

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

もしくは、helmを用いて取得

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade cert-manager jetstack/cert-manager \
  --install --version 1.5.4 \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true \
  --wait
```

詳しいインストール方法は[公式ページ](https://cert-manager.io/docs/installation/kubectl/)を参照のこと

↓こんな感じになってれば、apply完了している

```bash
$ kubectl get all -n cert-manager
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-74f46787b6-h9dds              0/1     Evicted   0          4d13h
pod/cert-manager-74f46787b6-jssgq              1/1     Running   0          3d13h
pod/cert-manager-cainjector-748dc889c5-2wgqr   0/1     Evicted   0          4d13h
pod/cert-manager-cainjector-748dc889c5-6hj8c   1/1     Running   0          3d13h
pod/cert-manager-webhook-5b679f47d6-5569c      0/1     Evicted   0          4d13h
pod/cert-manager-webhook-5b679f47d6-xtf7w      1/1     Running   0          3d13h

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.105.49.56     <none>        9402/TCP   4d13h
service/cert-manager-webhook   ClusterIP   10.108.119.231   <none>        443/TCP    4d13h

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           4d13h
deployment.apps/cert-manager-cainjector   1/1     1            1           4d13h
deployment.apps/cert-manager-webhook      1/1     1            1           4d13h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-74f46787b6              1         1         1       4d13h
replicaset.apps/cert-manager-cainjector-748dc889c5   1         1         1       4d13h
replicaset.apps/cert-manager-webhook-5b679f47d6      1         1         1       4d13h
```

## 2. issuerを作成する

issuerとは、SSL証明書を準備してくる役割を持つもの。

issuerには

- 一つのnamespaceで使用することができる 「**Issuer**」
- 複数のnamespaceで使用することができる「**clusterIssuer**」

の二種類が存在する。今回は、普通のissuerを用いてSSL通信を行う

### 2-1. Issuerのデプロイ

 今回は、lets encryptの証明書を自動で入手＋管理をしてくれるようにissuerを作成する。

以下のyamlを作成する

letsencrypt-issuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: example@example.com # ドメイン所有者のメールアドレス
    privateKeySecretRef:
      name: letsencrypt-issuer
    solvers:
      - selector:
          dnsZones:
          - "example.example.com" #Route53で設定しているドメイン名を書く
        dns01:
          route53:
            region: ap-northeast-1
            accessKeyID: ************** # AWSのアクセスキーID
            secretAccessKeySecretRef:
              name: route53-credentials-secret #登録したSecretリソース(後述)
              key: secret-access-key
```

要点は以下

- spec.acme.privateKeySecretRef: ここで指定した名前のsecretが作成され、その中にSSL証明書などが入る
- spec.solvers: Issuerは指定されたドメインが本当に所有しているものかどうかを確かめる必要があり、実際に今回はroute53にこのドメインの所持者かどうかを問い合わせしている。

※他のroute53以外のサービスを使用する場合のやり方は公式が提供してくれているのでその方法で認証することができる。

[https://cert-manager.io/docs/configuration/acme/dns01/#:~:text=during DNS01 challenges.-,Supported DNS01 providers,-A number of](https://cert-manager.io/docs/configuration/acme/dns01/#:~:text=during%20DNS01%20challenges.-,Supported%20DNS01%20providers,-A%20number%20of)

※Google domainにissuerが問い合わせる方法を見つけることはできなかった。その代わりに、取得しているドメインのサブドメインをroute53などで管理することもできる。その方法については下記参照

[https://dev.classmethod.jp/articles/create-subdomain-on-route53/](https://dev.classmethod.jp/articles/create-subdomain-on-route53/)

### 2-2. Issuerのsolverが使用するaws key などの準備

cluster issuer に、ドメインの持ち主であることを証明するために、route 53の場合、cluster issuerにAWS access key と AWS accesskey secretを教えておく必要がある。

それは、2-1のyamlの、spec.solvers.dns01.route53の部分で指定することができる。

このような指定方法の場合、route53-credentials-secretというsecret内にsecret-access-keyとしてsecretを登録する必要がある。

具体的には以下のように設定する。

aws_secret_key.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: route53-credentials-secret
type: Opaque
data:
  secret-access-key: <BASE64 encorded aws secret key>
```

secret-access-keyの項目に、base64エンコードしたシークレットキーを登録することで、secretを作成することができる。

### 3-3 Issuer のデプロイ

以下コマンドでデプロイすることで、issuerをデプロイできる

```bash
kubectl apply -f aws_secret_key.yaml
kubectl apply -f cluster_issuer.yaml
```

実際にSSL証明書を使用したいnamespaceにapplyする必要がある

## 3. Certificateをデプロイする

certificateは実際にkeyを作成し、secretに鍵を保存してくれるやつ。

keyが作成されれば、ingressからその秘密鍵、公開鍵を用いて通信をすることができるようになる。以下にその設定を書く。

certificate.yaml

```yaml
apiVersion: 
cert-manager.io/v1
kind: Certificate
metadata:
  name: home-srkr-dev
spec:
  secretName: home-srkr-dev-tls
  dnsNames:
  - home.srkr.dev
  - "*.home.srkr.dev"
  issuerRef:
    name: letsencrypt-issuer
    kind: Issuer
  #secretTemplate:
  #  annotations:
  #    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
  #    reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "dev,staging,prod,argocd,kubeflow"  # Control destination namespaces
  #    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true" # Auto create reflection for matching namespaces
  #    reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "dev,staging,prod,argocd,kubeflow" # Control auto-reflection namespaces
```

要点

- spec.dnsNames: ここに、route53で指定しているドメインを記すことができる
- spec.issuerRef: ここで、先ほど作成したissuerを指定することで、鍵を作成することができる
- spec.secretTemplate: ClusterIssuerはそのままでは複数のnamespaceに対して鍵を用意できるわけではなく、このように拡張機能を用いて使用したいnamespaceを指定することで他のnamespaceでもSSL通信をすることができるようになる

このsertificateをデプロイすることで、issuerをもとに鍵を作成することができる。

以下コマンドでデプロイ

```yaml
kubectl apply -f certificate.yaml
kubectl apply -f certificate.yaml -n dev
kubectl apply -f certificate.yaml -n prod
kubectl apply -f certificate.yaml -n staging
kubectl apply -f certificate.yaml -n argocd
```

ここで、使用したいnamespaceにそれぞれデプロイしないと、SSL通信がうまくできない。（できる方法あるのかな）

ここで、以下コマンドを打つことで、keyが作成されていることを確認することができる

```bash
$ kubectl get secret
NAME                          TYPE                                  DATA   AGE
home-srkr-dev-tls             kubernetes.io/tls                     2      25h
```

先ほど指定した名前で作成されていることを確認できればおk

### 4. Ingressを作成し、SSL証明書を指定する

最後に、ingressを使用して、ドメインを与えましょう。

route53で取得したドメインでingressをたて、先ほど作成したcertificateを指定することで、SSL/TLS通信をすることができるようになる。

以下サンプルのingress

ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
		# 以下のアノテーションで作成したissuerと紐付けることにより、SSL通信を行えるようになる
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/tls-acme: "true"
    ingress.kubernetes.io/ssl-redirect: "false"
    cert-manager.io/cluster-issuer: "letsencrypt-issuer"
spec:
  tls:
  - hosts:
      - example.example.example.com #Route53で登録したAレコードのドメイン名
    secretName: home-srkr-dev-tls # certificateで指定したsecretの名前
  ingressClassName: "nginx"
  rules:
    - host: test.home.srkr.dev #Route53で登録したAレコードのドメイン名
      http:
        paths:
          - backend:
              service:
                name: nginx-service
                port:
                  number: 443
            path: /
            pathType: Prefix
```

要点

- metadata.annotations: ここにTLSを使用する旨と、issuerの名前を設定する必要がある
- spec.tls: ここで所持しているドメインと先ほど作成したcertmanagerが作成したkeyの名前を指定する

ファイルが作成できたら

```yaml
kubectl apply -f ingress.yaml
```

でingressをapplyし、先ほど設定したドメインで接続することができれば完了

※オンプレでやっている場合には、80 と443がingressのIPにポートフォワーディングされるように設定しておかないと接続できない

![スクリーンショット 2022-09-04 15.36.24.png](/images/31cbfdb8e1bcbd/k8s%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%81%ABssl%E3%81%A6%E3%82%99%E9%80%9A%E4%BF%A1%E3%81%95%E3%81%9B%E3%82%8B%20217f3fa24bc741b0b07abf5859d673cc/%25E3%2582%25B9%25E3%2582%25AF%25E3%2583%25AA%25E3%2583%25BC%25E3%2583%25B3%25E3%2582%25B7%25E3%2583%25A7%25E3%2583%2583%25E3%2583%2588_2022-09-04_15.36.24.png)

やったね。

![スクリーンショット 2022-09-04 15.36.58.png](/images/31cbfdb8e1bcbd/k8s%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%81%ABssl%E3%81%A6%E3%82%99%E9%80%9A%E4%BF%A1%E3%81%95%E3%81%9B%E3%82%8B%20217f3fa24bc741b0b07abf5859d673cc/%25E3%2582%25B9%25E3%2582%25AF%25E3%2583%25AA%25E3%2583%25BC%25E3%2583%25B3%25E3%2582%25B7%25E3%2583%25A7%25E3%2583%2583%25E3%2583%2588_2022-09-04_15.36.58.png)

ちゃんと証明書も確認できる

以上！ご指摘いただける点などございましたら、お手数ですがコメントにいただけると幸いです
