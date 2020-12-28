### エフェラメラルコンテナを利用したデバッグ

https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container
https://qiita.com/ryysud/items/abf0f0590a66d1b3d426


FeatureGateを有効化したクラスタの作成
```
cat kind-debug-cluster.yaml 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  EphemeralContainers: true
nodes:
  - role: control-plane
```
kubectl の version が 1.18以上を確認
```
kubectl version --client -o yaml
clientVersion:
  buildDate: "2020-10-14T12:50:19Z"
  compiler: gc
  gitCommit: 1e11e4a2108024935ecfcb2912226cedeafd99df
  gitTreeState: clean
  gitVersion: v1.19.3
  goVersion: go1.15.2
  major: "1"
  minor: "19"
  platform: darwin/amd64
```
デバッグ対象のシェルも何も入ってないコンテナを立ち上げる
```
$ kubectl run ephemeral-demo --image=k8s.gcr.io/pause:3.1 --restart=Never
```
exec できない
```
kubectl exec -i -t ephemeral-demo -- sh
error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "83756100758aa0b781387a5962ddaa8000532cadd07caaa5847f2df453a101a4": OCI runtime exec failed: exec failed: container_linux.go:370: starting container process caused: exec: "sh": executable file not found in $PATH: unknown
```
そこで `kubectl alpha debug`コマンドを実行

```
$ kubectl alpha debug -i -t ephemeral-demo --image=busybox --target=ephemeral-demo
Defaulting debug container name to debugger-bxg8s.
If you don't see a command prompt, try pressing enter.
/ # 
```
この状態で、`kubectl describe`で podの状態を確認してみると、Ephemeral Containers という項目に作成されたコンテナが確認できる。
```
・
・
Containers:
  ephemeral-demo:
    Container ID:   containerd://802cf9fe327fb144844920613b2d07549b4fab56231fa61eede71f2c69a7a150
    Image:          k8s.gcr.io/pause:3.1
    Image ID:       k8s.gcr.io/pause@sha256:f78411e19d84a252e53bff71a4407a5686c46983a2c2eeed83929b888179acea
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 13 Dec 2020 18:39:12 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-7tnz5 (ro)
Ephemeral Containers:
  debugger-bxg8s:
    Container ID:   containerd://f49c0a3f8f7eff926bb86e5e08583b8aedc0cce41f12777ccc2c524de9e6b283
    Image:          busybox
    Image ID:       docker.io/library/busybox@sha256:bde48e1751173b709090c2539fdf12d6ba64e88ec7a4301591227ce925f3c678
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 13 Dec 2020 18:41:06 +0900
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
・
・    
```

このコンテナは Process 名前空間を共有しているので、デバッグ対象のプロセスを確認できる。

```
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    7 root      0:00 sh
   14 root      0:00 ps
/ # 
```
また、 `/proc`経由で ファイルシステムや環境変数を確認できる
```
/ # ls /proc/1/root
dev    etc    pause  proc   sys
/ # 
```
このあたりは以下のドキュメントを参照  
https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/


### Pod のコピーを利用したデバッグ

動作中のPodではなく、Podをコピーして、コピーしたPodでデバッグするということも可能

デバッグ対象のコンテナを作成
```
$ kubectl run myapp --image=busybox --restart=Never -- sleep 1d
```
Pod のコピーを作成
```
$ kubectl alpha debug myapp -i -t --image=ubuntu --share-processes --copy-to=myapp-debug
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
myapp         1/1     Running   0          7m20s
myapp-debug   2/2     Running   0          62s
```

この状態で `myapp-debug`のコンテナを確認してみる。
```
$ kubectl describe myapp-debug

・
・
Containers:
  myapp:
    Container ID:  containerd://27638133536166520b25d608ffe8d32146b7d4fedb5ef30776922a054b397c06
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:bde48e1751173b709090c2539fdf12d6ba64e88ec7a4301591227ce925f3c678
    Port:          <none>
    Host Port:     <none>
    Args:
      sleep
      1d
    State:          Running
      Started:      Sun, 13 Dec 2020 19:13:14 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-7tnz5 (ro)
  debugger-zjjht:
    Container ID:   containerd://b625968a58d401b45f600093bd6943df6966772e7d6ba60e7c67b166bac3de64
    Image:          ubuntu
    Image ID:       docker.io/library/ubuntu@sha256:c95a8e48bf88e9849f3e0f723d9f49fa12c5a00cfc6e60d2bc99d87555295e4c
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 13 Dec 2020 19:13:33 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-7tnz5 (ro)
・
・
```
myapp Pod のコピーと、コピーされた Podにデバッグ用のコンテナがアタッチされている。

他にも、コピーした Podに与える command や imageを変更することも可能