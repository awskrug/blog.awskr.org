---
layout: post
title:  "KMS로 인증 정보를 암호화 하고 Lambda 실행시에 복호화 하기"
date:   2017-01-02 20:15:59 +0900
author: 클래스메소드 주식회사(クラスメソッド株式会社)
about_stub: 정도현
profile_picture: http://cdn.dev.classmethod.jp/wp-content/uploads/2015/08/kms.png
author_site: http://moreagile.net/
categories: Lambda KMS
---

> 이 글은 일본 클래스메소드사의 허락하에 한글화 하였습니다. 한글화를 허락해 주신 클래스메소드사에 감사의 말씀을 전합니다.
[원문: KMSで認証情報を暗号化しLambda実行時に復号化する](http://dev.classmethod.jp/cloud/decrypt-sensitive-data-with-kms-on-lambda-invocation/)

# KMS로 인증 정보를 암호화 하고 Lambda 실행시에 복호화 하기

데이터베이스나 API서버와 같은 외부 시스템과 연결하는 경우 종종 인증 정보를 필요로 하게 됩니다.

이번에는 [AWS Key Management Service](https://aws.amazon.com/kms/)(이하 KMS)의 공통 키 암호방식을 사용하여 암호화 된 인증 정보를 AWS Lambda 함수의 코드에 포함 된 함수 호출시 인증 정보를 해독하는 방법을 소개합니다.

![decrypt-sensitive-data-with-kms-on-lambda-invocation](</images/2017-01-02/decrypt-sensitive-data-with-kms-on-lambda-invocation.png>)

기본적인 아이디어는 다음 블로그에 쓰여져 있으며, 이 포스팅에서는 KMS를 사용한 암호화 작업만을 추려내었습니다.

http://ijin.github.io/blog/2015/08/06/github-to-lambda-to-slack/

KMS는 마스터 키를 사용한 암호화와 복호화 처리를 API로 분리해 내었기 때문에 이를 사용하여 인증 정보를 암호화 합니다.

# KMS와 Lambda의 연계

다음의 순서로 동작을 확인합니다.

1. AWS KMS 마스터 키 생성
1. AWS CLI로 마스터 키를 사용하여 암호화와 복호화 처리를 확인
1. AWS Lambda에서 암호화 된 데이터(ciphertext)를 복호화 하여 출력
1. AWS Lambda 함수에 KMS:Decrypt 권한을 부여
1. AWS Lambda 함수를 실행하고 복호화 되어 있는지를 확인


## KMS 마스터 키 생성

우선 Lambda 함수에서 사용할 마스터 키를 만듭니다.

관리 콘솔에 로그인한 후 IAM에서 Encryption Keys를 선택하면 KMS관리 화면으로 이동할 수 있습니다. KMS라는 이름은 메뉴상에서는 존재하지 않으므로 주의하시기 바랍니다.

![kms_management_console](</images/2017-01-02/kms_management_console-640x335.png>)

KMS 키는 리전마다 개별적으로 존재합ㄴ디ㅏ. Filter의 풀다운 메뉴에서 키를 생성하는 지역을 선택하고 "Create Key"를 클릭하여 마스터 키 생성화면으로 이동합니다.

Key Administrative Permissions 와 Key Permissions 에는 평소 사용하는 시스템 관리자를 선택합니다.

## AWS CLI에서 암호화및 복호화 처리 확인

프로그램을 작성하기 전에 CLI에서 암호화 및 복호화 작업을 확인합니다.

마스터키의 일람을 확인합니다.

```bash
$ aws kms list-keys
{
    "Keys": [
        {
            "KeyArn": "arn:aws:kms:ap-northeast-1:123456789012:key/xxx-yyy-zzz",
            "KeyId": "xxx-yyy-zzz"
        }
    ],
    "Truncated": false
}
```

마스터키의 ARN을 환경변수에 설정합니다. 키를 지정하는 방법에는 KeyId 또는 별칭(Alias)의 두가지 방법이 있습니다.

```bash
$ export KEYID=arn:aws:kms:ap-northeast-1:123456789012:key/xxx-yyy-zzz # KeyId
$ export KEYID=arn:aws:kms:ap-northeast-1:123456789012:alias/lambda # Alias
```

## 마스터키를 사용한 복호화
마서터키로 암호화를 하기위해서는 `Encrypt` API를 사용합니다.

```bash
$ aws kms encrypt --key-id $KEYID --plaintext 'hello, world!'
{
    "KeyId": "arn:aws:kms:ap-northeast-1:123456789012:key/xxx-yyy-zzz",
    "CiphertextBlob": "CiAUkK3nep3+LDfCjRPA5NDnd5NEXv5BWWjweqEySvaTLBKUAQEBAgB4FJCt53qd/iw3wo0TwOTQ53eTRF7+QVlo8HqhMkr2kywAAABrMGkGCSqGSIb3DQEHBqBcMFoCAQAwVQYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAxHrDshpbxSGRgMAXECARCAKHfUd1sJjxRX/7tq7twil6vaXjtPZsnr9AURI1gjR+RPL4WlQTvNDjE="
}
```

레스폰스의 `Plaintext`는 복호화한 데이터(즉 plaintext)를 base64로 엔코드한 것 입니다.

base64로 디코드하면 plaintext가 됩니다.

```bash
$ aws kms decrypt --ciphertext-blob fileb://encrypted --query Plaintext --output text | base64 --decode
hello, world!
```

## Lambda 함수화

KMS로 암호화 한 후 base64로 엔코딩한 문자열을 람다 함수내에 적어넣어 Lambda함수 실행시에 복호화해 보겠습니다.

보통은 암호화한 유저데이터를 사용하여 무언가 처리를 하지만 이번에는 복호화한 문자열을 그대로 레스폰스로서 반환한ㅂ니다.

Lambda함수는 파이썬으로 구현하였습니다.

```python
import base64
import boto3
 
# base64 encoded ciphertext
ciphertext_blob_encoded = 'CiAUkK3nep3+LDfCjRPA5NDnd5NEXv5BWWjweqEySvaTLBKUAQEBAgB4FJCt53qd/iw3wo0TwOTQ53eTRF7+QVlo8HqhMkr2kywAAABrMGkGCSqGSIb3DQEHBqBcMFoCAQAwVQYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAxHrDshpbxSGRgMAXECARCAKHfUd1sJjxRX/7tq7twil6vaXjtPZsnr9AURI1gjR+RPL4WlQTvNDjE='
 
# ciphertext
ciphertext_blob = base64.b64decode(ciphertext_blob_encoded)
 
def lambda_handler(event, context):
    kms = boto3.client('kms')
    dec = kms.decrypt(CiphertextBlob = ciphertext_blob)
    return dec['Plaintext'] #plaintext
```

변수 ciphertext_blob_encoded 에는 ciphertext를 base64로 엔코드한 문자열을 설정했습니다. 위에서 CLI로 암호화한 문자열을 적어줍니다.

## KMS 사용권한을 Lambda 함수에게 부여한다

Lambda함수에 설정한 역할(Role)에 마스터키를 사용한 복호화 권한을 추가합니다. 역할에 인라인 정책(Inline policy)을 추가합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1448696327000",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt"
            ],
            "Resource": [
                "arn:aws:kms:ap-northeast-1:123456789012:key/xxx-yyy-zzz"
            ]
        }
    ]
}

```

Resource에는 KMS 마스터키의 arn을 설정합니다.

## Lambda 함수를 실행한다

```bash
$ aws lambda invoke \
        --function-name kms_demo \
        --payload '' \
        response.txt
{
    "StatusCode": 200
}
 
$ cat response.txt
"hello, world!"
```

무사히 복호화 되었군요.

# 마무리

민감한 정보는 KMS로 암호화 함으로서 프로그램의 소스코드 내에 안심하고 적어넣을 수 있습니다. 소스코드와 KMS의 Decrypt API를 실행권한을 모두 획득하지 않는 이상 원래 정보를 손에 넣는것은 불가능합니다.

KMS는 어떻게 써야 하는지 잘 모르겠다고 하는 이야기가 가끔 들려오지만 사용법이 단순하면서도 보안성을 높이는데 있어서 대딘히 강력한 서비스 입니다. 이번 포스팅에서는 이러한 KMS의 유즈케이스를 소개해 보았습니다.


