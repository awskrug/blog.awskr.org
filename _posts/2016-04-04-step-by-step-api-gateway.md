---
layout: post
title:  "Step by Step API Gateway + Lambda: 텔레그렘 -> 슬랙 전달 봇 만들기"
date:   2016-04-04 20:15:59 +0900
author: Josuha Kim
about_stub: 김승연 - AWSKRUG 운영진
profile_picture: https://scontent.xx.fbcdn.net/hprofile-xpf1/v/t1.0-1/p320x320/10985258_867345066661949_2320096950103893002_n.jpg?oh=6c0b1a3ddbde2daf2648ddbebe178731&oe=577C3AC6
author_site: http://acuros.pe.kr/
categories: lambda api-gateway chat-ops
---

# Step by Step API Gateway + Lambda: 텔레그렘 -> 슬랙 전달 봇 만들기

안녕하세요! GS SHOP에서 IT Driven Open Communication 이라는 프로젝트를 하고 있는 김승연입니다.

저는 전부터 Labmda에 관심이 많았는데요. 아무리 t1.micro로 돌린다고 해도 Tokyo region 기준으로 한 달에 15달러가 나오는게 너무 아깝더라고요. 실제로 서버가 필요한(Web 서버가 요청을 받고 처리하고 응답해주는) 시간은 한 달에 많아도 7분(90ms * 4300 request) 내외였는데 한 달동안 언제든 요청을 받기 위해서 29일 23시간 53분을 기다리는 요금을 내는게 비효율적이라고 생각했어요.

하지만 Lambda는 EC2와는 달리 가입 후 12개월이 지나도 프리티어를 유지해주고 저 정도의 request면 무료로 이용이 가능했습니다!

![Lambda-Free](</images/2016-04-04/lambda_free.png>)

![Lambda-Bill](</images/2016-04-04/lambda_bill.png>)

그래서 Lambda를 좋아하고, 이번에 회사에서 텔레그렘을 이용하는 팀과 슬랙을 이용하는 팀이 같이 소통해야될 일이 있어서 텔레그렘과 슬랙을 이어주는 봇을 만들어주면 어떨까? 하는 생각이 있어서 Lambda랑 API Gateway로 만들고 과정 일부를 공유하면 재밌겠다 싶어서 이 글을 쓰게 되었습니다 :D

## Lambda 시작하기
먼저 [Lambda console](https://ap-northeast-1.console.aws.amazon.com/lambda/home?region=ap-northeast-1)에 들어가봅시다!
좌측의 Create a Lambda function 클릭
여러 가지 형태의 lambda function을 만들 수 있는데 한 단계씩 올라갈 것이기 때문에 일단 Skip 하겠습니다!
Name에는 TelegramToSlack, 그리고 저는 Python 2.7을 이용할게요.
코드는 Code entry type은 Edit code inline을 선택하고 내용은 간단하게 event를 그대로 return해볼게요.

```python
def lambda_handler(event, context):
  return event
```
Nodejs version: https://gist.github.com/news700/6f1cbe80663a8965a0832ad277b4970e

Lambda function에는 *event*와 *context* 두 개의 파라미터가 넘어오는데, event는 lambda function으로 넘어오는 파라미터들이 Dictionary 형태로 넘어오는 곳이고 context는 함수 이름, 버젼, arn, 메모리 제한, 남은 시간 등을 가져올 수 있는 객체에요..

넘어가서 *handler*와 *role*을 정해줘야하는데요. 기본으로 Handler에는 *lambda_function.lambda_handler*라는 값이 들어있어요. *lambda_function*이라는 모듈의 *lambda_handler*라는 함수를 사용하겠다는 의미인데 inline으로 코드를 편집하게 되면 우리가 편집하는 코드의 파일명이 lambda_function.py이기 때문에 지금 우리가 짠 함수를 실행하겠다는 의미가 돼요. 나중에 프로젝트의 크기가 커지면 파일을 분리하게 되고 AWS Lambda가 실행해야할 함수를 찾아야될 때 이 값을 조정해서 사용하게 돼요.

Role은 이 함수가 어떤 권한을 가지고 실행될 것이냐를 결정해요. 최소한 lambda function을 실행할 수 있는 권한을 포함해야돼요. 그 외에 우리가 짜는 코드가 필요한 권한이 있다면(dynamodb에 데이터를 작성한다던가) 적절히 AWS IAM에서 Role을 생성하여 선택해주면 돼요. 일단 우리는 콤보박스에서 *Create New Role* 안의 *Basic execution role*을 이용할게요. lambda를 실행할 수만 있는 권한이에요. 일단 Role을 생성해준 뒤 해당 Role을 선택하여 사용할 수 있어요. *Basic execution role*을 클릭하면 새로운 창이 열려요(혹시 팝업이 차단돼서 안열릴 수 있으니 확인해보세요!)

IAM Management Console 창이 새로 떴는데 IAM Role에는 *Create a new IAM Role*을 선택해주시고 Role Name은 *lambda_basic_execution*으로 할게요. Allow 하고 돌아오면 Role에 *lambda_basic_execution*이 생성돼 있어요. 혹시 생성돼있지 않다면 새로고침을 해주세요! 다른 내용은 임시저장돼있어요 ㅎㅎ

여기까지 했다면 대략 이런 화면이 됐을거에요

![Create-Lambda](</images/2016-04-04/create_lambda.png>)

Memory와 Timeout은 사용할 수 있는 memory와 최대 실행시간을 정하는건데요. 기본 값으로 두고 넘어갈게요.
Next를 누르고 Create function을 누르면 *Congratulations! Your Lambda function "TelegramToSlack" has been successfully created.* 라는 메시지가 상단에 뜨네요!

그 위에 Test 버튼이 있어요. 이 함수를 웹상에서 테스트를 해볼 수 있는 기능이에요. JSON 형태로 파라미터를 지정할 수 있는데 이 파라미터들은 event변수에 dictionary 형태로 전달돼요. *Hello World* template 그대로 Save and test를 눌러봅시다. 그러면 아랫쪽에 *Execution result: succeeded*를 확인하실 수 있어요! 넘겼던 파라미터들이 잘 return된 모습을 확인할 수 있어요.

이렇게 Lambda의 기본 사용법을 알아봤습니다. 이제 API Gateway와의 연동을 진행해볼게요.

## Lambda의 API Gateway 연동
이제 [API Gateway Console](https://ap-northeast-1.console.aws.amazon.com/apigateway/home?region=ap-northeast-1#/apis)로 들어가보죠.
파란색 *Create API* 버튼을 눌러주세요.
API name에는 뭐 .. 음 .. *Telegram*으로 해볼게요.
이제 Create API를 눌러주세요.

이렇게 API Gateway에서 API를 생성해줬어요. Lambda와 연동해주기 위해 다시 Lambda console로 가서 TelegramToSlack을 선택해주세요.
화면에 보면 *Code* 탭 옆쪽에 *API Endpoints* 라는 탭이 있는 걸 확인할 수 있습니다.

![API-Endpoints-Tab](</images/2016-04-04/API_endpoints_tab.png>)

누르고 들어가서 *Add API endpoint*를 클릭, API endpoint type에 API Gateway를 선택해주세요.
API name을 클릭하면 *Telegram*이 보일거에요. *Telegram*을 선택해주시고 Resource name은 API path를 의미해요. */to-slack* 정도로 해줄게요. Method는 *POST*로 설정해주세요. Telegram Bot이 [POST요청](https://core.telegram.org/bots/api#setwebhook)을 보낼거에요.
Deployment stage는 여러 단계의 stage를 두고 API 버젼을 따로 관리할 수 있는데 기본값 *prod* 그대로 두겠습니다. Security는 *Open*으로 설정해줘야 Telegram Bot이 들어올 수 있어요. 이제 Submit을 클릭합시다!

![Add-Endpoint](</images/2016-04-04/Add_endpoint.png>)

이제 다시 API Gateway console로 가서 Telegram을 선택해보면 */to-slack* 이 추가돼있는 것을 볼 수 있어요. 클릭해보면 Test를 해볼 수 있어요.

![To-Slack-Added](</images/2016-04-04/to_slack_added.png>)

Request body에 *{"foo": "bar"}* 를 입력하고 Test를 눌러보면 response가 잘 오는 것을 확인할 수 있어요.

![To-Slack-Test](</images/2016-04-04/to_slack_test.png>)

## Telegram Bot 세팅하기
Telegram Bot API에는 [setWebHook](https://core.telegram.org/bots/api#setwebhook)이라는 API가 있는데요. 봇이 볼 수 있는 메시지를 설정한 https 주소로 보내주는 기능입니다. 먼저 Telegram에서 봇을 만들게요. https://core.telegram.org/bots 에 따라 텔레그렘에서 botfather 를 검색해 대화를 시작하면 Bot을 생성하고 API Key를 줍니다. 그 뒤에 텔레그램 Group에서 봇이 메시지를 받기 위해서는 privacy 모드를 해제해줘야돼요. botfather에게 /setprivacy 명령을 이용해 해제할 수 있어요.

![Telegram](</images/2016-04-04/telegram.png>)

Lambda console의 *API endpoints* 탭에 가보면 API endpoint URL이 있습니다. Telegram bot의 web hook을 해당 URL로 걸어줄게요.

```sh
$ curl https://api.telegram.org/bot<API TOKEN>/setWebhook?url=<API endpoint URL>
{"ok":true,"result":true,"description":"Webhook was set"}
```

이제 해당 봇이 참여하는 대화의 메시지들은 우리가 만든 API Gateway로 가고 그 endpoint는 Lambda function을 실행시켜서 POST로 넘어온 값을 그대로 돌려줄겁니다!

## Slack에 메시지 보내기
Slack의 *Incoming WebHook*을 이용해서 쉽게 한 채널에 메시지를 보낼 수 있습니다. Incoming WebHook은 [https://slack.com/apps/build](https://slack.com/apps/build) -> Make a Custom Integration -> Incoming WebHooks 에서 만들 수 있어요. 메시지를 보낼 채널을 선택하거나 만듭니다. 저는 *telegram*이라는 채널을 만들었어요. 초록색 *Add Incoming WebHooks integration* 버튼을 누르면 다음 페이지의 아랫쪽에 *Webhook URL*을 확인할 수 있어요.

이제 telegram에서 메시지가 올 때 마다 해당 URL에 "asdf"라는 메시지를 보내봅시다.
Lambda console에 가서 코드를 다음과 같이 수정해주세요.

```python
import httplib
import json
import urllib

def lambda_handler(event, context):
    conn = httplib.HTTPSConnection('hooks.slack.com')
    conn.request(
        'POST',
        '/services/T01234567/B0123456/abcdefghijklmnopqrstuvwxyz',  # slack에서 받은 webhook path
        urllib.urlencode({'payload': json.dumps({'text': 'asdf'})}),
        {'Content-Type': 'application/x-www-form-urlencoded'}
    )
    conn.getresponse()
    conn.close()
```
Nodejs version: https://gist.github.com/news700/37ab1ab1aa769c52e0361ee8a8ae0ed9

Save and test를 눌러 실행해보면 Slack telegram 채널에 asdf 라는 메시지가 온 것을 확인할 수 있어요!
Telegram에서도 우리가 만든 봇 이름을 검색해서 메시지를 전송해보면 봇한테 메시지를 쓸 때 마다 Slack에 asdf가 오는 것을 확인할 수 있어요!

![Slack](</images/2016-04-04/slack.png>)

이제 Telegram에서 보내주는 [포맷](https://core.telegram.org/bots/api#update)에 맞춰 전달받은 텍스트가 있으면 해당 텍스트를 슬랙에 넘겨주는 코드로 바꿔볼게요.

```python
import httplib
import json
import urllib

def lambda_handler(event, context):
    if 'message' not in event:
        return
    if 'text' not in event['message']:
        return
    text = event['message']['text']
    conn = httplib.HTTPSConnection('hooks.slack.com')
    conn.request(
        'POST',
        '/services/T01234567/B0123456/abcdefghijklmnopqrstuvwxyz',  # slack에서 받은 webhook path
        urllib.urlencode({'payload': json.dumps({'text': text})}),
        {'Content-Type': 'application/x-www-form-urlencoded'}
    )
    conn.getresponse()
    conn.close()
```
Nodejs version: https://gist.github.com/news700/ce8154389f7e0bb7a8cb2b6812a73eb0

![Final](</images/2016-04-04/final.png>)

이제 코드를 조금 고치면 새로운 방에 초대될 때마다 새로운 슬랙 채널을 만들 수도 있고 작성자도 표시해주거나 사진을 전송할 수도 있겠네요. 새로운 API를 추가해서 슬랙에서 쓴 메시지를 텔레그램에서도 받아볼 수 있겠네요.
여기서부터는 한 번 원하시는대로 직접 만들어보세요!

고맙습니다!
