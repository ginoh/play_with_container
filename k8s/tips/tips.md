個人的な Tips

### dryrunを利用してresourceを作成する雛形にする
kubectlの `--dry-run`オプションを利用することで resourceを作成する際の雛形として利用する

参考: [generator](https://kubernetes.io/docs/reference/kubectl/conventions/#generators)

(メモ)  
以前は、`kubectl run`を利用した雛形作成の情報がいたるところにあるが、`run-pod/v1`以外の`generator`が非推奨になっており pod作成以外は`kubectl create`のdryrunをしたほうがよいと思われる。

Podの作成
```
$ kubectl run nginx --generator=run-pod/v1 --image nginx:1.13 --dry-run -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx:1.13
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
`--dry-run`の結果として出力された内容を編集してresourceのためのyamlを作成して利用する

deploymentの作成の場合
```
$ kubectl create  deployment nginx --image=nginx:1.13 --dry-run -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.13
        name: nginx
        resources: {}
status: {}
```
この方法で yamlを生成した際、配列を値としてもつハッシュは以下のように表現されていた
```
containers:
- image: nginx:1.13
```
上記の例でいうところのimageのインデントの位置が気になる人はインデントをいれたり何かしらのFormatterツールを利用して整形するとよい
例えば、Prettier でフォーマットすると以下のようになる
```
containers:
  - image: nginx:1.13
```

### debug

(TBD) kubectl alpha debug とか
