### Tektonとは

一言で言うと、Kubernetes ベースの CI/CDのシステムを作成するための OSS フレームワーク

いくつかのプロジェクトがあるが、その中でもベースとなるプロジェクトである Pipelines は  
CI/CD のパイプラインを k8s のリソースとして定義できる機能を提供している

### Tektonのプロジェクト

Github の tektoncd OrganizationのTopレベルのリポジトリのものが Tektonのプロジェクト  
experimental リポジトリには Incubating プロジェクトがある

数あるプロジェクトの中で以下の4つは Core プロジェクトと呼ばれる
* Pipelines・・・CI/CD ワークフローの基礎となるビルディングブロック
* Triggers ・・・ CI/CD ワークフローの Eventトリガー
* CLI ・・・ CI/CD ワークフロー管理の CLIインタフェース
* Dashboard・・・WebUI

### Tekton Pipelines の コンセプト

(TBD)


https://tekton.dev/docs/concepts/

* step, task, pipeline の関係
* Resource
* Pipelineのインスタンス化

### Tutorial

#### 環境準備

参考：[Installing Tekton Pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/install.md)

kind でクラスタを作成する  
また、piplineでビルドしたイメージをpushできるように、[Local registry](https://kind.sigs.k8s.io/docs/user/local-registry/) のドキュメントのスクリプトを参考に作成する

```
$ ../../k8s/kind/kind-with-registry.sh tekton-pipelines
```

```
$ kubectl version --short
Client Version: v1.19.3
Server Version: v1.19.1
```

Tekton インストール

参考：[ installed and configured](https://github.com/tektoncd/pipeline/blob/master/docs/install.md)

```
$ kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

// コンポーネントが起動しているのを確認
$ kubectl -n tekton-pipelines get pods

// Tekton CLIのインストール
// macの場合
$ brew tap tektoncd/tools
$ brew install tektoncd/tools/tektoncd-cli

// kubectl plugin も設定されている
$  which kubectl-tkn
/usr/local/bin/kubectl-tkn
```

PipelineResource を task間で共有したい場合などは storage の  
設定をするが今回は何もしないのでそのまま


以降、[Tekton Pipelines Tutorial](https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md) を試す

#### Taskの実行
```
$ cd pipelines/tutorial/task

// Hello Worldするだけの Task
$ kubectl apply -f ./hello-world/echo-hello-world.yaml
$ kubectl tkn task describe echo-hello-world

$ kubectl apply -f ./hello-world/echo-hello-world-run.yaml
// tkn CLI (kubectl plugin を利用する場合)、taskrun リソース名は自動生成される
// kubectl tkn task start echo-hello-world

$ kubectl tkn taskrun describe echo-hello-world-run
$ kubectl tkn taskrun logs echo-hello-world-run
[echo] Hello World
```

#### Task で input/output を利用

// イメージを保存するための registryはローカル環境に構築済

```
$ kubectl apply -f \
    -f input-output/resources.yaml \
    -f input-output/build-docker-image-from-git-source.yaml \
    -f input-output/build-docker-image-from-git-source-run.yaml
$ kubectl get tekton-pipelines

$ kubectl tkn taskrun describe build-docker-image-from-git-source-run
$ kubectl tkn taskrun logs -f build-docker-image-from-git-source-run

$ kubectl tkn taskrun list
NAME                                     STARTED         DURATION     STATUS
build-docker-image-from-git-source-run   3 minutes ago   2 minutes    Succeeded
echo-hello-world-run-xbgt8               1 day ago       19 seconds   Succeeded

// ローカルで実行
$ curl localhost:5000/v2/_catalog
{"repositories":["ginoh/hello-world"]}
```

#### Pipeline の作成・実行
```
$ cd ../pipeline

$ kubectl create clusterrole tutorial-role \
    --verb=* \
    --resource=deployments,deployments.apps

$ kubectl create clusterrolebinding tutorial-binding \
    --clusterrole=tutorial-role \
    --serviceaccount=default:tutorial-service
$ kubectl create serviceaccount tutorial-service

$ kubectl apply -f deploy-using-kubectl.yaml -f tutorial-pipeline.yaml 
$ kubectl apply -f tutorial-pipeline-run-1.yaml
$ kubectl tkn pipelinerun describe tutorial-pipeline-run-1
$ kubectl tkn pipelinerun logs -f tutorial-pipeline-run-1
```


### 参考
https://github.com/tektoncd/pipeline/blob/master/docs/install.md
https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md