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
    --template-body file://template.yaml
```
