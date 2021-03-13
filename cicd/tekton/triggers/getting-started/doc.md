### Tekton Triggers とは

イベントペイロードを、リソースのテンプレートへとマッピングすることで、
k8sリソースとしてモデル化、インスタンス化できるようにする

概要図
![Tekton Triggers](https://github.com/tektoncd/triggers/blob/master/images/TriggerFlow.png?raw=true "Tekton Triggers")

実装としては、カスタムリソースのコントローラであり、ペイロードから情報を抽出して、k8sリソースを作るということを行う


### Triggers 入門

[Getting started with Triggers](https://github.com/tektoncd/triggers/tree/v0.10.1/docs/getting-started)を参考にして試す

中身は、Github webhook によって PipelineRunをキックするという内容
ただし、以下はローカルクラスタを使う為、webhookリクエストを擬似的に自分でリクエストする

#### Tekton Triggers の超基本

(TBD)
* EventListener
* Trigger
* TriggerBinding
* TriggerTemplate

#### 環境準備


```
// k8s cluster
$ ./../../k8s/kind/kind-with-registry.sh tekton-triggers

// Tekton
$ kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

// Triggers
$ kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

$ kubectl -n tekton-pipelines get pods
NAME                                                 READY   STATUS              RESTARTS   AGE
tekton-pipelines-controller-fc64466f6-kxf28          1/1     Running             0          75s
tekton-pipelines-webhook-56d5b66c98-dkg8f            1/1     Running             0          75s
tekton-triggers-controller-787f6b449-zg9zf           0/1     ContainerCreating   0          7s
tekton-triggers-core-interceptors-5fcdf79666-z682v   0/1     ContainerCreating   0          7s
tekton-triggers-webhook-599898d4d7-sdjc7             0/1     ContainerCreating   0          6s
```

#### getting-started

今回はローカルクラスタでやるため、GithubのWebhookをインターネットから受け付けることはしない  
curl により payload を直接 POST する

```
// namespace
$ kubectl create namespace getting-started

// admin role (Trigger が pipelinerunなど生成するために必要な rbac)
$ kubectl -n getting-started apply -f rbac.yaml
```

pipeline等 apply。いくつか公式にあったものから修正している.（2021/02)
```
// pipeline
$ kubectl apply -f pipeline.yaml

// triggers
$ kubectl apply -f triggers.yaml

// svc,pods の確認
// eventlistenerの service は NodePort に変更している
$ kubectl -n getting-started get svc,pods
NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/el-getting-started-listener   NodePort    10.96.36.181   <none>        8080:30903/TCP   31m
service/event-display                 ClusterIP   10.96.220.61   <none>        8080/TCP         98m

NAME                                               READY   STATUS    RESTARTS   AGE
pod/el-getting-started-listener-7b9fd6dd59-9jddh   1/1     Running   0          4m26s
pod/event-display                                  1/1     Running   0          98m
```

自分の環境では kind の extraPortMappingsで、 今は以下のように portのマッピングをしている状態なので
```
57b2beed9d43   kindest/node:v1.19.1   "/usr/local/bin/entr…"   2 hours ago    Up 2 hours    0.0.0.0:3219->30000/tcp     tekton-triggers-worker
2edbb1e732b2   kindest/node:v1.19.1   "/usr/local/bin/entr…"   2 hours ago    Up 2 hours    127.0.0.1:61878->6443/tcp   tekton-triggers-control-plane
c647aca46dc7   kindest/node:v1.19.1   "/usr/local/bin/entr…"   2 hours ago    Up 2 hours    0.0.0.0:7274->30001/tcp     tekton-triggers-worker2
```

serviceをeditして NodePortを `30000`に書き換えて、localの `3219`からアクセスできるようにしてみる
```
$ kubectl -n getting-started edit service el-getting-started-listener

$ curl localhost:3219
{"eventListener":"getting-started-listener","namespace":"getting-started","errorMessage":"Invalid event body format format: unexpected end of JSON input"}
```
今回は service経由でアクセスするために上記のようなことをしたが、kubectlの port-forward でPodに直接アクセスするということもできる
```
$ kubectl port-forward $(kubectl get pod -o=name -l eventlistener=listener) 8080
```

github から イベントのペイロードがきたことを擬似的に実現する
```
$ curl -X POST http://localhost:3219 \
-H 'Content-Type: application/json' \
-d '{
    "repository": {
        "full_name": "ginoh/sample-hello-world"
    },
    "head_commit": {
        "id": "d42345f9e4b413673526a2e8637241245a719ee0"
    }
}'

$ kubectl tkn -n getting-started pr list
$ kubectl tkn -n getting-started tr list
```

$ kubectl -n getting-started port-forward pod/tekton-triggers-built-me 8080

$ curl localhost:8080                                        
"Hello, Tekton Triggers!"