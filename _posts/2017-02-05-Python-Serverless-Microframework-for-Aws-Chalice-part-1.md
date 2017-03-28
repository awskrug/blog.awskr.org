---
layout: post
title: "빠르게 시작하는 AWS 파이썬 서버리스 마이크로프레임웍 Chalice 튜토리얼 - 파트1"
translate: 2017-02-05
author: 
about_stub: 이상록
profile_picture:
author_site:
categories: Amazon Api Gateway, Amazon Lambda, Serverless
---

# 빠르게 시작하는 AWS 파이썬 서버리스 마이크로프레임웍 Chalice 튜토리얼 - 파트1

chalice로 Amazon API Gateway와 AWS Lambda를 사용하는 어플리케이션을 빠르게 만들고 배포할 수 있습니다. chalice는 다음과 같은 도구를 제공합니다:

- 앱을 만들고 배포하고 관리하기 위한 커맨드라인 툴
- 파이썬에서 뷰를 선언하는 익숙하고 사용하기 쉬운 API
- 자동 IAM 정책 생성기

이 튜토리얼에서는 REST API를 만들고 배포하기 위해 `chalice` 커맨드라인을 사용합니다. 처음에 `chalice`를 설치해야 합니다. virtualenv를 사용하는 것이 좋습니다:

```bash
$ pip install virtualenv
$ virtualenv ~/.virtualenvs/chalice-demo
$ source ~/.virtualenvs/chalice-demo/bin/activate
```

주의: **파이썬 2.7 버전을 사용해야 합니다.** `chalice` CLI 와 `chalice` 파이썬 패키지는 AWS Lambda가 지원하는 버전을 지원합니다. 현재, AWS Lambda는 파이썬 2.7 버전만 지원하기 때문에 이 프로젝트에서도 2.7 버전만 지원합니다. 다음과 같이 실행해서 2.7 버전 파이썬으로 virtualenv를 실행해야 함을 명심하세요.

```bash
# Double check you have python2.7
$ which python2.7
/usr/local/bin/python2.7
$ virtualenv --python $(which python2.7) ~/.virtualenvs/chalice-demo
$ source ~/.virtualenvs/chalice-demo/bin/activate
```

다음, virtualenv에서 `chalice`를 설치하세요:

```bash
$ pip install chalice
```

아래와 같이 실행해서 chalice가 설치되었는지 확인할 수 있습니다:

```bash
$ chalice --help
Usage: chalice [OPTIONS] COMMAND [ARGS]...
...
```



## 자격 증명 설정

어플리케이션을 배포하기 전에 자격 증명을 설정했는지 확인하세요. 이전에 boto3(파이썬 AWS SDK) 또는 AWS CLI를 실행하기 위해 설정한 적이 있다면 이 부분은 건너뛰세요.

이번에 처음으로 AWS 자격 증명을 설정한다면 다음의 단계를 따라서 빠르게 시작할 수 있습니다.

```bash
$ mkdir ~/.aws
$ cat >> ~/.aws/config
[default]
aws_access_key_id=YOUR_ACCESS_KEY_HERE
aws_secret_access_key=YOUR_SECRET_ACCESS_KEY
region=YOUR_REGION (such as us-west-2, us-west-1, etc)
```

자격 증명을 설정하는데 지원되는 모든 방법에 대한 자세한 정보는 [boto3 문서](http://boto3.readthedocs.io/en/latest/guide/configuration.html)를 참조하세요.



## 프로젝트 생성

다음으로 해야 할 일은 `chalice` 명령어를 사용해 새 프로젝트를 생성하는 것입니다.

```bash
$ chalice new-project helloworld
```

이 명령어는 `helloworld` 디렉토리를 생성합니다. 이 디렉토리에서 Cd 명령어를 사용하면 생성된 파일들을 볼 수 있습니다:

```bash
$ cd helloworld
$ ls -la
drwxr-xr-x   .chalice
-rw-r--r--   app.py
-rw-r--r--   requirements.txt
```

지금은 `.chalice` 디렉토리를 무시하세요. 우리가 중점을 둘 중요 파일은 `app.py` 와 `requirements.txt` 입니다.

`app.py` 파일을 살펴봅시다:

```python
from chalice import Chalice

app = Chalice(app_name='helloworld')


@app.route('/')
def index():
    return {'hello': 'world'}
```

`new-project` 명령어는 호출했을 때 JSON body `{"hello": "world"}`를 반환하는,  `/` 라우터를 가진 단일 뷰를 정의하는 샘플 앱을 생성했습니다.



## 배포

이 앱을 배포해 봅시다. `helloworld` 디렉토리에 위치해 있는지 확인하고 `calice deploy`를 실행하세요:

```bash
$ chalice deploy
...
Initiating first time deployment...
https://qxea58oupc.execute-api.us-west-2.amazonaws.com/dev/
```

API Gateway와 Lambda를 사용해 API를 작동합니다:

```bash
$ curl https://qxea58oupc.execute-api.us-west-2.amazonaws.com/dev/
{"hello": "world"}
```

`index()` 함수에서 반환하는 딕셔너리를 바꿔보세요. 그 뒤에 변경사항을 `chalice deploy`로 재배포 할 수 있습니다. 이 튜토리얼의 남은 부분에서는 API를 테스트 하기 위해 `curl` 대신 `httpie`(https://github.com/jkbrzt/httpie)를 사용합니다. `pip install httpie` 를 사용해서 `httpie`를 설치할 수 있고 맥이라면 `brew install httpie`로 설치할 수 있습니다. 위의 깃허브 링크에서 설치 명령어에 대한 자세한 정보를 얻을 수 있습니다. 다음은 `httpie`를 사용해 방금 만든 API의 루트 리소스에 요청을 보내는 예제입니다. 명령어 이름이 `http` 인 것에 주의하세요:

```bash
$ http https://qxea58oupc.execute-api.us-west-2.amazonaws.com/dev/
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 18
Content-Type: application/json
Date: Mon, 30 May 2016 17:55:50 GMT
X-Cache: Miss from cloudfront

{
    "hello": "world"
}
```

추가적으로, 간결성을 위해 API Gateway의 endpoint를 `https://endpoint/dev/`로 줄이겠습니다. API를 배포했을 때 chalice CLI가 보여주는 실제 endpoint로 `https://endpoint/dev/`를 바꿔야 하는 것을 명심하세요. (`https://abcdefg.execute-api.us-west-2.amazonaws.com/dev/` 이렇게 생겼습니다)



## 다음 단계

`chalice`를 사용해 첫 번째 앱을 만들었습니다. 

다음 단원에서는 빠르게 시작하는 Chalice 튜토리얼을 기반으로 URL 매개 변수 캡처, 오류 처리, 고급 라우팅, 현재 요청 메타 데이터 및 자동 정책 생성과 같은 추가 기능을 소개합니다.