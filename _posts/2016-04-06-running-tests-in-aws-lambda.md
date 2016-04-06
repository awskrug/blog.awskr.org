---
layout: post
title: "AWS Lambda 를 이용한 테스트 구현"
translate: 2016-04-06T18:48:06-07:00
author: Edwin Xie, Corey Innis, 번역 - Younjin Jeong, Pivotal Labs 
about_stub: 정윤진 - Pivotal Labs. 
profile_picture: https://blog.pivotal.io/kr/wp-content/uploads/sites/3/2015/10/profile-150x150.jpg
author_site: http://kerberosj.tistory.com/
categories: lambda test s3 iam-role iam-user   
--- 

# 살펴보기 

본 포스팅에서는 [`s3cli`](https://github.com/pivotal-golang/s3cli)라는 S3 및 S3호환 Go_lang 기반 클라이언트의 동작 테스트를 위해 AWS Lambda 를 사용한 예를 소개하고 있습니다. 
이는 [Cloud Foundry](https://github.com/cloudfoundry/) 및 [BOSH](https://github.com/cloudfoundry/bosh)로 알려진 시스템들이 S3나 Swift와 같은 [외부 Blob store][blobstore]에 
다양한 파일 및 데이터를 저장하기 위해 사용되고 있는 도구 입니다.  

아래에 소개되는 테스트의 내용은, AWS access_key_id 및 secret_key_id 없이 IAM Role 을 사용하는 경우 문제없이 S3 버켓에 주어진 권한을 사용해서 접근해야 하는 기능을 테스트하기 위해 구현되었습니다. 
IAM Role은 AWS가 제공하는 서비스간 상호 접근을 위해 IAM을 통해 할당받은 별도의 사용자 키가 없이도 다른 서비스에 접근 권한을 지정할 수 있는 방법입니다. 권한이 지정된 IAM Role 이 활성화된 상태로 구동된 EC2에서 
동작하는 AWS SDK는 인증 정보가 코드에 주어지지 않더라도 내부적으로 임시 인증을 할당 받는 방법을 사용하여 다른 서비스에 접근할 수 있습니다. 
 

# EC2 를 사용한 테스트 

이러한 IAM Role 기능은 AWS 서비스에서만 유효한 것이며, 따라서 종래의 s3cli 에 대한 기능 테스트 방법은 EC2와 S3 버켓을 준비하고 s3cli가 EC2 위에서 문제없이 동작하는지 확인하는 것이었습니다. 
이 테스트를 위해 이전에 우리는 다음과 같은 방법을 사용했었습니다.  

  1. CloudFormation 을 사용하여 VPC, IAM Role, 그리고 CI worker로 부터 SSH를 받을 수 있도록 구성된 EC2 인스턴스를 start 
  1. 생성된 EC2 인스턴스에 s3cli 바이너리를 SCP로 복사하고 실행하여 바이너리가 의존성 문제 없이 동작하는지 확인 
  1. CI Worker는 EC2 인스턴스에 SSH 로 접근하여 테스트를 수행하고 0 이외의 exit 이 발생했는지 확인 

위에 설명된 방법의 문제는 일단 CloudFormation 을 사용한 환경의 구성에 시간이 매우 오래 걸린다는 것과, 테스트를 위해 관련이 없는 
자잘한 리소스들이 함께 생성되어 비용을 낭비한다는 점, 그리고 기본적으로 이 잠깐의 테스트 하나를 위해 사용되는 EC2의 비용이 낮지 않다는 점에 있습니다. 
이를테면 CloudFormation 은 VPC를 생성 수의 제한, EC2 생성 수의 제한등, s3cli의 기능 테스트와는 관계 없는 다른 수많은 원인으로 인해 스택의 생성이 실패할 수 있는 가능성이 있기 때문에, 이를 
추적하기 위한 시간과 비용의 낭비를 초래할 가능성이 있겠습니다.  

# Lambda 를 사용한 테스트 

[AWS Lambda](https://aws.amazon.com/lambda/)는 이런 문제를 해결하기에 매우 적절한 도구입니다. Lambda는 이런 간단한 테스트를 위해 EC2를 구동하기 위해 필요한 전체 네트워크 스택과 CloudFormation을 
사용할 필요를 없앨 수 있습니다. 또한 CloudFormation을 사용해 스택을 구성하고 EC2 인스턴스를 기동하는데 필요한 기나긴 시간을 낭비할 필요도 없습니다. 

한가지 더, s3cli의 기능을 테스트하기에 더 없이 좋았던 것중 하나는 바로 s3cli 도구의 IAM Role 을 사용한 AWS 서비스 연동의 테스트에 10초 이상 필요하지 않다는 것입니다. 즉, 기존 한번 켜면 무조건 1시간의 EC2 비용을 지불하는 대신 
10초의 Lambda 사용료만 지불하면 되므로 매우 저렴한 테스트를 구현할 수 있게 된 것입니다. 

현재 AWS Lambda 를 사용하는데 있어 제약 조건 중 하나는 바로 Node.js, Python, Java 이 세가지만 사용할 수 있다는 것입니다. 또한 별도의 BASH 지원이 없으므로 임의의 패키지를 만들어 업로드 하는 방법을 사용하고, 이 업로드된 패키지를 
읽기 전용 파일 시스템을 통해 접근해서 구동합니다. (물론 Lambda 에서도 /tmp 를 사용하면 파일시스템에 읽기 쓰기 모두 가능합니다.) 

## 배포용 패키지 작성 

위에 설명한 방법을 사용하여 s3cli를 테스트 하고 그 결과를 확인하기 위해 아래의 코드를 참조합니다. 여기 예제에서는 [Ginko](http://onsi.github.io/ginkgo/) 를 사용 했습니다.  

~~~python
import os
import logging
import subprocess

def test_runner_handler(event, context):
    os.environ['S3_CLI_PATH'] = './s3cli'
    os.environ['BUCKET_NAME'] = event['bucket_name']
    os.environ['REGION'] = event['region']
    os.environ['S3_HOST'] = event['s3_host']

    logger = logging.getLogger()
    logger.setLevel(logging.DEBUG)

    try:
        output = subprocess.check_output(['./integration.test', '-ginkgo.focus', 'AWS STANDARD IAM ROLE'],
                                env=os.environ, stderr=subprocess.STDOUT)
        logger.debug("INTEGRATION TEST OUTPUT:")
        logger.debug(output)
    except subprocess.CalledProcessError as e:
        logger.debug("INTEGRATION TEST EXITED WITH STATUS: " + str(e.returncode))
        logger.debug(e.output)
        raise
~~~

위에 사용된 Lambda 함수는 매우 간단합니다. 첫째, event를 통해 필요한 환경 변수를 가져옵니다. 이 event는 Lambda 함수가 호출될때 함께 전달 됩니다. 그리고 테스트에서 발생하는 로그를
확인하기 위해 STDOUT과 STDERR 로거를 구성합니다. 또한 Lambda의 exit 코드를 확보하기 위해서 try 블럭 안에 서브 프로세스 콜을 구성하고 subprocess.CalledProcessError를 통해 로그를 확보합니다. 

s3cli와 테스트를 위한 실행 파일들이 스크립트로서 동일한 디렉토리에서 호출되는 것에 주의 합니다. 

다음으로, 테스트 도구와 s3cli에 64-bit Linux 아키텍처를 사용하도록 컴파일 옵션을 구성합니다.  

~~~bash
git clone https://github.com/pivotal-golang/s3cli
cd s3cli
GOOS=linux GOARCH=amd64 go build s3cli/s3cli
GOOS=linux GOARCH=amd64 ginkgo build src/s3cli/integration
~~~

이제 필요한 파일을 모두 묶어 패키지로 구성합니다. 

~~~bash
zip -j deployment.zip src/s3cli/integration/integration.test s3cli lambda_function.py
~~~

## AWS 환경 준비 

s3cli 테스를 위한 사전 준비는 아래와 같습니다. 

  - 테스트용 S3 버켓 생성
  - [AWS IAM 사용자 생성][create_iam_user] A) Lambda 함수 실행 권한 B) CloudWatch 읽기/쓰기 권한 [다음 링크의 정책을 적용][policies] 
  - [AWS IAM Service role 생성][create_iam_role] AWS Lambda 서비스가 CloudWatch와 S3버켓에 다음의 권한을 허용하도록 구성 
    s3::GetObject*, s3::PubObject*, s3::List*, s3::DeleteObject*. 이 Role을 생성할때 "AWS Lambda"를 선택하여 Lambda가
    CloudWatch와 S3에 쓰기 접근을 할 수 있도록 구성. 이 과정을 거쳐 신규로 생성된 IAM Role 의 ARN 정보를 복사해 두는 것을 잊지 않고 
    생성될 Lambda 함수에 IAM_ROLE_ARN 환경 변수로 전달 해야 함. 
    
위의 설정 구성은 [CloudFormation 템플릿][cloudformation]을 사용하여 간편하게 구성이 가능합니다. 
아래에 소개된 단계는 [AWS 명령줄 인터페이스](https://aws.amazon.com/cli/) 를 설정하고 사용하는 방법에 대해서 소개합니다. 

~~~bash
# example session
$ aws configure

AWS Access Key ID [****************AAAA]:
AWS Secret Access Key [****************AAAA]:
Default region name [us-east-1]:
Default output format [json]:
~~~

## Lambda 함수 실행 

이제 우리는 압축된 배포 패키지를 준비했으므로 아래의 AWS 명령을 사용하여 Lambda 함수를 생성하고 
호출할 수 있습니다. 

~~~bash
aws lambda create-function \
  --region us-east-1 \
  --function-name MY_LAMBDA_FUNCTION \
  --zip-file fileb://deployment.zip \
  --role ${IAM_ROLE_ARN} \
  --timeout 300 \
  --handler lambda_function.test_runner_handler \
  --runtime python2.7

aws lambda invoke \
  --invocation-type RequestResponse \
  --function-name MY_LAMBDA_FUNCTION \
  --region us-east-1 \
  --log-type Tail \
  --payload '{"region": "us-east-1", "bucket_name": "YOUR_BUCKET_NAME", "s3_host": "s3.amazonaws.com"}' \
  lambda.log
~~~

중요: 위의 내용에 fileb:// 는 잘못 기재된 것이 아님 [fileb][fileb] 

aws lambda invoke 명령의 실행 결과는 아래와 같이 JSON 으로 표시 될 것입니다. 

~~~json
{
  "LogResult": "<BASE64 ENCODED OUTPUT TRUNCATED TO LAST 4KB>",
  "FunctionError": "(Unhandled|Handled)",
  "StatusCode": 200
}
~~~

첫째로, aws lambda invoke 명령 자체는 실행한 Lambda 함수가 에러 없이 종료 되었는지 아닌지와 
관계 없이 동작합니다. 이는 출력되는 FunctionError 키를 통해 해당 Lambda 함수의 실행에 에러가 있었는지를 
판단할 수 있으며, 이러한 output에 jq 와 같은 도구를 사용하여 간단히 테스트가 성공인지 실패인지 알수 있겠습니다. 

추가적으로, 이 테스트는 기본적으로 4kb 이상의 output 을 생성할 것이며, Base64로 인코딩 된 LogResult를 디코드 
하더라도 일부 로그가 삭제되는 현상이 있을 수 있습니다. 

## 완전한 로그를 가져오기 

다행이 우리에겐 logging.Logger 로부터 생성되는 console output 은 AWS CloudWatch Logs 로그로 보내도록 구성했습니다. 
물론, 이를 위해서는 IAM Role에 Labmda 함수가 로그 그룹, 로그 스트림, 로그 이벤트에 대한 생성 권한이 주어져야 합니다. 

CloudWatch 에서 로그를 가져오기 위해서는 Lambda 함수 이름을 바탕으로 로그 그룹의 이름을 사용해야 합니다. 이후 아래의 
명령어를 사용하여 누락 없는 전체 로그를 가져올 수 있습니다. 

~~~bash
LOG_GROUP_NAME="/aws/lambda/MY_LAMBDA_FUNCTION"
LOG_STREAM_NAME=$(
  aws logs describe-log-streams --log-group-name=${LOG_GROUP_NAME} \
  | jq -r ".logStreams[0].logStreamName"
)
LOG_STREAM_EVENTS_JSON=$(
  aws logs get-log-events \
    --log-group-name=${LOG_GROUP_NAME} \
    --log-stream-name=${LOG_STREAM_NAME}
)

echo ${LOG_STREAM_EVENTS_JSON} | jq -r ".events | map(.message) | .[]"
~~~

## 정리 

이와 같은 방법을 사용하면 고가의 EC2 대신 간단한 테스트에 대해 Lambda 를 사용하여 비용과 시간, 그리고 EC2와 VPC 네트워크를 
생성하는데 있어 발생할 수 있는 실패요인을 줄일 수 있습니다. 여기서는 s3cli 라는 도구의 IAM Role 동작에 대한 테스트를 구성하는 
방법에 대해 기술하였으나, 이와 유사한 목적에 어디든 응용하실 수 있겠습니다. 

원문: http://engineering.pivotal.io/post/running-tests-in-aws-lambda/ 

[blobstore]:       http://bosh.io/docs/director-configure-blobstore.html
[iam_role]:        http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
[create_iam_user]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html
[create_iam_role]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html#roles-creatingrole-service-console
[policies]:        http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-using.html#attach-managed-policy-console
[cloudformation]:  https://github.com/pivotal-golang/s3cli/blob/4e8430386451979c65b38b234f34be4ea695c14d/ci/assets/cloudformation-s3cli-iam.template.json
[fileb]:           http://docs.aws.amazon.com/cli/latest/userguide/cli-using-param.html#cli-using-param-file-binary

