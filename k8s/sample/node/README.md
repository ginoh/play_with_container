### Node情報の取得
```
// yaml 形式
$ kubectl get nodes -o yaml

// json形式
$ kubectl get nodes -o json


// 特定のNode
$ kubectl get nodes
NAME                 STATUS   ROLES    AGE   VERSION
kind-control-plane   Ready    master   22m   v1.17.0
kind-worker          Ready    <none>   21m   v1.17.0
kind-worker2         Ready    <none>   21m   v1.17.0

$ kubectl get nodes kind-control-plane -o yaml
・
・
・

// describeを利用して取得
$ kubectl describe node
```

### Namespace

Namespaceによる論理的なクラスタの分離
 
 ```
 // e.g.
 $ kubectl create namespace sample-namespace
 ```