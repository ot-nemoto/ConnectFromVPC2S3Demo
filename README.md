# VpcEndpointDemo

## 概要
## 構成
## デプロイ

**Properties**

|Name|Type|Default|Description|
|--|--|--|--|
|AMIId|String|ami-0ff21806645c5e492|インスタンスのマシンイメージID|
|InstanceType|String|t2.micro|インスタンスタイプ|

```sh
aws cloudformation create-stack \
    --stack-name vpc-endpoint-demo \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=KeyName,ParameterValue=KEY_NAME \
    --template-body file://template.yaml
```

※KeyNameは自身の環境で作成済みのKeyNameを指定（未作成の場合は要作成）

## 検証

Public Instanceへログイン

- SSMのManaged Instancesから、インスタンスを選択し、**Session Start** でログイン

Private Instanceへログイン

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
