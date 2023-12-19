# microservices-demoを一部Rustで置き換え

## 概要

以下マイクロサービスのデモアプリケーションを一部Rustで置き換えました。

<https://github.com/GoogleCloudPlatform/microservices-demo>

## EKSへのデプロイ手順

### gRPCサーバーの実装について

詳細は[src](./src/server.rs)を参照ください。 
以下でgRPCサーバーを起動できます。  

```sh
cargo run --bin server
```

以下でgRPCクライアントを起動して動作確認できます。  

```sh
cargo run --bin server
```

### Dockerイメージの作成 & ECRへpush

今回は東京リージョンとしています。
AWSコンソールからECRにリポジトリを作成します。  
<https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/repository-create.html>  
今回はリポジトリ名をproductcatalogserviceとしています。  
リポジトリ作成後にDockerイメージを作成しECRへpushします。  

```shell
# イメージ作成
docker build -t productcatalogservice .

# タグ付けする
docker tag productcatalogservice:latest <アカウント番号>.dkr.ecr.ap-northeast-1.amazonaws.com/productcatalogservice:latest

# dockerクライアントの認証
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin <アカウント番号>.dkr.ecr.ap-northeast-1.amazonaws.com

# push
docker push <アカウント番号>.dkr.ecr.ap-northeast-1.amazonaws.com/productcatalogservice:latest
```

### CloudShell準備

kubectlを使えるようにします。  
kubectlはkubernetes用のコマンドラインツールです。  
マイクロサービスのデプロイに使用します。  

```shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl

chmod +x ./kubectl

mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin

echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

kubectl version --short --client
```

eksctrlを使えるようにします。  
eksctlはAmazon EKSの環境を簡単に構築できるコマンドラインツールです。  
(CloudFormationを利用して構築しています)  

```shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

### AWS上にEKSクラスター/ノードグループを作成する

以下コマンドでクラスターを作成できます。  

```shell
eksctl create cluster --name <cluster_name> --region <region> --without-nodegroup
```

今回サービスはonlineboutique、リージョンは東京リージョンとします。  

```shell
eksctl create cluster --name onlineboutique --region ap-northeast-1 --without-nodegroup
```

クラスター作成後にノードグループを作成します。  

```shell
eksctl create nodegroup --cluster onlineboutique --nodes 4 --nodes-min 4 --nodes-max 4
```

### マイクロサービスのデプロイ

microservices-demoをクローンした後
ECRへpushしたイメージを使うようマニフェストを修正します  

```shell
# microservices-demoをクローン
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git

# リポジトリへ移動
cd microservices-demo

# 書き換え(★)
vi ./release/kubernetes-manifests.yaml

# デプロイ
kubectl apply -f ./release/kubernetes-manifests.yaml
```

★は以下のように修正してください  

- 修正前: gcr.io/google-samples/microservices-demo/productcatalogservice:<バージョン>
- 修正後: <アカウント番号>.dkr.ecr.ap-northeast-1.amazonaws.com/productcatalogservice:latest

使用後は以下で削除できます。  

```shell
# podを削除
kubectl delete -f ./release/kubernetes-manifests.yaml

# ノードグループを削除する
eksctl delete nodegroup --cluster=onlineboutique --name=<ノードグループ名> --wait

# クラスターを削除する
eksctl delete cluster --name onlineboutique --wait
```

