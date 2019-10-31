# ConnectToS3WebsiteDemo

## 概要

- S3でStatic website hostingした場合に、各インスタンスはどのような挙動をするのか検証するための環境を構築する
- Bucketポリシーは、**SourceCidrIp**で指定したCIDRからの接続、および当該VPCからの接続を許可している

## 構成
## デプロイ

**Properties**

|Name|Type|Default|Description|
|--|--|--|--|
|AMIId|String|ami-0ff21806645c5e492|インスタンスのマシンイメージID|
|InstanceType|String|t2.micro|インスタンスタイプ|
|KeyName|AWS::EC2::KeyPair::KeyName|*require*|キーペア名|
|SourceCidrIp|String|0.0.0.0/0|デモサイトへの接続許可するCIDR|

```sh
aws cloudformation create-stack \
    --stack-name vpc-endpoint-demo \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=KeyName,ParameterValue=KEY_NAME \
    --template-body file://template.yaml
```

※KeyNameは自身の環境で作成済みのKeyNameを指定（未作成の場合は要作成）

## デモサイトデプロイ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name vpc-endpoint-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
echo ${WEBSITE_BUCKET}
  # (e.g.)
  # vpc-endpoint-demo-websitebucket-7q5fuawbt02h

aws s3 cp --content-type text/html index.html s3://${WEBSITE_BUCKET}
  # (e.g.)
  # upload: ./index.html to s3://vpc-endpoint-demo-websitebucket-7q5fuawbt02h/index.html
```

## ログイン

**Public Instance**へログイン

- SSMのManaged Instancesから、インスタンスを選択し、**Start Session** でログイン

**Private Instance**へログイン

- Public Instanceにログインした状態で、キーをコピペしsshで接続

```sh
cat <<EOT > key.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAxtTC+BRf2xeuiuH4vEBzl1cfqTSzJXhquzY2LYWNNpX0kKzmYW+fSc4vgzkm
...
AMjGg3t/Ml8Cw8uarpXwJLJnsNooX65OMfeEkFYlVl3yiCty1xZMxi8bdS6ZH+B9PphRLw==
-----END RSA PRIVATE KEY-----
EOT

chmod 600 key.pem

ssh -i key.pem ec2-user@PRIVATE_INSTANCE_PRIVATE_IP
```

## 結果

**デモサイトへの接続許可したCIDR**からのアクセス

![](https://github.com/ot-nemoto/ConnectToS3WebsiteDemo/blob/images/HelloConnectToS3WebsiteDemo.png)

**PublicSubnet**のインスタンスからのアクセス

```sh
curl --connect-timeout 5 http://vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com/
  # <html>
  # <head><title>403 Forbidden</title></head>
  # <body>
  # <h1>403 Forbidden</h1>
  # <ul>
  # <li>Code: AccessDenied</li>
  # <li>Message: Access Denied</li>
  # <li>RequestId: C89772684B5AE2D9</li>
  # <li>HostId: 6nGLklWcP1b6XsaoaPEjQOjTnEc4q8LEFJP6diLTeLLxfDReXGU8aiv/bp6l4jpWLL8dL4/WBOY=</li>
  # </ul>
  # <hr/>
  # </body>
  # </html>
```

**PrivateSubnet1**のインスタンスからのアクセス

```sh
curl --connect-timeout 5 http://vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com/
  # <h1>Hello VpcEndpointDemo</h1>
```

**PrivateSubnet2**のインスタンスからのアクセス

```sh
curl --connect-timeout 5 http://vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com/
  # curl: (28) Connection timed out after 5001 milliseconds
```

**PrivateSubnet3**のインスタンスからのアクセス

```sh
curl --connect-timeout 5 http://vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com/
  # <html>
  # <head><title>403 Forbidden</title></head>
  # <body>
  # <h1>403 Forbidden</h1>
  # <ul>
  # <li>Code: AccessDenied</li>
  # <li>Message: Access Denied</li>
  # <li>RequestId: D9C1604E310E423F</li>
  # <li>HostId: VZRhKyKCO//IpMAm5BmggwzC3zD2UziLugN0V1/ljKfHuWmbA13NvWfP/naHcwTOJ/4Ul2/tOK0=</li>
  # </ul>
  # <hr/>
  # </body>
  # </html>
```

## ログを確認

ログをダウンロード（ログがS3のBucketに出力されるまで時間がかかるぽいので注意）

```sh
LOGGING_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name vpc-endpoint-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteLoggingBucket`].OutputValue' \
    --output text)
echo ${LOGGING_BUCKET}
  # (e.g.) vpc-endpoint-demo-websiteloggingbucket-1gcf53tbvqv6n

mkdir logs
aws s3 cp s3://${LOGGING_BUCKET} logs --recursive
```

ログを確認（index.htmlを参照しているログのみ抽出）

```sh
cat logs/* | grep index.html
  # 8e90407bc27c842be769d179392a2c331f4dfd979e258d3e84fc75487a5f0586 vpc-endpoint-demo-websitebucket-ldj25c96umm1 [31/Oct/2019:02:54:43 +0000] xx.xx.xx.xx - 98A3184CA55C8A08 WEBSITE.GET.OBJECT index.html "GET / HTTP/1.1" 200 - 38 38 41 40 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.70 Safari/537.36" - iuV2wfA6Pfxi3pXckYDjG3W5pLAGBLcRs13k3OcBFzWlxYsVoG+9c1R+VCzg7lGNBm8Q71e+oO8= - - - vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com -
  # 8e90407bc27c842be769d179392a2c331f4dfd979e258d3e84fc75487a5f0586 vpc-endpoint-demo-websitebucket-ldj25c96umm1 [31/Oct/2019:02:56:21 +0000] 52.198.48.49 - 72B6E8BCF735F705 WEBSITE.GET.OBJECT index.html "GET / HTTP/1.1" 403 AccessDenied 303 - 23 - "-" "curl/7.61.1" - Jf2w98/R9W3TJHldf/+u2OYOoco/Jp8ssO8mVjQG9YpTJHPmUBFSmdCPX9vUZiaEHvh/iIP0crw= - - - vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com -
  # 8e90407bc27c842be769d179392a2c331f4dfd979e258d3e84fc75487a5f0586 vpc-endpoint-demo-websitebucket-ldj25c96umm1 [31/Oct/2019:02:56:38 +0000] 10.38.1.122 - F9D92E75D177AE8D WEBSITE.GET.OBJECT index.html "GET / HTTP/1.1" 200 - 38 38 42 42 "-" "curl/7.61.1" - 7GUGYssXuKGrzSndTeQN/OPyqdUjDo1IZ2xEKxtkm8EOH9RSU0jETi9U/hvMCo/5fSR6AVVZPdQ= - - - vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com -
  # 8e90407bc27c842be769d179392a2c331f4dfd979e258d3e84fc75487a5f0586 vpc-endpoint-demo-websitebucket-ldj25c96umm1 [31/Oct/2019:02:57:11 +0000] 18.183.0.212 - 61D3B12411C2EA88 WEBSITE.GET.OBJECT index.html "GET / HTTP/1.1" 403 AccessDenied 303 - 27 - "-" "curl/7.61.1" - BWDuiznQEfeWR4Ui30J64rrzozls4gzDJ2kmOPllWnkxOZOEirPJIZagkIuaBvyfyc97mdo23nc= - - - vpc-endpoint-demo-websitebucket-ldj25c96umm1.s3-website-ap-northeast-1.amazonaws.com -
```

**xx.xx.xx.xx**

- デモサイトへの接続許可したCIDRからのアクセス
- バケットポリシーで許可しているので**200**

**52.198.48.49**

- PublicSubnetのインスタンスからのアクセス
- IGWを介したアクセスは、インスタンスのパブリックIPアドレスでアクセスしている
- バケットポリシーで許可していないので**403**

**10.38.1.122**

- PrivateSubnet1のインスタンスからのアクセス
- VPC Endpointからのアクセスは、インスタンスのプライベートIPアドレスでアクセスしている
- バケットポリシーで許可しているので**200**

**18.183.0.212**

- PrivateSubnet3のインスタンスからのアクセス
- NAT Gatewayからのアクセスは、NAT GatewayのEIPでアクセスしている
- バケットポリシーで許可していないので**403**
