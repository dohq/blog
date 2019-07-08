---
title: "k3s on Time4vps"
date: 2019-07-07T00:51:39+09:00
tags: ["k8s","k3s"]
categories: ["k8s"]
draft: false
---

# k8sクラスタ欲しい…欲しくない？
勉強の為にk8sのクラスタが欲しかった訳だが、  
GKEとかKESはマスターがマネージドだからなーとか  
AWSにデプロイとか重すぎてとか(主に費用的な意味で)  
ツラいなーと思ってたところ、今年の頭くらいに[k3s](http://k3s.io)が出てk8s動かすのに必要なリソースがググっと減った。  

## k8s改めk3s on VPS
最初は自宅の鯖に立てていたのだが、出先から(勉強会とか)で試しつつやろうとすると自宅のクソザコネットワーク(レオ○ット)では外からの接続がつらい。  
面倒なのでVPSに立てようと思いつつ安いところを探してたら[time4vps](http://time4vps.com)なるVPSサービスを見付けた。  
これが安い。リージョンが日本では無い為、レスポンスはあまり良くないが
```
OS: Ubuntu 18.04 (64-bit)
Processor: 2 x 2.6 GHz
Memory: 8192 MB
Hard disk: 80 GB
Bandwidth: 1000 Mbps (Monthly limit: 8 TB)
Fast SSD storage (1.00 EUR)
1 Gbps port speed (1.00 EUR )
```
のスペックで月1500円くらい(2019-07-07現在)  
今回はとりあえずお試しでSingle構成で構築  
ただし最低1ヶ月からなので、時間単位で課金にしたい場合は[ScaleWay](http://scaleway.com)が安そう。  
ここまで書いといてなんだけどぶっちゃけVPSに関しては蛇足で本番はk3sのセットアップについて

## セットアップ
### k3sインストール
えっ、1コマンドでk8sのインストールを！？  

出来らぁっ！  
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--tls-san <Public IP>" sh -"
```

### セットアップ確認
リモートホスト上で
```
kubectl get nodes
```
実行してSTATUSが **Ready** になってればOK  
`/etc/rancher/k3s/k3s.yaml`の中身を`$HOME/.kube/config`に書いてアドレスを修正しとくと手元のPCからkubectlコマンドが打ててハッピー

## ダッシュボード追加
k3sはminikubeなんかと違ってインストールしたてだとdashboardが見れない。のでインストール
```
(*'-') <  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```
でk8s上へのデプロイは完了だが、dashboard用adminユーザが必要な為これを作成する。
```
(*'-') < cat <<EOF > dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

(*'-') < kubectl apply -f dashboard-admin.yaml
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created

(*'-') < kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

(*'-') < kubectl proxy
```
最後のコマンドでズラズラっと出てきた**token**を使って  
[http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)  
にアクセスすればdashboardが見れる筈

### podデプロイテスト
適当なpod(replica-set)をデプロイする
```
(*'-') < kubectl create namespace demo
namespace/demo created

(*'-') < cat<<EOF > demo.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: whoami-deployment
  namespace: demo
  labels:
    system: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami-container
          image: containous/whoami
          ports:
          - containerPort: 80
EOF

(*'-') < kubectl apply -f demo.yaml
deployment.extensions/whoami-deployment created

(*'-') < kubectl get pods -n demo
NAME                                 READY   STATUS    RESTARTS   AGE
whoami-deployment-8446fcc466-pwncz   1/1     Running   0          4m54s
```
よさそう

### サービス動確
```diff
(*'-') < git diff demo.yaml
diff --git a/demo.yaml b/demo.yaml
index 0184f13..2c9884b 100644
--- a/demo.yaml
+++ b/demo.yaml
@@ -21,3 +21,22 @@ spec:
           image: containous/whoami
           ports:
           - containerPort: 80
+
+---
+# Service
+apiVersion: v1
+kind: Service
+metadata:
+  name: whoami-services
+  namespace: demo
+  labels:
+    system: demo
+spec:
+  ports:
+    - name: http
+      targetPort: 80
+      port: 32324
+      protocol: TCP
+  selector:
+    app: whoami
+  type: LoadBalancer
```

```
(*'-') < kubectl apply -f demo.yaml
deployment.extensions/whoami-deployment unchanged
service/whoami-services configured

(*'-') < curl <PUBLIC IP>:32324
Hostname: whoami-deployment-8446fcc466-k8bhz
IP: 127.0.0.1
IP: 10.42.0.13
GET / HTTP/1.1
Host: <PUBLIC IP>:32324
User-Agent: curl/7.65.1
Accept: */*
```
いいのではないでしょうか

### kubectl topしたい
初期状態だと
```
(*'-') < kubectl top node
Error from server (NotFound): the server could not find the requested resource (get services http:heapster:)

(*'-') < kubectl top pods
Error from server (NotFound): the server could not find the requested resource (get services http:heapster:)
```
どうやら[metrics-server](https://github.com/kubernetes-incubator/metrics-server)のデプロイが必要らしい。  
これ入れないとリソース使用率によるAutoScaleなどが出来ない。  
README読むと
```
# Kubernetes 1.7
$ kubectl create -f deploy/1.7/

# Kubernetes > 1.8
$ kubectl create -f deploy/1.8+/
```
でいけそうなのだが微妙にいけない。 ログを見ると名前解決がアレっぽい。  
調べてみると、以下のようにマニフェストにちょろっと追記してapplyするといいらしい
```diff
diff --git a/deploy/1.8+/metrics-server-deployment.yaml b/deploy/1.8+/metrics-server-deployment.yaml
index c2311b7..6231209 100644
--- a/deploy/1.8+/metrics-server-deployment.yaml
+++ b/deploy/1.8+/metrics-server-deployment.yaml
@@ -31,7 +31,10 @@ spec:
       - name: metrics-server
         image: k8s.gcr.io/metrics-server-amd64:v0.3.3
         imagePullPolicy: Always
+        command:
+        - /metrics-server
+        - --kubelet-insecure-tls
+        - --kubelet-preferred-address-types=InternalDNS,InternalIP,ExternalDNS,ExternalIP,Hostname
         volumeMounts:
         - name: tmp-dir
           mountPath: /tmp
```
(参考URL: [https://qiita.com/chataro0/items/28f8744e2781f730a0e6](https://qiita.com/chataro0/items/28f8744e2781f730a0e6)

```
(*'-') < kubectl top node
NAME                     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
<Name>                   53m          2%     518Mi           6%

(*'-') < kubectl top pods -n demo

NAME                                 CPU(cores)   MEMORY(bytes)
svclb-whoami-services-7phjn          0m           0Mi
whoami-deployment-8446fcc466-k8bhz   0m           1Mi
```
いいね

### ついでにhelmインストール
```
(*'-') < kubectl -n kube-system create serviceaccount tiller
serviceaccount/tiller created

(*'-') < kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding.rbac.authorization.k8s.io/tiller created

(*'-') < helm init --service-account tiller
$HELM_HOME has been configured at /home/dohq/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation

(*'-') < kubectl get deployments -n kube-system

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
coredns                1/1     1            1           106m
kubernetes-dashboard   1/1     1            1           89m
metrics-server         1/1     1            1           8m59s
tiller-deploy          1/1     1            1           3m26s ★
traefik                1/1     1            1           105m
```
おk

## 後半へー続く
疲れたので続きは次へ
