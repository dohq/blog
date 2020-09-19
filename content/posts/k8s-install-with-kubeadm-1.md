---
title: "kubeadmでk8sをインストール(single node)その1"
date: 2020-09-13T00:42:22+09:00
tags: ["k8s","linux"]
categories: ["tips"]
draft: true
---
BOSH環境はあるけど時代に取り残されるのもツライ…のでk8s環境を作る  
とりあえず  
* シングルノード
* pod network addon はcanal
* インストーラはkubeadm

BOSH環境があるので[kube-release](https://github.com/cloudfoundry-incubator/kubo-release)でも良かったんだけど出来るだけ素っぽいのがいいかなと思ってkubeadmにした。  
更に書くとkubeadmで構築する前にホスト汚したくないなー→やっぱコンテナk8sやろ！  
のアレで[rke](https://rancher.com/docs/rke/latest/en/)で構築したのだけれど「kube-proxy無いんだが？」「hyperkubeって素のkubeと何が違うんや？」と悩んでるのが面倒だった。  
ページめっちゃ長いやんってなったけど大概ログなので実は大した事はしていない。  
タイトルどおり途中まで

## 環境
```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04.5 LTS"
NAME="Ubuntu"
VERSION="18.04.5 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.5 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```
## 事前準備
### 対象ホストでnet.ipv4.ipv4_forwardを有効に
```
$ sudo -i
# vim /etc/sysctl.conf
net.ipv4.ipv4_forward = 1 # <- 追加
```
### aptリポジトリの追加
```
apt -y install apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | tee -a /etc/apt/sources.list.d/kubernetes.list
apt update
apt -y install kubeadm kubelet kubectl
```
`systemctl enable kubelet` しておく

### docker daemonの設定変更
最初これやらなくて  
`[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/`
ってWARNが出て気持ち悪かったので提示されてるリンク先に書いてあるとおりにやった。
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl restart docker
docker ps
```

### kubeadm init
こっからk8sのインストール  
```
kubeadm init --apiserver-advertise-address=172.16.1.15 --pod-network-cidr=10.244.0.0/16

W0914 07:53:53.614664   12845 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local t401-srv] and IPs [10.96.0.1 172.16.1.15]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost t401-srv] and IPs [172.16.1.15 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost t401-srv] and IPs [172.16.1.15 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.003113 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node t401-srv as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node t401-srv as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 9upefv.s6dmvgyqiq9fl4c0
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.1.15:6443 --token <<とーくん>> \
    --discovery-token-ca-cert-hash sha256:<<はっしゅ>>

```
**Your Kubernetes control-plane has initialized successfully!**  
って出ればOK  
一般ゆーざでもkubectlを使えるようにするコマンドも出してくれているので、これをやっとくと良し
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
NAME       STATUS     ROLES    AGE     VERSION
t401-srv   NotReady   master   5m43s   v1.19.1
```
まだこの時点ではSTATUSはNotReadyでOK

### pod network addonのデプロイ
`kubectl get po -A` するとcorednsが動いてないが、pod network addonが動いてないので正しい
```
(*'-') < kgpo -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-f9fd979d6-wpmkx            0/1     Pending   0          17m
kube-system   coredns-f9fd979d6-xthds            0/1     Pending   0          17m
kube-system   etcd-t410-srv                      1/1     Running   0          18m
kube-system   kube-apiserver-t410-srv            1/1     Running   0          18m
kube-system   kube-controller-manager-t410-srv   1/1     Running   0          18m
kube-system   kube-proxy-jp9xp                   1/1     Running   0          17m
kube-system   kube-scheduler-t410-srv            1/1     Running   0          18m
```
ので、`kubectl apply`する  
んだけど[kapp](https://get-kapp.io)経由で入れて管理してみる
```
(*'-') < kapp deploy -a canal -f https://docs.projectcalico.org/manifests/canal.yaml
Changes

Namespace    Name                                                 Kind                      Conds.  Age  Op      Op st.  Wait to    Rs  Ri
(cluster)    bgpconfigurations.crd.projectcalico.org              CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            bgppeers.crd.projectcalico.org                       CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            blockaffinities.crd.projectcalico.org                CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            calico-kube-controllers                              ClusterRole               -       -    create  -       reconcile  -   -
^            calico-kube-controllers                              ClusterRoleBinding        -       -    create  -       reconcile  -   -
^            calico-node                                          ClusterRole               -       -    create  -       reconcile  -   -
^            canal-calico                                         ClusterRoleBinding        -       -    create  -       reconcile  -   -
^            canal-flannel                                        ClusterRoleBinding        -       -    create  -       reconcile  -   -
^            clusterinformations.crd.projectcalico.org            CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            felixconfigurations.crd.projectcalico.org            CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            flannel                                              ClusterRole               -       -    create  -       reconcile  -   -
^            globalnetworkpolicies.crd.projectcalico.org          CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            globalnetworksets.crd.projectcalico.org              CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            hostendpoints.crd.projectcalico.org                  CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            ipamblocks.crd.projectcalico.org                     CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            ipamconfigs.crd.projectcalico.org                    CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            ipamhandles.crd.projectcalico.org                    CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            ippools.crd.projectcalico.org                        CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            kubecontrollersconfigurations.crd.projectcalico.org  CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            networkpolicies.crd.projectcalico.org                CustomResourceDefinition  -       -    create  -       reconcile  -   -
^            networksets.crd.projectcalico.org                    CustomResourceDefinition  -       -    create  -       reconcile  -   -
kube-system  calico-kube-controllers                              Deployment                -       -    create  -       reconcile  -   -
^            calico-kube-controllers                              ServiceAccount            -       -    create  -       reconcile  -   -
^            canal                                                DaemonSet                 -       -    create  -       reconcile  -   -
^            canal                                                ServiceAccount            -       -    create  -       reconcile  -   -
^            canal-config                                         ConfigMap                 -       -    create  -       reconcile  -   -

Op:      26 create, 0 delete, 0 update, 0 noop
Wait to: 26 reconcile, 0 delete, 0 noop

Continue? [yN]: y

8:20:01AM: ---- applying 24 changes [0/26 done] ----
8:20:01AM: create clusterrole/calico-kube-controllers (rbac.authorization.k8s.io/v1) cluster
8:20:01AM: create serviceaccount/calico-kube-controllers (v1) namespace: kube-system
8:20:01AM: create customresourcedefinition/ipamconfigs.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:01AM: create customresourcedefinition/ippools.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:01AM: create configmap/canal-config (v1) namespace: kube-system
8:20:02AM: create customresourcedefinition/ipamblocks.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:02AM: create customresourcedefinition/clusterinformations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:02AM: create customresourcedefinition/bgpconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:02AM: create clusterrole/flannel (rbac.authorization.k8s.io/v1) cluster
8:20:02AM: create clusterrolebinding/calico-kube-controllers (rbac.authorization.k8s.io/v1) cluster
8:20:02AM: create customresourcedefinition/ipamhandles.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:02AM: create clusterrole/calico-node (rbac.authorization.k8s.io/v1) cluster
8:20:02AM: create customresourcedefinition/globalnetworkpolicies.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:02AM: create clusterrolebinding/canal-calico (rbac.authorization.k8s.io/v1) cluster
8:20:02AM: create clusterrolebinding/canal-flannel (rbac.authorization.k8s.io/v1) cluster
8:20:03AM: create customresourcedefinition/globalnetworksets.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:03AM: create serviceaccount/canal (v1) namespace: kube-system
8:20:03AM: create customresourcedefinition/hostendpoints.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:03AM: create customresourcedefinition/bgppeers.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:03AM: create customresourcedefinition/felixconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:03AM: create customresourcedefinition/blockaffinities.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: create customresourcedefinition/networksets.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: create customresourcedefinition/kubecontrollersconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: create customresourcedefinition/networkpolicies.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ---- waiting on 24 changes [0/26 done] ----
8:20:04AM: ok: reconcile serviceaccount/calico-kube-controllers (v1) namespace: kube-system
8:20:04AM: ok: reconcile clusterrole/calico-kube-controllers (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/ipamconfigs.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile clusterrolebinding/canal-calico (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile clusterrolebinding/canal-flannel (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/hostendpoints.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile serviceaccount/canal (v1) namespace: kube-system
8:20:04AM: ok: reconcile customresourcedefinition/globalnetworksets.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile configmap/canal-config (v1) namespace: kube-system
8:20:04AM: ok: reconcile customresourcedefinition/bgpconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/ippools.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/globalnetworkpolicies.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile clusterrolebinding/calico-kube-controllers (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/networkpolicies.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/clusterinformations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/ipamblocks.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile clusterrole/calico-node (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/ipamhandles.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/blockaffinities.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/bgppeers.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile clusterrole/flannel (rbac.authorization.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/networksets.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/kubecontrollersconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ok: reconcile customresourcedefinition/felixconfigurations.crd.projectcalico.org (apiextensions.k8s.io/v1) cluster
8:20:04AM: ---- applying 2 changes [24/26 done] ----
8:20:05AM: create deployment/calico-kube-controllers (apps/v1) namespace: kube-system
8:20:05AM: create daemonset/canal (apps/v1) namespace: kube-system
8:20:05AM: ---- waiting on 2 changes [24/26 done] ----
8:20:05AM: ongoing: reconcile deployment/calico-kube-controllers (apps/v1) namespace: kube-system
8:20:05AM:  ^ Waiting for 1 unavailable replicas
8:20:05AM:  L ok: waiting on replicaset/calico-kube-controllers-65755b7cd5 (apps/v1) namespace: kube-system
8:20:05AM:  L ongoing: waiting on pod/calico-kube-controllers-65755b7cd5-zs2wr (v1) namespace: kube-system
8:20:05AM:     ^ Pending: Unschedulable (message: 0/1 nodes are available: 1 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didn't tolerate.)
8:20:05AM: ongoing: reconcile daemonset/canal (apps/v1) namespace: kube-system
8:20:05AM:  ^ Waiting for 1 unavailable pods
8:20:05AM:  L ongoing: waiting on pod/canal-ljphd (v1) namespace: kube-system
8:20:05AM:     ^ Pending: PodInitializing
8:20:05AM:  L ok: waiting on controllerrevision/canal-8445694ccb (apps/v1) namespace: kube-system
8:20:35AM: ongoing: reconcile deployment/calico-kube-controllers (apps/v1) namespace: kube-system
8:20:35AM:  ^ Waiting for 1 unavailable replicas
8:20:35AM:  L ok: waiting on replicaset/calico-kube-controllers-65755b7cd5 (apps/v1) namespace: kube-system
8:20:35AM:  L ongoing: waiting on pod/calico-kube-controllers-65755b7cd5-zs2wr (v1) namespace: kube-system
8:20:35AM:     ^ Pending: ContainerCreating
8:20:46AM: ongoing: reconcile daemonset/canal (apps/v1) namespace: kube-system
8:20:46AM:  ^ Waiting for 1 unavailable pods
8:20:46AM:  L ongoing: waiting on pod/canal-ljphd (v1) namespace: kube-system
8:20:46AM:     ^ Condition Ready is not True (False)
8:20:46AM:  L ok: waiting on controllerrevision/canal-8445694ccb (apps/v1) namespace: kube-system
8:20:52AM: ongoing: reconcile deployment/calico-kube-controllers (apps/v1) namespace: kube-system
8:20:52AM:  ^ Waiting for 1 unavailable replicas
8:20:52AM:  L ok: waiting on replicaset/calico-kube-controllers-65755b7cd5 (apps/v1) namespace: kube-system
8:20:52AM:  L ongoing: waiting on pod/calico-kube-controllers-65755b7cd5-zs2wr (v1) namespace: kube-system
8:20:52AM:     ^ Condition Ready is not True (False)
8:20:54AM: ok: reconcile daemonset/canal (apps/v1) namespace: kube-system
8:20:54AM: ---- waiting on 1 changes [25/26 done] ----
8:20:55AM: ok: reconcile deployment/calico-kube-controllers (apps/v1) namespace: kube-system
8:20:55AM: ---- applying complete [26/26 done] ----
8:20:55AM: ---- waiting complete [26/26 done] ----

Succeeded
Time: 0h:00m:58s
```
これでNodeもReadyだしcorednsも正常に稼動している筈
```
(*'-') < kgpo -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-65755b7cd5-zs2wr   1/1     Running   0          3m16s
kube-system   canal-ljphd                                2/2     Running   0          3m16s
kube-system   coredns-f9fd979d6-wpmkx                    1/1     Running   0          28m
kube-system   coredns-f9fd979d6-xthds                    1/1     Running   0          28m
kube-system   etcd-t410-srv                              1/1     Running   0          28m
kube-system   kube-apiserver-t410-srv                    1/1     Running   0          28m
kube-system   kube-controller-manager-t410-srv           1/1     Running   0          28m
kube-system   kube-proxy-jp9xp                           1/1     Running   0          28m
kube-system   kube-scheduler-t410-srv                    1/1     Running   0          28m

(*'-') < kgno
NAME       STATUS   ROLES    AGE   VERSION
t410-srv   Ready    master   29m   v1.19.1
```
### podデプロイ出来るようにする
今回はMaster Node1台で構築している為、デフォルトではpodのデプロイが出来ないようになっている。
```
(*'-') < kubectl describe node t410-srv
Name:               t410-srv
Roles:              master
...
Taints:             node-role.kubernetes.io/master:NoSchedule
...
```
これを外してpodのデプロイを出来るようにする。
```
(*'-') < kubectl taint nodes --all node-role.kubernetes.io/master-
node/t410-srv untainted
```
### テスト
この時点でちょっとテスト
```
(*'-') < cat nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
(*'-') < kapp d -a nginx -f nginx.yml
Target cluster 'https://172.16.1.15:6443' (nodes: t410-srv)

Changes

Namespace  Name              Kind        Conds.  Age  Op      Op st.  Wait to    Rs  Ri
default    nginx             Service     -       -    create  -       reconcile  -   -
^          nginx-deployment  Deployment  -       -    create  -       reconcile  -   -

Op:      2 create, 0 delete, 0 update, 0 noop
Wait to: 2 reconcile, 0 delete, 0 noop

Continue? [yN]: y

7:37:53AM: ---- applying 2 changes [0/2 done] ----
7:37:53AM: create service/nginx (v1) namespace: default
7:37:54AM: create deployment/nginx-deployment (apps/v1) namespace: default
7:37:54AM: ---- waiting on 2 changes [0/2 done] ----
7:37:54AM: ok: reconcile service/nginx (v1) namespace: default
7:37:54AM: ongoing: reconcile deployment/nginx-deployment (apps/v1) namespace: default
7:37:54AM:  ^ Waiting for generation 2 to be observed
7:37:54AM:  L ok: waiting on replicaset/nginx-deployment-797f9f767d (apps/v1) namespace: default
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-mmbdh (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-dlftp (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-6dq6w (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:54AM: ---- waiting on 1 changes [1/2 done] ----
7:37:54AM: ongoing: reconcile deployment/nginx-deployment (apps/v1) namespace: default
7:37:54AM:  ^ Waiting for 3 unavailable replicas
7:37:54AM:  L ok: waiting on replicaset/nginx-deployment-797f9f767d (apps/v1) namespace: default
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-mmbdh (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-dlftp (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:54AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-6dq6w (v1) namespace: default
7:37:54AM:     ^ Pending: ContainerCreating
7:37:59AM: ongoing: reconcile deployment/nginx-deployment (apps/v1) namespace: default
7:37:59AM:  ^ Waiting for 2 unavailable replicas
7:37:59AM:  L ok: waiting on replicaset/nginx-deployment-797f9f767d (apps/v1) namespace: default
7:37:59AM:  L ok: waiting on pod/nginx-deployment-797f9f767d-mmbdh (v1) namespace: default
7:37:59AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-dlftp (v1) namespace: default
7:37:59AM:     ^ Pending: ContainerCreating
7:37:59AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-6dq6w (v1) namespace: default
7:37:59AM:     ^ Pending: ContainerCreating
7:38:01AM: ongoing: reconcile deployment/nginx-deployment (apps/v1) namespace: default
7:38:01AM:  ^ Waiting for 1 unavailable replicas
7:38:01AM:  L ok: waiting on replicaset/nginx-deployment-797f9f767d (apps/v1) namespace: default
7:38:01AM:  L ok: waiting on pod/nginx-deployment-797f9f767d-mmbdh (v1) namespace: default
7:38:01AM:  L ongoing: waiting on pod/nginx-deployment-797f9f767d-dlftp (v1) namespace: default
7:38:01AM:     ^ Pending: ContainerCreating
7:38:01AM:  L ok: waiting on pod/nginx-deployment-797f9f767d-6dq6w (v1) namespace: default
7:38:04AM: ok: reconcile deployment/nginx-deployment (apps/v1) namespace: default
7:38:04AM: ---- applying complete [2/2 done] ----
7:38:04AM: ---- waiting complete [2/2 done] ----

Succeeded
Time: 0h:00m:13s

(*'-') < kgpo
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-797f9f767d-6dq6w   1/1     Running   0          15s
nginx-deployment-797f9f767d-dlftp   1/1     Running   0          15s
nginx-deployment-797f9f767d-mmbdh   1/1     Running   0          15s

(*'-') < kgsvc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   13m
nginx        ClusterIP   10.105.147.100   <none>        80/TCP    20s

(*'-') < kd svc nginx
Name:              nginx
Namespace:         default
Labels:            app=nginx
                   kapp.k14s.io/app=1600468671765235154
                   kapp.k14s.io/association=v1.d8db29936a13a6dcce60af89eb6e0cd2
Annotations:       kapp.k14s.io/identity: v1;default//Service/nginx;v1
                   kapp.k14s.io/original:
                     {"apiVersion":"v1","kind":"Service","metadata":{"labels":{"app":"nginx","kapp.k14s.io/app":"1600468671765235154","kapp.k14s.io/association...
                   kapp.k14s.io/original-diff-md5: 9e4358320402790700b5cb4f8f85c2d9
Selector:          app=nginx,kapp.k14s.io/app=1600468671765235154
Type:              ClusterIP
IP:                10.105.147.100
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.0.10:80,10.244.0.8:80,10.244.0.9:80
Session Affinity:  None
Events:            <none>

(*'-') < k port-forward nginx-deployment-797f9f767d-6dq6w 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Handling connection for 8080

(*'-') < curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
良さそう

### metrics-serverのデプロイ
AutoScaleはしないような気がするけど一応  
一部ファイルの書き換えが必要なので、マニフェストは一度ダウンロードして編集する
```
(*'-') < curl -fsL https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml -O

(*'-') < vim components.yaml
Time: 0h:00m:32s

(*'-') < diff -u <(curl -fsL https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml) components.yaml
--- /proc/self/fd/18    2020-09-14 16:06:09.295625922 +0900
+++ components.yaml     2020-09-14 16:05:42.831584498 +0900
@@ -88,6 +88,10 @@
         args:
           - --cert-dir=/tmp
           - --secure-port=4443
+        command:
+        - /metrics-server
+        - --kubelet-insecure-tls
+        - --kubelet-preferred-address-types=InternalIP
         ports:
         - name: main-port
           containerPort: 4443

(*'-') < kapp d -a metrics-server -f components.yaml

Changes

Namespace    Name                                  Kind                Conds.  Age  Op      Op st.  Wait to    Rs  Ri
(cluster)    metrics-server:system:auth-delegator  ClusterRoleBinding  -       -    create  -       reconcile  -   -
^            system:aggregated-metrics-reader      ClusterRole         -       -    create  -       reconcile  -   -
^            system:metrics-server                 ClusterRole         -       -    create  -       reconcile  -   -
^            system:metrics-server                 ClusterRoleBinding  -       -    create  -       reconcile  -   -
^            v1beta1.metrics.k8s.io                APIService          -       -    create  -       reconcile  -   -
kube-system  metrics-server                        Deployment          -       -    create  -       reconcile  -   -
^            metrics-server                        Service             -       -    create  -       reconcile  -   -
^            metrics-server                        ServiceAccount      -       -    create  -       reconcile  -   -
^            metrics-server-auth-reader            RoleBinding         -       -    create  -       reconcile  -   -

Op:      9 create, 0 delete, 0 update, 0 noop
Wait to: 9 reconcile, 0 delete, 0 noop

Continue? [yN]: y

7:47:55AM: ---- applying 7 changes [0/9 done] ----
7:47:55AM: create clusterrolebinding/system:metrics-server (rbac.authorization.k8s.io/v1) cluster
7:47:55AM: create clusterrole/system:aggregated-metrics-reader (rbac.authorization.k8s.io/v1) cluster
7:47:55AM: create clusterrolebinding/metrics-server:system:auth-delegator (rbac.authorization.k8s.io/v1) cluster
7:47:55AM: create rolebinding/metrics-server-auth-reader (rbac.authorization.k8s.io/v1) namespace: kube-system
7:47:55AM: create serviceaccount/metrics-server (v1) namespace: kube-system
7:47:55AM: create clusterrole/system:metrics-server (rbac.authorization.k8s.io/v1) cluster
7:47:56AM: create apiservice/v1beta1.metrics.k8s.io (apiregistration.k8s.io/v1beta1) cluster
7:47:56AM: ---- waiting on 7 changes [0/9 done] ----
7:47:56AM: ongoing: reconcile apiservice/v1beta1.metrics.k8s.io (apiregistration.k8s.io/v1beta1) cluster
7:47:56AM:  ^ Condition Available is not True (False)
7:47:56AM: ok: reconcile clusterrolebinding/system:metrics-server (rbac.authorization.k8s.io/v1) cluster
7:47:56AM: ok: reconcile serviceaccount/metrics-server (v1) namespace: kube-system
7:47:56AM: ok: reconcile clusterrolebinding/metrics-server:system:auth-delegator (rbac.authorization.k8s.io/v1) cluster
7:47:56AM: ok: reconcile clusterrole/system:aggregated-metrics-reader (rbac.authorization.k8s.io/v1) cluster
7:47:56AM: ok: reconcile clusterrole/system:metrics-server (rbac.authorization.k8s.io/v1) cluster
7:47:56AM: ok: reconcile rolebinding/metrics-server-auth-reader (rbac.authorization.k8s.io/v1) namespace: kube-system
7:47:56AM: ---- applying 2 changes [7/9 done] ----
7:47:56AM: create service/metrics-server (v1) namespace: kube-system
7:47:57AM: create deployment/metrics-server (apps/v1) namespace: kube-system
7:47:57AM: ---- waiting on 3 changes [6/9 done] ----
7:47:57AM: ok: reconcile service/metrics-server (v1) namespace: kube-system
7:47:57AM: ongoing: reconcile deployment/metrics-server (apps/v1) namespace: kube-system
7:47:57AM:  ^ Waiting for generation 2 to be observed
7:47:57AM:  L ok: waiting on replicaset/metrics-server-547f9bb789 (apps/v1) namespace: kube-system
7:47:57AM:  L ongoing: waiting on pod/metrics-server-547f9bb789-9k9kn (v1) namespace: kube-system
7:47:57AM:     ^ Pending: ContainerCreating
7:47:57AM: ---- waiting on 2 changes [7/9 done] ----
7:47:57AM: ongoing: reconcile deployment/metrics-server (apps/v1) namespace: kube-system
7:47:57AM:  ^ Waiting for 1 unavailable replicas
7:47:57AM:  L ok: waiting on replicaset/metrics-server-547f9bb789 (apps/v1) namespace: kube-system
7:47:57AM:  L ongoing: waiting on pod/metrics-server-547f9bb789-9k9kn (v1) namespace: kube-system
7:47:57AM:     ^ Pending: ContainerCreating
7:48:03AM: ongoing: reconcile deployment/metrics-server (apps/v1) namespace: kube-system
7:48:03AM:  ^ Waiting for 1 unavailable replicas
7:48:03AM:  L ok: waiting on replicaset/metrics-server-547f9bb789 (apps/v1) namespace: kube-system
7:48:03AM:  L ok: waiting on pod/metrics-server-547f9bb789-9k9kn (v1) namespace: kube-system
7:48:04AM: ok: reconcile deployment/metrics-server (apps/v1) namespace: kube-system
7:48:04AM: ---- waiting on 1 changes [8/9 done] ----
7:48:06AM: ok: reconcile apiservice/v1beta1.metrics.k8s.io (apiregistration.k8s.io/v1beta1) cluster
7:48:06AM: ---- applying complete [9/9 done] ----
7:48:06AM: ---- waiting complete [9/9 done] ----

Succeeded
Time: 0h:00m:19s

(*'-') < k top nodes
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
t410-srv   327m         2%     1349Mi          1%
```

良さそう
### cert-managerをイスントール
TLS証明書の発行とか管理をしてくれる君とのこと  
とりあえず[cert-manager](https://cert-manager.io/docs/)デプロイ
```
(*'-') < kapp d -a cert-manager -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml
Target cluster 'https://172.16.1.15:6443' (nodes: t410-srv)

Changes

Namespace     Name                                    Kind                            Conds.  Age  Op      Op st.  Wait to    Rs  Ri
(cluster)     cert-manager                            Namespace                       -       -    create  -       reconcile  -   -
^             cert-manager-cainjector                 ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-cainjector                 ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-certificates    ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-certificates    ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-challenges      ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-challenges      ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-clusterissuers  ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-clusterissuers  ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-ingress-shim    ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-ingress-shim    ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-issuers         ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-issuers         ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-controller-orders          ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-controller-orders          ClusterRoleBinding              -       -    create  -       reconcile  -   -
^             cert-manager-edit                       ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-view                       ClusterRole                     -       -    create  -       reconcile  -   -
^             cert-manager-webhook                    MutatingWebhookConfiguration    -       -    create  -       reconcile  -   -
^             cert-manager-webhook                    ValidatingWebhookConfiguration  -       -    create  -       reconcile  -   -
^             certificaterequests.cert-manager.io     CustomResourceDefinition        -       -    create  -       reconcile  -   -
^             certificates.cert-manager.io            CustomResourceDefinition        -       -    create  -       reconcile  -   -
^             challenges.acme.cert-manager.io         CustomResourceDefinition        -       -    create  -       reconcile  -   -
^             clusterissuers.cert-manager.io          CustomResourceDefinition        -       -    create  -       reconcile  -   -
^             issuers.cert-manager.io                 CustomResourceDefinition        -       -    create  -       reconcile  -   -
^             orders.acme.cert-manager.io             CustomResourceDefinition        -       -    create  -       reconcile  -   -
cert-manager  cert-manager                            Deployment                      -       -    create  -       reconcile  -   -
^             cert-manager                            Service                         -       -    create  -       reconcile  -   -
^             cert-manager                            ServiceAccount                  -       -    create  -       reconcile  -   -
^             cert-manager-cainjector                 Deployment                      -       -    create  -       reconcile  -   -
^             cert-manager-cainjector                 ServiceAccount                  -       -    create  -       reconcile  -   -
^             cert-manager-webhook                    Deployment                      -       -    create  -       reconcile  -   -
^             cert-manager-webhook                    Service                         -       -    create  -       reconcile  -   -
^             cert-manager-webhook                    ServiceAccount                  -       -    create  -       reconcile  -   -
^             cert-manager-webhook:dynamic-serving    Role                            -       -    create  -       reconcile  -   -
^             cert-manager-webhook:dynamic-serving    RoleBinding                     -       -    create  -       reconcile  -   -
kube-system   cert-manager-cainjector:leaderelection  Role                            -       -    create  -       reconcile  -   -
^             cert-manager-cainjector:leaderelection  RoleBinding                     -       -    create  -       reconcile  -   -
^             cert-manager:leaderelection             Role                            -       -    create  -       reconcile  -   -
^             cert-manager:leaderelection             RoleBinding                     -       -    create  -       reconcile  -   -

Op:      39 create, 0 delete, 0 update, 0 noop
Wait to: 39 reconcile, 0 delete, 0 noop

Continue? [yN]: y

10:56:23PM: ---- applying 25 changes [0/39 done] ----
10:56:23PM: create namespace/cert-manager (v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-certificates (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-issuers (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-edit (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create validatingwebhookconfiguration/cert-manager-webhook (admissionregistration.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-clusterissuers (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-challenges (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-orders (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrolebinding/cert-manager-controller-certificates (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrolebinding/cert-manager-cainjector (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrolebinding/cert-manager-controller-clusterissuers (rbac.authorization.k8s.io/v1) cluster
10:56:23PM: create clusterrole/cert-manager-controller-ingress-shim (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create customresourcedefinition/certificaterequests.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:25PM: create clusterrolebinding/cert-manager-controller-challenges (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create clusterrolebinding/cert-manager-controller-orders (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create clusterrolebinding/cert-manager-controller-ingress-shim (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create mutatingwebhookconfiguration/cert-manager-webhook (admissionregistration.k8s.io/v1) cluster
10:56:25PM: create customresourcedefinition/certificates.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:25PM: create clusterrole/cert-manager-view (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create clusterrolebinding/cert-manager-controller-issuers (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create clusterrole/cert-manager-cainjector (rbac.authorization.k8s.io/v1) cluster
10:56:25PM: create customresourcedefinition/challenges.acme.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: create customresourcedefinition/issuers.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: create customresourcedefinition/orders.acme.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: create customresourcedefinition/clusterissuers.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ---- waiting on 25 changes [0/39 done] ----
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-challenges (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-issuers (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile validatingwebhookconfiguration/cert-manager-webhook (admissionregistration.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-ingress-shim (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-issuers (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile mutatingwebhookconfiguration/cert-manager-webhook (admissionregistration.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-view (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-clusterissuers (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-orders (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-certificates (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-cainjector (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-challenges (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-ingress-shim (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-clusterissuers (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-cainjector (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrolebinding/cert-manager-controller-orders (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/certificaterequests.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-edit (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile namespace/cert-manager (v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/certificates.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ok: reconcile clusterrole/cert-manager-controller-certificates (rbac.authorization.k8s.io/v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/orders.acme.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/challenges.acme.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/clusterissuers.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ok: reconcile customresourcedefinition/issuers.cert-manager.io (apiextensions.k8s.io/v1) cluster
10:56:26PM: ---- applying 9 changes [25/39 done] ----
10:56:26PM: create role/cert-manager-webhook:dynamic-serving (rbac.authorization.k8s.io/v1) namespace: cert-manager
10:56:26PM: create rolebinding/cert-manager-webhook:dynamic-serving (rbac.authorization.k8s.io/v1) namespace: cert-manager
10:56:26PM: create rolebinding/cert-manager:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: create role/cert-manager-cainjector:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: create rolebinding/cert-manager-cainjector:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: create serviceaccount/cert-manager-cainjector (v1) namespace: cert-manager
10:56:26PM: create serviceaccount/cert-manager (v1) namespace: cert-manager
10:56:26PM: create serviceaccount/cert-manager-webhook (v1) namespace: cert-manager
10:56:26PM: create role/cert-manager:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: ---- waiting on 9 changes [25/39 done] ----
10:56:26PM: ok: reconcile rolebinding/cert-manager-webhook:dynamic-serving (rbac.authorization.k8s.io/v1) namespace: cert-manager
10:56:26PM: ok: reconcile rolebinding/cert-manager:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: ok: reconcile role/cert-manager-webhook:dynamic-serving (rbac.authorization.k8s.io/v1) namespace: cert-manager
10:56:26PM: ok: reconcile serviceaccount/cert-manager-cainjector (v1) namespace: cert-manager
10:56:26PM: ok: reconcile role/cert-manager:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: ok: reconcile role/cert-manager-cainjector:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: ok: reconcile rolebinding/cert-manager-cainjector:leaderelection (rbac.authorization.k8s.io/v1) namespace: kube-system
10:56:26PM: ok: reconcile serviceaccount/cert-manager (v1) namespace: cert-manager
10:56:26PM: ok: reconcile serviceaccount/cert-manager-webhook (v1) namespace: cert-manager
10:56:26PM: ---- applying 5 changes [34/39 done] ----
10:56:26PM: create service/cert-manager-webhook (v1) namespace: cert-manager
10:56:26PM: create service/cert-manager (v1) namespace: cert-manager
10:56:27PM: create deployment/cert-manager (apps/v1) namespace: cert-manager
10:56:27PM: create deployment/cert-manager-webhook (apps/v1) namespace: cert-manager
10:56:27PM: create deployment/cert-manager-cainjector (apps/v1) namespace: cert-manager
10:56:27PM: ---- waiting on 5 changes [34/39 done] ----
10:56:27PM: ok: reconcile service/cert-manager-webhook (v1) namespace: cert-manager
10:56:27PM: ok: reconcile service/cert-manager (v1) namespace: cert-manager
10:56:27PM: ongoing: reconcile deployment/cert-manager-cainjector (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for generation 2 to be observed
10:56:27PM:  L ok: waiting on replicaset/cert-manager-cainjector-84666fb587 (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-cainjector-84666fb587-2hnwr (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:27PM: ongoing: reconcile deployment/cert-manager (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for generation 2 to be observed
10:56:27PM:  L ok: waiting on replicaset/cert-manager-5749666d47 (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-5749666d47-jcwfk (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:27PM: ongoing: reconcile deployment/cert-manager-webhook (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for generation 2 to be observed
10:56:27PM:  L ok: waiting on replicaset/cert-manager-webhook-8647f4cdcd (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-webhook-8647f4cdcd-vlccm (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:27PM: ---- waiting on 3 changes [36/39 done] ----
10:56:27PM: ongoing: reconcile deployment/cert-manager-cainjector (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for 1 unavailable replicas
10:56:27PM:  L ok: waiting on replicaset/cert-manager-cainjector-84666fb587 (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-cainjector-84666fb587-2hnwr (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:27PM: ongoing: reconcile deployment/cert-manager (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for 1 unavailable replicas
10:56:27PM:  L ok: waiting on replicaset/cert-manager-5749666d47 (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-5749666d47-jcwfk (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:27PM: ongoing: reconcile deployment/cert-manager-webhook (apps/v1) namespace: cert-manager
10:56:27PM:  ^ Waiting for 1 unavailable replicas
10:56:27PM:  L ok: waiting on replicaset/cert-manager-webhook-8647f4cdcd (apps/v1) namespace: cert-manager
10:56:27PM:  L ongoing: waiting on pod/cert-manager-webhook-8647f4cdcd-vlccm (v1) namespace: cert-manager
10:56:27PM:     ^ Pending: ContainerCreating
10:56:39PM: ongoing: reconcile deployment/cert-manager-webhook (apps/v1) namespace: cert-manager
10:56:39PM:  ^ Waiting for 1 unavailable replicas
10:56:39PM:  L ok: waiting on replicaset/cert-manager-webhook-8647f4cdcd (apps/v1) namespace: cert-manager
10:56:39PM:  L ongoing: waiting on pod/cert-manager-webhook-8647f4cdcd-vlccm (v1) namespace: cert-manager
10:56:39PM:     ^ Condition Ready is not True (False)
10:56:45PM: ok: reconcile deployment/cert-manager-webhook (apps/v1) namespace: cert-manager
10:56:45PM: ---- waiting on 2 changes [37/39 done] ----
10:56:49PM: ok: reconcile deployment/cert-manager-cainjector (apps/v1) namespace: cert-manager
10:56:49PM: ---- waiting on 1 changes [38/39 done] ----
10:56:57PM: ok: reconcile deployment/cert-manager (apps/v1) namespace: cert-manager
10:56:57PM: ---- applying complete [39/39 done] ----
10:56:57PM: ---- waiting complete [39/39 done] ----

Succeeded
Time: 0h:00m:48s

(*'-') < kgpo -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5749666d47-jcwfk              1/1     Running   0          48s
cert-manager-cainjector-84666fb587-2hnwr   1/1     Running   0          48s
cert-manager-webhook-8647f4cdcd-vlccm      1/1     Running   0          48s
```
今回はここまで
