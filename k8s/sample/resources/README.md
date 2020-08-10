### リソース制限

コンテナ単位でリソースの制限が可能。  
制限の対象は CPU と Memory、Device Pluginsを利用して他のリソース制限も可能。

CPU の制限は 1vCPU = 1000 millicoresで計算する

### 下限と上限

下限は spec.containers[].resources.requestsに指定する。  
上限は spec.containers[].resources.limitsに指定する

Requestsを満たせないとスケジューリングされない。  
Limitsのリソースがない場合でもスケジューリングされるため、  
コンテナの負荷があがるとクラスタの負荷が高くなる。

### Node の割り当て済みリソースを確認
```
e.g. 
$ kubectl describe nodes kind-control-plane
・
・
・
Non-terminated Pods:         (7 in total)
  Namespace                  Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                          ------------  ----------  ---------------  -------------  ---
  kube-system                etcd-kind-control-plane                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         7d8h
  kube-system                kindnet-dj9th                                 100m (5%)     100m (5%)   50Mi (2%)        50Mi (2%)      7d8h
  kube-system                kube-apiserver-kind-control-plane             250m (12%)    0 (0%)      0 (0%)           0 (0%)         7d8h
  kube-system                kube-controller-manager-kind-control-plane    200m (10%)    0 (0%)      0 (0%)           0 (0%)         7d8h
  kube-system                kube-proxy-4slqq                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         7d8h
  kube-system                kube-scheduler-kind-control-plane             100m (5%)     0 (0%)      0 (0%)           0 (0%)         7d8h
  local-path-storage         local-path-provisioner-7745554f7f-4hkvk       0 (0%)        0 (0%)      0 (0%)           0 (0%)         7d8h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                650m (32%)  100m (5%)
  memory             50Mi (2%)   50Mi (2%)
  ephemeral-storage  0 (0%)      0 (0%)
Events: 
```

### LimitRange でのリソース制限
Podに対して、CPU/Memory のmin/max や デフォルトを設定
新規作成の Pod が対象で、既存の Pod には影響しない。

設定する対象は、Pod/Container/PersistentVolumeClaim。

### ResourceQuota による制限

TBD

### HorizontalPodAutoscaler

TBD