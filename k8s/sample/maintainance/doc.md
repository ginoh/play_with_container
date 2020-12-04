### スケジューリング対象の除外と復帰

ノードは `SchedulingEnabled` と `SchedulingDisabled` のステータスを持つ。  
`SchedulingDisabled` になったノードはノード上でPodが新規に作成されなくなる。  
デフォルトは `SchedulingEnabled`

#### 除外
```
$ kubectl get nodes 
NAME                 STATUS   ROLES    AGE   VERSION
kind-control-plane   Ready    master   18d   v1.17.0
kind-worker          Ready    <none>   18d   v1.17.0
kind-worker2         Ready    <none>   18d   v1.17.0

// 除外
$ kubectl cordon kind-worker2 
node/kind-worker2 cordoned

$ kubectl get nodes          
NAME                 STATUS                     ROLES    AGE   VERSION
kind-control-plane   Ready                      master   18d   v1.17.0
kind-worker          Ready                      <none>   18d   v1.17.0
kind-worker2         Ready,SchedulingDisabled   <none>   18d   v1.17.0

// 戻す
$ kubectl uncordon kind-worker2 
node/kind-worker2 uncordoned
```

### 排出
drain を利用して、ノードからPodを退避させる。除外処理も含んでいるため、  
cordon を事前に実行する必要はない。
