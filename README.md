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
