# Oracle Container Engine for Kubernetes(OKE)を利用したNetwork Policyデモ

## 概要

このデモは、2021/5/12開催のOCHaCafe(Oracle Cloud Hangout Cafe)Season4の第2回「Kubernetesのネットワーク」で実演したものです。  
このデモでは、Oracle Container Engine for Kubernetes(以降、OKEとします)を利用してNetworkPolicyの挙動を確認できるようになっています。  
今回は、WordPressおよびMySQLを利用してデモを行います。

### OKE環境の構築

OKEのプロビジョニングは、[こちらのハンズオン手順](https://oracle-japan.github.io/paasdocs/documents/containers/common/)の1~4に従って実施してください。

### Calicoのインストール

OKEでは、CNIプラグインとしてflannelを利用しています。  
flannelはNetworkPolicyには非対応ですが、OKEではCalicoを利用してNetworkPolicyを利用することが可能です。(Canal方式)  
OKEでのCalicoのインストール方法は[こちら](https://docs.oracle.com/ja-jp/iaas/Content/ContEng/Tasks/contengsettingupcalico.htm)のドキュメントに従って実施してください。

### 事前準備

ここでは、NetworkPolicyの挙動確認に利用するWordPressをインストールします。  

#### MySQLで利用するクレデンシャルの作成

WordPressで利用するMySQLのクレデンシャルを作成します。

```sh
kubectl create secret generic mysql --from-literal=password=<パスワード文字列>
```

`<パスワード文字列>`には任意のパスワードを指定してください。

#### MySQLのインストール

まずは、MySQLをインストールします。  
`01-mysql.yaml`をapplyします。

```sh
kubectl create -f 01-mysql.yaml
```

起動(Running状態)まで少し時間がかかります。  

```sh
[opc@oke-client ~]$ kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-mysql-5f8d9899fb-q5cff   1/1     Running   0          2m
[opc@oke-client ~]$ 
```

#### WordPressのインストール

次に、WordPressをインストールします。  
`02-wordpress.yaml`をapplyします。  

```sh
kubectl create -f 02-wordpress.yaml
```

こちらも、起動(Running状態)まで少し時間がかかります。  

```sh
[opc@oke-client ~]$ kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-6466984786-r9wsj         1/1     Running   0          2m
wordpress-mysql-5f8d9899fb-q5cff   1/1     Running   0          4m
[opc@oke-client ~]$ 
```

起動できたら、WordPressにアクセスできることを確認しておきます。  
WordPressへのアクセスはLoad Balancerを利用します。  
先ほどWordPressをインストールした際にLoad Balancerもプロビジョニングされているので、確認してみます。  

```sh
[opc@oke-client ~]$ kubectl get svc
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes        ClusterIP      10.96.0.1       <none>           443/TCP        20d
wordpress         LoadBalancer   10.96.109.198   158.101.67.120   80:31813/TCP   8m
wordpress-mysql   ClusterIP      10.96.122.255   <none>           3306/TCP       10m
[opc@oke-client ~]$
```

`wordpress`という名前のServiceができているので、`ExternalIP`(上記では、`158.101.67.120`)にブラウザからアクセスします。  
WordPressの初期インストール画面が表示されれば成功です。  
以降のNetworkPolicyの挙動確認のために初期インストールを済ませておきます。  

これで事前準備は完了です。  

### NetworkPolicyの適用

ここからは、NetworkPolicyを適用していきます。

#### 全てのPod間通信を遮断

デフォルトでは、全てのPodのIngress/Egressが全許可になっているので、まずは全ての通信を遮断してみます。  
`03-deny-all.yaml`をapplyします。  

```sh
kubectl create -f 03-deny-all.yaml
```

適用できてきるかどうかを確認します。  

```sh
[opc@oke-client ~]$ kubectl get netpol
NAME                   POD-SELECTOR    AGE
default-deny-all       <none>          2m
[opc@oke-client ~]$ 
```

詳細を確認してみます。  

```sh
[opc@oke-client ~]$ kubectl describe netpol default-deny-all
Name:         default-deny-all
Namespace:    default
Created on:   2021-05-06 03:00:51 +0000 GMT
Labels:       <none>
Annotations:  Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Ingress, Egress
[opc@oke-client ~]$ 
```

このように、全てのPodに対して`Selected pods are isolated for ingress(egress) connectivity`となっています。  
全てのPod間通信が遮断されているので、ブラウザでWordPressにアクセスしても無応答になります。  

#### WordPress(フロントエンドPod)のみ全ての通信を許可

次に、WordPress(フロントエンド)Podに対して全ての通信を許可してみます。  
`04-allow-all-frontend.yaml`をapplyします。  

```sh
kubectl create -f 04-allow-all-frontend.yaml
```

適用できてきるかどうかを確認します。  

```sh
[opc@oke-client ~]$ kubectl get netpol
NAME                   POD-SELECTOR    AGE
allow-all-ingress      tier=frontend   2m
default-deny-all       <none>          8m
[opc@oke-client ~]$ 
```

詳細を確認してみます。  

```sh
[opc@oke-client ~]$ kubectl describe netpol allow-all-ingress
Name:         allow-all-ingress
Namespace:    default
Created on:   2021-05-06 03:02:38 +0000 GMT
Labels:       <none>
Annotations:  Spec:
  PodSelector:     tier=frontend
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From: <any> (traffic not restricted by source)
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To: <any> (traffic not restricted by source)
  Policy Types: Ingress, Egress
[opc@oke-client ~]$ 
```

このように、WordPress Podにも付与している`tier=frontend`というラベルが付与されているPodに対しては、Ingress/Egressともに`traffic allowed to all ports`と`traffic not restricted by source`という形で、全ての通信が許可されています。  

この状態で、WordPressにブラウザでアクセスすると、フロントエンドへはアクセスできますが、依然としてMySQL Podへのアクセスは遮断されているので、データベース接続エラーが発生します。  

#### MySQL Podにたいして特定の通信を許可

最後に、MySQL Podに対して特定の通信のみを許可してみます。  
今回は、`WordPress Pod(「tier=frontend」というラベルが付与されたPod)から3306ポートへのIngress TCP通信のみ許可`というルールで適用します。  
`05-allow-mysql-ingress.yaml`をapplyします。  

```sh
kubectl create -f 05-allow-mysql-ingress.yaml
```

適用できてきるかどうかを確認します。  

```sh
[opc@oke-client ~]$ kubectl get netpol
NAME                   POD-SELECTOR    AGE
allow-all-ingress      tier=frontend   8m
default-deny-all       <none>          18m
allow-ingress-mysql   tier=mysql      2m
[opc@oke-client ~]$ 
```

詳細を確認してみます。  

```sh
[opc@oke-client ~]$ kubectl describe netpol allow-ingress-mysql
Name:         allow-ingress-mysql
Namespace:    default
Created on:   2021-05-06 03:07:44 +0000 GMT
Labels:       <none>
Annotations:  Spec:
  PodSelector:     tier=mysql
  Allowing ingress traffic:
    To Port: 3306/TCP
    From:
      PodSelector: tier=frontend
  Not affecting egress traffic
  Policy Types: Ingress
[opc@oke-client ~]$ 
```

このように、WordPress Podにも付与している`tier=frontend`というラベルが付与されているPodからの`3306/TCP`のIngress通信のみを許可しています。  

この状態で、WordPressにアクセスすると、正常にWordPressのページが表示されます。  
