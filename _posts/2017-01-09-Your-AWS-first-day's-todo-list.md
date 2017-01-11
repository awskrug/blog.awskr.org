---
layout: post
title:  "당신이 AWS 계정을 만들고 가장 먼저 해야 할 일 들과 하지 말아야 할 일 들"
date:   2017-01-02 20:15:59 +0900
author: 정도현
about_stub: 정도현
author_site: http://moreagile.net/
categories: AWS IAM(Identity and Access Management)
---

> 이 글은 [tmknom](https://twitter.com/tmknom)님이 Qiita에 올려주신 '[AWS 계정을 만들고 나서 바로 해야 할 일들 정리(AWSアカウントを取得したら速攻でやっておくべき初期設定まとめ)](http://qiita.com/tmknom/items/303db2d1d928db720888)'에서 영감을 받아 작성되었습니다. 처음 AWS 계정을 만들고 나서 무엇을 해야 할 것인가도 중요하지만 하지 말아야 할 것을 아는 것도 중요하기 때문에 해야 할 것들과 하지말아야 할 것들에 대해서 정리해 보았습니다.

# AWS 계정(루트 사용자) 보호

**루트 사용자란?** 

AWS에 가입할때 맨 처음 만드는 e-mail어드레스와 패스워드로 인증하는 계정을 루트 사용자 라고 합니다. 이 루트 사용자는 그야말로 모든 권한을 지니고 있으며 권한에 대한 제약이 불가능하기 때문에 각별히 신경써서 다루어야 합니다.
AWS의 모범사례에 의하면 AWS 루트 사용자를 만든 이후 가장 먼저 해야 할 일은 IAM 계정을 만들고 권한을 부여할 수 있는 IAM 관리자 계정을 만들고 이후 루트 사용자 대신 그 계정을 사용하는 것 입니다. 루트 사용자는 원칙적으로는 평상시엔 사용하지 않는 것이 좋습니다. 혹시 이 글을 읽으시는 분 중에 아직 루트 사용자를 이용하여 작업을 하시는 분이 계시다면 지금 즉시 관리자 권한을 지닌 IAM유저를 만들고 루트 사용자의 사용을 중지하시기 바랍니다.

## AWS 계정에 대한 액세스 키를 만들지 말 것
AWS는 패스워드 인증 이외에 CLI나 SDK 사용을 위한 액세스 키 인증을 제공합니다. 하지만 루트 사용자에 대해서 액세스 키를  만들지는 마세요. 이미 있다면 삭제하시고 다른 IAM유저의 액세스 키를 사용해서 작업하시기 바랍니다.

## 루트 사용자 암호 변경
루트 사용자의 경우 강력한 암호를 사용하시기 바랍니다. 루트 사용자의 암호 변경 방법은 아래 링크에서 확안하실 수 있습니다.

- [AWS 계정 ("루트") 암호 변경](http://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_credentials_passwords_change-root.html)


## 루트 사용자에 대해 MFA 활성화
보안을 강화하기 위해 패스워드만 사용한 인증보다는 MFA(Multi-Factor Authentication)을 사용한 인증을 설정해 두세요. AWS에서는 [Google Authenticator](https://support.google.com/accounts/answer/1066447?hl=en)와 같은  개방형 TOPT 표준을 지원하는 애플리케이션을 MFA에 사용할 수 있습니다. 루트 사용자의 인증에 MFA를 설정하는 방법은 아래 링크를 참고하시기 바랍니다.

- [AWS에서 멀티 팩터 인증(MFA) 사용하기](http://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_credentials_mfa.html)
- [Multi-Factor Authentication](https://aws.amazon.com/ko/iam/details/mfa/)

## 관리자용 IAM 그룹과 사용자의 작성
관리자용 IAM그룹을 만들고 IAM 사용자를 작성하여 관리자 그룹에 배치시킵니다. 구체적인 방법은 아래 링크를 참고하세요.

- [첫 번째 IAM 관리자 및 그룹 생성](http://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/getting-started_create-admin-group.html)

> IAM유저 중에서도 관리자 권한과 같은 중요한 권한을 지닌 유저에 대해서는 MFA사용을 강제하는것이 좋습니다. 루트 사용자가 아닌 IAM유저에 대해서는 하드웨어 MFA나 가상 MFA이외에 휴대폰 문자 메세지(SMS)에 의한 MFA 사용도 가능합니다.

## 암호 정책 구성
관리자용 계정으로 로그인 한 후에 암호 정책을 설정하여 IAM 사용자의 암호에 대하여 복잡성 요건과 의무적인 교체주기를 지정할 수 있습니다.

- [IAM 사용자의 계정 암호 정책 설정](http://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_credentials_passwords_account-policy.html)

## 보안 상태(Security Status)의 확인

여기까지 진행하셨다면 AWS를 안전하게 사용하기 위한 최소한의 조치가 완료되었습니다.
IAM 서비스 페이지(https://console.aws.amazon.com/iam/home)로 이동하면 Security Status가 다음과 같이 모두 녹색이 되어 있는것을 확인합니다.

![Security Status](/images/2017-01-09/security-status.png)

# 청구정보 관련 설정

IAM에 대한 설정이 끝났다면 다음은 청구정보에 대한 설정을 진행해 봅시다.

## IAM유저의 청구서 정보에 대한 액세스 허가

기본설정에서는 루트 사용자 이외에 유저가 청구정보에 접극하는것이 허가되어 있지 않습니다. 루트 사용자를 가급적 사용하지 않기 위해 IAM유저가 청구서 정보에 접근할 수 있도록 허가해 봅시다.

- [결제 정보 및 도구에 대한 액세스 허용](http://docs.aws.amazon.com/ko_kr/awsaccountbilling/latest/aboutv2/grantaccess.html)

IAM유저에 대해 결제 정보 액세스를 허용했다면 이후 작업은 관리자 권한을 지닌 IAM유저를 사용해서 진행 할 수 있습니다.

# 청구정보와 비용 관리에 대한 설정

## 이메일로 청구서 받기

아래 링크의 지시에 따라 간단한 설정으로 이메일로 청구서를 받아볼 수 있습니다.

- [이메일로 인보이스 받기](http://docs.aws.amazon.com/ko_kr/awsaccountbilling/latest/aboutv2/emailed-invoice.html)

또한 프리티어 한고를 초과할 경우 메일이나 휴대폰 문자(SMS)를 통해 알려주는 결제 경보를 생성할 수도 있습니다.

- [결제 경보 만들기](http://docs.aws.amazon.com/ko_kr/awsaccountbilling/latest/aboutv2/free-tier-alarms.html)

# 그 밖의 설정들

## 이메일 설정

AWS로부터의 이메일을 설정합니다. 마케팅 메일을 받을지는 알아서들...

- [AWS Email Preferences](https://pages.awscloud.com/communication-preferences.html)

![AWS Email Preferences](/images/2017-01-09/aws-email-preferences.png)

## CloudTrail 유효화

AWS API호출에 대한 이력을 추적할 수 있는 CloudTrail이라는 서비스가 있습니다. 추적 내용은 S3 버킷에 저장됩니다.

- [CloudTrail 맨 처음 설정하기(영문)](http://docs.aws.amazon.com/ko_kr/awscloudtrail/latest/userguide/cloudtrail-create-a-trail-using-the-console-first-time.html)

## git-secrets

[git-secrets](https://github.com/awslabs/git-secrets)는 AWS가 만들어 공개하고 있는 툴로서 시크릿 액세스키와 같은 비밀 정보가 git에 커밋 되는 것을 막아줍니다.

GitHub에는 이렇게 잘못 올라온 비밀정보를 노리는 봇들이 수없이 동작하고 있어 실수로 정보가 공개되게 되면 악용되게 될 소지가 큽니다.

- [git-secrets 페이지](https://github.com/awslabs/git-secrets)

## Lambda + CloudWatch Events + KMS를 사용하여 AWS 메니지먼트 콘솔에 대한 무단 액세스를 빠르게 감지하기

제목 그대로 입니다. 사실 내용이 초급자가 진행하기엔 만만치 않지만 도움이 되시는 분 도 있을것 같아 링크를 올려봅니다. 원문은 일본어 기사인데 조만간 번역해서 소개해 보기로 하고 일단은 구글번역 링크를 적어봅니다.
- https://goo.gl/3D7nXX

# 끝으로

이것으로 일단 가장 기본적인 설정이 끝났습니다.

추가적으로는 아래 IAM 모범 사례의 내용을 살펴보고 적용시킴으로서 더욱 강력한 보안을 확보 할 수 있습니다.
몇가지는 이 글과 겹치는 부분도 있지만 상세한 내용을 읽고 확인해 두는것도 좋을듯 합니다.

- [IAM 모범 사례](http://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/best-practices.html)