### 概要


### 参考

minikube の install
https://kubernetes.io/ja/docs/tasks/tools/install-minikube/
https://kubernetes.io/ja/docs/setup/learning-environment/minikube/

Hello Minikube
https://kubernetes.io/ja/docs/tutorials/hello-minikube/




### セットアップ

#### ハイパーバイザのインストール

Hypervisor がインストールされていない場合はインストールする。
今回はMacOS で HyperKit を利用する想定。

```
$ which hyperkit
/usr/local/bin/hyperkit

$ ls -l /usr/local/bin/hyperkit 
/usr/local/bin/hyperkit@ -> /Applications/Docker.app/Contents/Resources/bin/com.docker.hyperkit
```
Docker for Macをインストールしていると　Hyperkitはあるはず。

#### minikube のインストール

```
$ brew install minikube
```

確認
```
$ minikube start --driver=hyperkit
$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

$ minikube delete
```

#### minikube Quick Start

```
$ minikube start 
$ kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
$ kubectl expose deployment hello-minikube --type=NodePort --port=8080
$ kubectl get pods,svc
NAME                                  READY   STATUS    RESTARTS   AGE
pod/hello-minikube-5d9b964bfb-q5gq6   1/1     Running   0          9m15s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/hello-minikube   NodePort    10.104.207.93   <none>        8080:30168/TCP   117s
service/kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          10m
```

以下のコマンドでアクセスURLが取得できる
```
minikube service hello-minikube --url
```

```
$ kubectl delete services hello-minikube
$ kubectl delete deployment hello-minikube
$ minikube stop
$ minikube delete
```