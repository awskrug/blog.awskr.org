---
layout: post
title: "AWS ECS로 Swift 웹 어플리케이션 실행하기"
translate: 2017-01-21
author: Chris Barclay
about_stub: 이상록
profile_picture:
author_site:
categories: Amazon ECS
---

#AWS ECS로 Swift 웹 어플리케이션 실행하기

게스트인 Asif Khan님이 AWS ECS로 Swift 웹 어플리케이션 실행하기에 대해 쓴 글입니다.  
  
  
Swift는 안전, 성능 및 소프트웨어 디자인 패턴에 대해 현대적으로 접근해 만들어진 범용 프로그래밍 언어입니다. 
Swift의 목표는 시스템 프로그래밍에서 데스크탑과 모바일 어플리케이션 프로그래밍, 클라우드 서비스까지 확장하는 최고의 언어가 되는 것입니다. 
개발자로서 동일한 스택으로 어플리케이션을 구성할 수 있다는 가능성과 Swift의 장점을 클라이언트 및 서버단에서 활용할 수 있다는 사실에 흥분했습니다. 
코드는 더욱 간결해지고 iOS 환경과 더 밀접하게 통합됩니다.  
  
이 글에서 Swift를 이용해 웹 어플리케이션을 만들고 우분투 리눅스 이미지와 [Amazon ECR](https://aws.amazon.com/ecr)을 사용해 [Amazon ECS](https://aws.amazon.com/ecs)에 배포 하는 방법에 대해 설명합니다. 

##컨테이너 배포 개요

Swift는 우분투에서 사용할 수 있는 버전의 컴파일러를 제공합니다. 컴파일러 외에도 웹 서버, 컨테이너 전략 및 
트레픽 최고치를 자동으로 확대하는 자동 클러스터 관리가 필요합니다.  
  
다음은 서비스를 클라우드에 배포 할 때 내려야 할 결정들입니다:
  
* HTTP server
Swift를 지원하는 HTTP 서버를 선택하세요. Vapor가 가장 쉽습니다. Vapor는 iOS, MACOS 및 우분투에서 동작하는 Swift 3.0의 
type-safe한 웹 프레임워크입니다. Swift 어플리케이션을 배포하는 것은 매우 간단하고 쉽습니다. 
Vapor는 새 Vapor 어플리케이션을 만들고, Xcode 프로젝트를 생성 및 빌드하는 것 뿐만 아니라 어플리케이션을 Heroku 또는 Docker에 배포 하는 것을 도와주는 
CLI를 가지고 있습니다. 다른 Swift 웹 서버도 좋지만 이 글에서는 제가 시작하기에 쉽다고 생각한 Vapor를 사용하겠습니다.  
  
Tip: Vapor 슬렉 그룹에 들어오세요, 매우 도움이 됩니다. 질문이 있다면 원하는 답변을 받을 수 있습니다.  
  
* Container model
Docker는 배포된 어플리케이션을 소프트웨어 [컨테이너](http://aws.amazon.com/containers/) 안에서 빌드, 실행, 테스트, 배포 할 수 있게 해주는 오픈 소스 기술입니다. 
Docker를 사용하면 개발을 위해 표준화된 단위로 소프트웨어를 패키징 할 수 있습니다. 여기에는 소프트웨어 실행에 필요한 모든 것이 포함됩니다: 코드, 런타임, 소프트웨어 툴, 시스템 라이브러리 등 
Docker는 빠르고 안전하고 일관되게, 환경에 구애받지 않고 어플리케이션을 배포 할 수 있게 해줍니다.  
이 글에서는 Docker를 사용하지만 Heroku를 선호하는 경우 Vapor도 Heroku와 호환됩니다.  
  
* 이미지 저장소
Docker를 컨테이너 배포 단위로 선택한 후에는 규모에 맞춰 배포를 자동화 하기 위해 Docker 이미지를 저장소에 저장해야 합니다. 
Amazon ECR은 완벽하게 관리되는 Docker 레지스트리며 AWS IAM 정책을 사용해 저장소를 안전하게 보호할 수 있습니다.  
  
* 클러스터 관리 솔루션
Amazon ECS는 확장성이 뛰어나고 Docker 컨테이너를 지원하는 성능이 좋은 컨테이너 관리 서비스며 Amazon EC2 인스턴스의 관리 클러스터에서 
어플리케이션을 쉽게 실행할 수 있게 해줍니다. ECS가 있으면 자체 클러스터 관리 인프라를 실치하고 운영하고 확장 할 필요가 없습니다.  
  
ECS를 사용하면 자체 클러스터 인프라를 설치, 운영 확장 할 필요가 없기 때문에 컨테이너를 어플리케이션의 한 부분(분산 또는 기타)으로 채택하는것이 쉽습니다. 
ECS에서 Docker를 사용하면 장기간 실행되는 어플리케이션, 서비스 및 배치 프로세스를 유연하게 스케줄링 할 수 있습니다. 
ECS는 어플리케이션 가용성을 유지 관리하며 컨테이너를 확장 할 수 있게 해줍니다.  
  
![1](/images/2017-01-23/swift-ecs.png)  
  
HTTP 서버(Vapor)에서 실행되는 Swift 웹 어플리케이션이 있고, 이 서버는 컨테이너(Docker)에 배포 되었으며, 
이미지는 수평으로 확장할 수 있도록 자동 클러스터 관리(ECS)가 있는 보안 저장소 (ECR)에 저장되어 있습니다다.  
  
##AWS 계정 준비하기
1. AWS 계정을 가지고 있지 않다면 [http://aws.amazon.com](http://aws.amazon.com)에 접속해 화면의 명령을 따라 계정을 생성하세요.  
2. 네비게이션 바에 있는 지역 선택자를 사용해 Swift 웹 어플리케이션을 배포하고 싶은 AWS 지역을 선택하세요.  
3. 원하는 지역에서 보안 키를 만드세요.  
  
##둘러보기
다음은 Swift로 작성된 웹 어플리케이션을 설정하고 ECS에 배포하는데 필요한 단계들입니다:  
  
1. [AWS CloudFormation template](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#cstack=sn%7ESwift-web-app%7Cturl%7Ehttps://s3.amazonaws.com/quickstart-reference/swift/latest/templates/quickstart-swift.template) 인스턴스를 다운로드하고 실행하세요. 
CloudFormation template은 Swift, Vapor, Docker 및 AWS CLI를 설치합니다.  
2. SSH로 인스턴스에 접속하세요.
3. vapor 예제 코드를 다운받으세요.
4. 로컬에서 Vapor 웹 어플리케이션을 테스트하세요.
5. 새로운 API를 포함해서 Vapor 예제 코드를 향상시키세요.
6. 코드를 저장소에 푸쉬하세요.
7. 코드를 도커 이미지로 만드세요.
8. 이미지를 Amazon ECR에 푸쉬하세요.
9. Swift 웹 어플리케이션을 Amazon ECS에 배포하세요.

##세부 단계
1. CloudFormation template을 다운로드하고 EC2 인스턴스를 작동시키세요. CloudFormation은 Swift와 Vapor, Docker 및 git을 설치하고 설정합니다. 
인스턴스를 시작하려면 [여기서](https://www.amazon.com/ap/signin?openid.assoc_handle=aws&openid.return_to=https%3A%2F%2Fsignin.aws.amazon.com%2Foauth%3Fresponse_type%3Dcode%26client_id%3Darn%253Aaws%253Aiam%253A%253A015428540659%253Auser%252Fcloudformation%26redirect_uri%3Dhttps%253A%252F%252Fconsole.aws.amazon.com%252Fcloudformation%252Fhome%253Fregion%253Dus-east-1%2526state%253DhashArgs%252523cstack%25253Dsn%2525257ESwift-web-app%2525257Cturl%2525257Ehttps%25253A%25252F%25252Fs3.amazonaws.com%25252Fquickstart-reference%25252Fswift%25252Flatest%25252Ftemplates%25252Fquickstart-swift.template%2526isauthcode%253Dtrue%26noAuthCookie%3Dtrue&openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&action=&disableCorpSignUp=&clientContext=&marketPlaceId=&poolName=&authCookies=&pageId=aws.ssop&siteState=registered%2Cko_KR&accountStatusPolicy=P1&sso=&openid.pape.preferred_auth_policies=MultifactorPhysical&openid.pape.max_auth_age=120&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&server=%2Fap%2Fsignin%3Fie%3DUTF8&accountPoolAlias=&forceMobileApp=0&language=ko_KR&forceMobileLayout=0) CloudFormation을 실행하세요.
2. SSH로 인스턴스에 접속하세요:  
    ```bash
    ssh –i ec2-user@<bastion host ip>
    ```
  
3. vapor 예제 코드를 다운받으세요. 이 코드는 웹 어플리케이션에 사용중인 예제를 배포하는데 도움이됩니다:  
    ```bash
    git clone https://github.com/awslabs/ecs-swift-sample-app.git
    ```
  
4. Vapor 어플리케이션을 로컬에서 테스트하세요:  
    1. Vapor 프로젝트 빌드:  
        ```bash
        cd ~/ecs-swift-sample-app/example \
        vapor build
        ```
  
    2. Vapor 프로젝트 실행:  
        ```bash
        vapor run serve --port=8080
        ```
      
    3. 서버 실행 확인(새 터미널 창에서):  
        ```bash
        ssh -i ubuntu@ curl localhost:8080
        ```
5. Vapor 코드를 향상시키세요:  
    1. 샘플 어플리케이션에 새 route를 추가하는 가이드를 따라 하세요: [https://Vapor.readme.io/docs/hello-world](https://Vapor.readme.io/docs/hello-world)
    2. 웹 어플리케이션을 로컬에서 테스트하세요:  
        ```bash
        vapor run serve --port=8080
        curl http://localhost/hello.
        ```
  
6. 변경사항을 커밋하고 깃허브 저장소에 푸쉬하세요:  
    ```bash
    git add –all
    git commit –m
    git push
    ```
  
7. 코드를 이용해 새 도커 이미지를 빌드하세요:  
    ```bash
    docker build -t swift-on-ecs \
    --build-arg SWIFT_VERSION=DEVELOPMENT-SNAPSHOT-2016-06-06-a \
    --build-arg REPO_CLONE_URL=https://github.com/awslabs/ecs-swift-sample-app.git/ \
    ~/ ecs-swift-sample-app/example
    ```
  
8. ECR에 업로드: ECR 저장소를 만들고 [Amazon ECR 시작하기](http://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_GetStarted.html)에 있는 단계를 따라 이미지를 푸쉬하세요.  
9. ECS 클러스터를 생성하고 [Amazon ECS 시작하기](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_GetStarted.html)에 있는 단계를 따라 작업을 실행하세요:  
    1. 작업을 생성할 때 ECR 이미지에 전체 레지스트리/저장소:태그 이름을 사용해야 합니다. 예를 들면, aws_account_id.dkr.ecr.us-east-1.amazonaws.com/my-web-app:latest.
    2. 포트 포워딩을 8080으로 설정해야 합니다.
10. 이제 컨테이너로 가서 public IP 주소를 확인하고 결과를 보기 위해 접속해 봅시다.  
    1. 실행 중인 작업을 열고 URL을 확인합니다:  
    ![2](/images/2017-01-23/swift-container-view.png)
    2. 브라우저에서 URL에 접속합니다:  
    ![3](/images/2017-01-23/swift-hello-world.png)  
  
당신의 첫 번째 Swift 웹 어플리케이션이 지금 실행되고 있습니다.  
  
이제, 서비스를 확장하기 위해 [ECS with Auto Scaling](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)를 사용할 수 있고 
서비스를 모니터링 하기 위해 [CloudWatch metrics](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-metrics.html) 와 CloudWatch events를 사용할 수 있습니다.  

##결론
Swift의 장점을 활용하고 싶다면, Amazon ECS와 Amazon ECR을 활용해 Vapor를 웹 컨테이너로 사용하고 Swift 웹 어플리케이션을 대규모로 배포할 수 있으며 
클러스터 관리를 Amazon ECS에 위임할 수 있습니다.  
  
이 글을 넘어서 Swift를 가지고 할 수 있는 많은 흥미로운 것들이 있습니다. Swift를 더 배우시려면 [추가적인 Swift 라이브러리들](https://swiftmodules.com/)을 보시고 [Swift 문서](https://swift.org/documentation/)를 읽어보세요.  
  
질문이나 제안이 있으시면 코멘트로 남겨주세요.