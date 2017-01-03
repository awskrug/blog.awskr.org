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

# KMS로 인증 정보를 암호화 하고 Lambda 실행시에 복호화 하기

데이터베이스나 API서버와 같은 외부 시스템과 연결하는 경우 종종 인증 정보를 필요로 하게 됩니다.

이번에는 [AWS Key Management Service](https://aws.amazon.com/kms/)(이하 KMS)의 공통 키 암호방식을 사용하여 암호화 된 인증 정보를 AWS Lambda 함수의 코드에 포함 된 함수 호출시 인증 정보를 해독하는 방법을 소개합니다.

![decrypt-sensitive-data-with-kms-on-lambda-invocation](</images/2017-01-02/decrypt-sensitive-data-with-kms-on-lambda-invocation.png>)

기본적인 아이디어는 다음 블로그에 쓰여져 있으며, KMS를 사용한 암호화 작업만을 추려내었습니다.

http://ijin.github.io/blog/2015/08/06/github-to-lambda-to-slack/

KMS는 마스터 키를 사용한 암호화와 복호화 처리를 API로 분리해 내었기 때문에 이를 사용하여 인증 정보를 암호화 합니다.

## KMS와 Lambda의 연계

다음의 순서로 동작을 확인합니다.

1. AWS KMS 마스터 키 생성
1. AWS CLI로 마스터 키를 사용하여 암호화와 복호화 처리를 확인
1. AWS Lambda에서 암호화 된 데이터(ciphertext)를 복호화 하여 출력
1. AWS Lambda 함수에 KMS:Decrypt 권한을 부여
1. AWS Lambda 함수를 실행하고 복호화 되어 있는지를 확인


### KMS 마스터 키 생성

우선 Lambda 함수에서 사용할 마스터 키를 만듭니다.

관리 콘솔에 로그인한 후 IAM에서 Encryption Keys를 선택하면 KMS관리 화면으로 이동할 수 있습니다. KMS라는 이름은 메뉴상에서는 존재하지 않으므로 주의하시기 바랍니다.

![kms_management_console](</images/2017-01-02/kms_management_console-640x335.png>)

KMS 키는 리전마다 개별적으로 존재합ㄴ디ㅏ. Filter의 풀다운 메뉴에서 키를 생성하는 지역을 선택하고 "Create Key"를 클릭하여 마스터 키 생성화면으로 이동합니다.

Key Administrative Permissions 와 Key Permissions 에는 평소 사용하는 시스템 관리자를 선택합니다.

### AWS CLI에서 암호화및 복호화 처리 확인

프로그램을 작성하기 전에 CLI에서 암호화 및 복호화 작업을 확인합니다.
