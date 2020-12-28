### kubectlを利用してAPIサーバへアクセスする

kubectl を利用して k8sのリソースを管理するためにする設定をメモ  
いわゆるアカウント追加のこと

基本的には kubectl を利用するのは ユーザアカウントだが、サービスアカウントを使うこともできる

### 前提
今回利用する Kubernetes クラスタは kindで作成  

kubernetes の Version は以下
```
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
serverVersion:
  buildDate: "2020-09-14T07:30:52Z"
  compiler: gc
  gitCommit: 206bcadf021e76c27513500ca24182692aabd17e
  gitTreeState: clean
  gitVersion: v1.19.1
  goVersion: go1.15
  major: "1"
  minor: "19"
  platform: linux/amd64
```

TBD

### ユーザアカウントを利用してアクセスする

認証戦略はいくつかあるが、ここではクライアント証明書を利用する  
クラスタの認証局で署名された証明書で表すユーザは認証済みとなる

クライアント証明書の CommonNameがユーザ名として利用される

#### 証明書の作成

kubernetes の証明書APIに CSRをつけてリクエストすると証明書を作ってくれる

鍵を作成する
```
$ openssl genrsa -out adol.key 4096
```
CSR を作成
```
$ openssl req -new -key adol.key -out adol.csr -subj "/CN=adol"
```
subject の `CN` が ユーザ名になる  
`O`でグループを指定できる(e.g. /O=system:masters とするとsystem:masters に所属するようになる)

API を利用して署名する
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: csr-adol
spec:
  signerName: kubernetes.io/kube-apiserver-client
  groups:
    - q
  request: $(cat adol.csr | base64 | tr -d '\n')
  usages:
    - digital signature
    - key encipherment
    - client auth
EOF

certificatesigningrequest.certificates.k8s.io/csr-adol created
```
csr は base64 エンコードする  
groupsに指定されたグループは APIサーバがリソース作成時に追加する  

以前試した時は問題なかったが、今回試したversionでは `spec.signerName` が必須のためエラーとなったので追加している  

signerNameについては以下や `explain`コマンドを参照
https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers

csr 確認と承認
```
// pendingになっている
$ kubectl get csr
NAME       AGE   SIGNERNAME                            REQUESTOR          CONDITION
csr-adol   50s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Pending

// approve
$ kubectl certificate approve csr-adol
certificatesigningrequest.certificates.k8s.io/csr-adol approved

$ kubectl get csr                     
NAME       AGE     SIGNERNAME                            REQUESTOR          CONDITION
csr-adol   2m57s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Approved,Issued
```

証明書の取得
```
$ kubectl get csr csr-adol -o jsonpath='{.status.certificate}' | base64 -d > adol.crt
```

RBACの設定
APIアクセスした際に認証は通ったとしても、権限はRBACにより管理されているため、ユーザに対してRBACを設定する  

RBACについてはドキュメントを参照  
https://kubernetes.io/ja/docs/reference/access-authn-authz/rbac/

今回はひとまず、user-test に対して admin の権限を与える
```
// namespace 作成しておく
$ kubectl create namespace user-test

$ cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: adol-admin
  namespace: user-test
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - name: adol
    kind: User
    apiGroup: rbac.authorization.k8s.io

EOF
```

kubeconfigの準備

```
// masterの確認
$ kubectl cluster-info 
Kubernetes master is running at https://127.0.0.1:64407
KubeDNS is running at https://127.0.0.1:64407/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

// clusterの登録
$ kubectl config set-cluster kind-tips-user --insecure-skip-tls-verify=true --server=https://127.0.0.1:64407

// user の作成
$ kubectl config set-credentials adol --client-certificate=adol.crt --client-key=adol.key --embed-certs=true

// context の作成
$ kubectl config set-context kind-tips-user --cluster=kind-tips-user --user=adol

// context の利用
$ kubectl config use-context kind-tips-user
```

API アクセス

```
$ kubectl -n user-test run nginx --image nginx --port 80

$ kubectl  -n user-test get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          9s

$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "adol" cannot list resource "pods" in API group "" in the namespace "default"
```

Appendix
CertificateSigningRequest リソースは以下のドキュメントにあるように  
自動で削除される
https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#request-signing-process

```
Approved requests: automatically deleted after 1 hour
Denied requests: automatically deleted after 1 hour
Pending requests: automatically deleted after 1 hour
```

### サービスアカウントを利用してアクセスする

ユーザアカウントではなくサービスアカウントをkubeconfigに追加してAPIアクセスすることもできる

検証用に namespace,serviceaccount を作成
```
$ kubectl create namespace user-test-2
$ kubectl -n user-test-2 create serviceaccount my-svc

// secret が自動で生成される
$ kubectl -n user-test-2 get serviceaccounts  
NAME      SECRETS   AGE
default   1         27s
my-svc    1         16s
$ kubectl -n user-test-2 get secrets
NAME                  TYPE                                  DATA   AGE
default-token-5khn7   kubernetes.io/service-account-token   3      78s
my-svc-token-sb6x2    kubernetes.io/service-account-token   3      67s
```

user-test-2 に対して admin の権限を与える
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-svc-admin
  namespace: user-test-2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - name: my-svc
    kind: ServiceAccount
    namespace: user-test-2
EOF
rolebinding.rbac.authorization.k8s.io/my-svc-admin created
```

my-svc 用に生成された tokenを取得する
```
$ kubectl -n user-test-2 get secrets my-svc-token-sb6x2 -o jsonpath='{.data.token}' | base64 -d > my-svc-token
```
kubeconfig に user 追加、context 作成、context切り替え
```
$ kubectl config set-credentials my-svc --token $(cat my-svc-token)
$ kubectl config set-context kind-tips-user-2 --cluster kind-tips --user my-svc
$ kubectl config use-context kind-tips-user-2
```

動作確認
```
$ kubectl -n user-test-2 run nginx --image nginx --port 80
$ kubectl -n user-test-2 get pods
```

### 参考

kubernteesの認証  
https://kubernetes.io/ja/docs/reference/access-authn-authz/authentication/

証明書APIを利用して x509証明書をプロビジョニング  
https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/

kubernetes のPKI証明書について  
https://kubernetes.io/docs/setup/best-practices/certificates/

https://qiita.com/knqyf263/items/aefb0ff139cfb6519e27
