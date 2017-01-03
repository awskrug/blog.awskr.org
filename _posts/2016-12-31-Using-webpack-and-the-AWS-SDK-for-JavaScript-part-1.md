---
layout: post
title: "Webpack과 자바스크립트용 AWS SDK 를 사용해 어플리케이션을 만들고 번들하기 - 파트 1"
translate: 2016-12-31
author: Christopher Radek, 번역 - Sangrok Lee 
about_stub: 이상록
profile_picture:
author_site:
categories: JavaScript
--- 

#Webpack과 자바스크립트용 AWS SDK 를 사용해 어플리케이션을 만들고 번들하기 - 파트 1

우리는 [자바스크립트용 AWS SDK](https://aws.amazon.com/documentation/sdk-for-javascript/?) 2.6.1 버전에서 [webpack](https://webpack.github.io/)에 대한 지원을 추가했습니다. 
webpack과 SDK 같은 툴을 사용하면 모듈화된 코드를 작성할 수 있도록 자바스크립트 모듈을 번들할 수 있는 방법을 제공합니다.  
  
이 글은 webpack과 자바스크립트용 AWS SDK를 사용해서 버킷에 있는 [Amazon S3](https://aws.amazon.com/s3/) 객체 목록를 보여주는 간단한 어플리케이션을 만들고 
번들하는 방법을 설명합니다.  
  
##왜 webpack을 사용해야 할까?
webpack과 같은 툴은 어플리케이션 코드를 분석하고 import와 require 구문을 찾아 
어플리케이션이 필요로하는 모든 에셋들을 가지는 번들을 생성합니다.  
  
webpack은 기본적으로 자바스크립트 파일을 다루지만 JSON이나 CSS, 이미지 파일같이 다른 종류도 다루도록 설정할 수 있습니다!
이렇게하면 웹페이지를 통해 에셋을 쉽게 전달하도록 어플리케이션의 에셋을 잘 패키징할 수 있습니다.  
  
webpack을 사용하면 필요한 서비스로 이루어진 번들을 만들고 Node.js에서도 작동하는 번들을 만들수 있습니다.

##선행 조건
이 글을 따라가기 위해서 [node.js](https://nodejs.org/)와 npm이 설치되어 있어야 합니다 (npm은 node.js와 함께 설치됩니다).
이 툴들이 설치되어 있다면 새로운 디렉토리를 만들고 **npm install x** 명령어를 사용해 이 프로젝트에 필요한 의존성 파일들을 다운로드 합니다.
x에는 아래와 같은 단어가 들어갑니다.

* aws-sdk: AWS SDK
* webpack: webpack CLI 와 자바스크립트 모듈
* json-loader: webpack에게 JSON파일 읽는 방법을 알려주는 플러그인

##어플리케이션 세팅하기
프로젝트를 저장하는 디렉토리를 생성하는 것으로 시작합니다. 프로젝트 이름은 aws-webpack으로 짓겠습니다.  
  
어플리케이션은 아래에 있는 세개의 파일을 가집니다:  
  
 * s3.js는 버킷을 문자열로 받아들이는 함수와 콜백 함수를 내보내고 객체 목록을 콜백 함수에 반환합니다.
 * browser.js는 s3.js 모듈을 포함하고 listObjects 함수를 호출하며 결과를 보여줍니다.
 * index.html은 webpack이 생성한 자바스크립트 번들을 참조합니다.
  
아래와 같이 프로젝트의 루트 디렉토리에 이 파일들을 생성합니다:

**s3.js**  
중요: [보안 자격증명 설정](http://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/configuring-the-jssdk.html#Loading_Credentials_in_the_Client_s_Browser)은 스스로 하셔야 합니다.

```JavaScript
// AWS SDK를 포함합니다
var AWS = require('aws-sdk');

// 지역(region)과 자격증명(credentials)을 설정 하세요,
// 이것은 서비스 클라이언트로 직접 이동할 수 있습니다
AWS.config.update({region: 'REGION', credentials: {/* */}});

var s3 = new AWS.S3({apiVersion: '2006-03-01'});

/**
 * 이 함수는 버킷에서 객체 목록을 검색합니다
 * 그 다음, 제공된 콜백을 전달받은 에러 혹은 데이터와 함께 실행합니다
 */
function listObjects(bucket, callback) {
  s3.listObjects({
    Bucket: bucket
  }, callback);
}

// 함수 핸들러를 내보냅니다
module.exports = listObjects;
```

**browser.js**
```JavaScript
// listObjects 함수를 포함합니다
var listObjects = require('./s3');
var bucket = 'BUCKET';
// 특정한 버킷에서 listObjects 를 호출합니다
listObjects(bucket, function(err, data) {
  if (err) {
    alert(err);
  } else {
    var listElement = document.getElementById('list');
    var content = 'S3 Objects in ' + bucket + ':n';
    // 반환된 객체의 키 값을 출력합니다
    content +=  data.Contents.map(function(metadata) {
      return 'Key: ' + metadata.Key;
    }).join('n');
    listElement.innerText = content;
  }
});
```
  
지금, Amazon S3에 요청을 담당하는 자바스크립트 파일이 하나, 
웹페이지에 S3 객체 목록을 추가하는 자바스크립트 파일이 하나, 
하나의 div 태그와 script 태그를 가지고 있는 HTML파일이 있습니다. 
웹페이지에서 데이터를 출력주기 전 마지막 순서로 webpack을 사용해서 script 태그를 참조하는 bundle.js 파일을 생성합니다.  

##webpack 설정하기
일반 자바스크립트 파일을 사용해서 webpack의 설정 옵션을 지정합니다. 
기본적으로 webpack은 프로젝트의 루트 디렉토리에 있는 webpack.config.js 을 찾습니다. 
webpack.config.js 파일을 만들어 봅시다.  
  
**webpack.config.js**  
```JavaScript
// 파일 경로를 분석하기 위해 path를 포함합니다
var path = require('path');
module.exports = {
  // 어플리케이션의 진입점을 지정합니다.
  entry: [
    path.join(__dirname, 'browser.js')
  ],
  // 번들된 코드를 포함하는 출력 파일을 지정합니다
  output: {
    path: __dirname,
    filename: 'bundle.js'
  },
  module: {
    /**
      * webpack에게 'json'파일 읽는 방법을 말해줍니다
      * 왜냐하면 기본적으로 webpack은
      * 자바스크립트 파일을 다루는 방법만 알기 때문입니다.
      * webpack이 'json' 파일이 포함되어있는 'require()' 구문을 만나면
      * json-loader를 사용하게 됩니다
      */
    loaders: [
      {
        test: /.json$/, 
        loaders: ['json']
      }
    ]
  }
}
```
  
webpack.config.js에서 진입점을 browser.js로 지정했습니다. 진입점이란 포함된 모듈 탐색을 시작하는 파일입니다. 
그리고 출력을 bundle.js로 정의했습니다. 이 번들은 어플리케이션을 실행하기 위해 필요한 모든 자바스크립트를 포함합니다. 
s3.js가 browser.js에 포함되어있기 때문에 webpack은 이미 s3.js를 포함하는 것을 알고있습니다. 
따라서 s3.js를 진입점으로 지정할 필요가 없습니다. 
또한, aws-sdk가 s3.js에 포함되어 있기 때문에 aws-sdk를 포함해야 하는 것도 알고있습니다.  
  
webpack에게 포함된 JSON 파일을 다루는 방법을 말해주는 로더를 지정했음을 주목하세요, 이 경우에 이전에 설치한 json-loader를 사용합니다. 
기본적으로 webpack은 자바스크립트 파일만 지원하지만 로더를 사용해서 다른 종류의 파일을 포함하는 지원 또한 추가할 수 있습니다. 
AWS SDK는 JSON 파일을 엄청나게 사용하기 때문에 이런 추가적인 설정이 없으면 webpack이 번들을 생성할 때 에러를 던집니다.  
  
##webpack 실행하기
어플리케이션을 빌드할 준비가 거의 다 됐습니다! package.json에서 scripts 객체에 "build": "webpack" 를 추가하세요.  
```javascript
{
  "name": "aws-webpack",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo "Error: no test specified" && exit 1",
    "build": "webpack"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "aws-sdk": "^2.6.1"
  },
  "devDependencies": {
    "json-loader": "^0.5.4",
    "webpack": "^1.13.2"
  }
}
```
  
이제 커맨드라인에서 npm run build 명령어를 실행하면 webpack이 프로젝트 파일의 루트 디렉토리에 bundle.js 파일을 생성합니다. 
webpack이 알려주는 결과가 아래와 같아야 합니다:

```bash
    Version: webpack 1.13.2
    Time: 1442ms
      Asset     Size  Chunks             Chunk Names
    bundle.js  2.38 MB     0  [emitted]  main
      [0] multi main 28 bytes {0} [built]
      [1] ./browser.js 653 bytes {0} [built]
      [2] ./s3.js 760 bytes {0} [built]
       + 343 hidden modules   
```
  
  
이제 브라우저에서 index.html파일을 열 수 있고 예시에 있는 것과 같은 출력을 볼 수 있습니다.
  
##시도해 보세요!

다음 글에서는 webpack과 자바스크립트용 AWS SDK를 사용하는 것의 다른 기능을 살펴보겠습니다.  
  
자바스크립트용 AWS SDK v2.6.1 에서 새로 webpack을 지원하는 것에 대한 의견을 듣고 싶습니다! 시도해 보시고 피드백을 코멘트나 [Github](https://github.com/aws/aws-sdk-js)에 남겨주세요!