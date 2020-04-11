### kindとは
kind (Kubernetes in Docker)は Docker コンテナを Nodeとしてローカル環境で Kubernetesクラスタを構築可能とするツール.
マルチノードKubernetesクラスタの構築が可能。

個人的な実験や検証に利用できそうなので利用してみる

[公式ドキュメント](https://kind.sigs.k8s.io/)
[github](https://github.com/kubernetes-sigs/kind)

[kubectlのリファレンス](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

### install
```
export GO111MODULE="on"
go get sigs.k8s.io/kind@v0.7.0
```

### Quick Start + α
```
$ kind create cluster --name cluster1
Creating cluster "cluster1" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-cluster1"
You can now use your cluster with:

kubectl cluster-info --context kind-cluster1

Have a nice day! 👋

$ kind create cluster --name cluster2
$ kind get clusters
cluster1
cluster2

// 個人的な alias
$ alias kc
kc=kubectl

$ kc cluster-info --context kind-cluster1
Kubernetes master is running at https://127.0.0.1:32768
KubeDNS is running at https://127.0.0.1:32768/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

$ kc get nodes
NAME                     STATUS   ROLES    AGE   VERSION
cluster1-control-plane   Ready    master   23m   v1.17.0

$ kc run nginx --image nginx
$ kc get pods  
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5578584966-7gpd7   1/1     Running   0          8s

```
### マルチノード + NodePort
```
$ cat kind-node-port-sample.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 8888
- role: worker
- role: worker

$ kind create cluster --name node-port-cluster --config kind-node-port-sample.yaml
```
host(ローカル開発機) の 8888 を control-planeの node (のコンテナ)の 30000 にマッピングした

```
$ kc run nginx --image nginx --labels "app=sample-app"

$ kc create service nodeport sample-node-port-svc --tcp=80:80 --node-port=30000
$ kc edit services sample-node-port-svc
$ kc patch services sample-node-port-svc -p '{"spec":{"selector":{"app":"sample-app"}}}'
```

```
$ curl localhost:8888
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
・
・
・
```


### マルチノード + Ingress

[参考: 公式ドキュメントのingressのサンプル](https://kind.sigs.k8s.io/docs/user/ingress/)

```
$ cat kind-ingress-sample.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 1080
    protocol: TCP
  - containerPort: 443
    hostPort: 1443
    protocol: TCP
- role: worker
- role: worker
```
extraPortMappings で hostのportをコンテナにマッピングする

```
$ kind create cluster --name ing-sample --config kind-ingress-sample.yaml
$ kc get nodes
NAME                       STATUS   ROLES    AGE    VERSION
ing-sample-control-plane   Ready    master   2m2s   v1.17.0
ing-sample-worker          Ready    <none>   86s    v1.17.0
ing-sample-worker2         Ready    <none>   85s    v1.17.0

// Ingress nginx
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
limitrange/ingress-nginx created
```
サービスを作成する

```
$ kc apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

service/ingress-nginx created

$ kc get services -n ingress-nginx -o wide
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
ingress-nginx   NodePort   10.96.35.215   <none>        80:32566/TCP,443:32451/TCP   5h54m   app.kubernetes.io/name=ingress-nginx,app.kubernetes
.io/part-of=ingress-nginx
```
sampleに従いサービスを作ったが、このサンプルの最後の確認は  
ローカルから直接 ingressのコンテナにアクセスしているのでなくてもよいはず。

patch をあてる
```
$ kc patch deployments -n ingress-nginx nginx-ingress-controller -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx-ingress-controller","ports":[{"containerPort":80,"hostPort":80},{"containerPort":443,"hostPort":443}]}],"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'

deployment.apps/nginx-ingress-controller patched
```
[Using Ingress](https://kind.sigs.k8s.io/docs/user/ingress#using-ingress) に従いhttps://kind.sigs.k8s.io/docs/user/ingress/#using-ingress にある通りに Ingress を試す。

```
$ kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/usage.yaml
pod/foo-app created
service/foo-service created
pod/bar-app created
service/bar-service created
ingress.extensions/example-ingress created
```
curlで確認
```
curl localhost:1080/foo
foo
 curl localhost:1080/bar
bar

curl -k https://localhost:1443/foo
foo

curl -k https://localhost:1443/bar
bar
```
