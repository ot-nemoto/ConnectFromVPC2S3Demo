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
