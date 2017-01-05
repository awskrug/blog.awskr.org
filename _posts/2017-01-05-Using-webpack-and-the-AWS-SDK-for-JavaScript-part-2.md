---
layout: post
title: "Webpack과 자바스크립트용 AWS SDK 를 사용해 어플리케이션을 만들고 번들하기 - 파트 2"
translate: 2017-01-05
author: Christopher Radek 
about_stub: 이상록
profile_picture:
author_site:
categories: JavaScript
---

#Webpack과 자바스크립트용 AWS SDK 를 사용해 어플리케이션을 만들고 번들하기 - 파트 2

[이전 글](http://www.awskr.org/2017/01/sing-webpack-and-the-aws-sdk-for-javascript-to-create-and-bundle-an-application-part-1/)에서 
webpack과 자바스크립트 AWS SDK를 사용해서 어플리케이션을 만들고 번들하는 방법에 대해서 소개했습니다.
  
이번 글에서는 필요한 AWS 서비스로만 번들 만들기, Node.js에서도 동작하는 번들 생성하기와 같은 다른 기능을 파헤쳐보겠습니다.
  
##개별 서비스를 포함하기
webpack을 사용하는 것의 장점 중 하나는 webpack이 의존성을 분석하고 어플리케이션이 필요한 코드만을 포함할 수 있다는 것입니다. 
이전 프로젝트에서 webpack이 2.38 MB짜리 번들을 생성했다는 것을 알아챘을 수도 있습니다. 
왜냐 하면 webpack이 현재 s3.js에 있는 아래 구문을 기반으로 자바스크립트 AWS SDK 전체를 가져오기 때문입니다.  
```JavaScript
var AWS = require('aws-sdk');
```
require 구문을 아래와 같이 바꾸면 webpack이 Amazon S3 서비스만을 가져오게 할 수 있습니다:
```JavaScript
var AWS = require('aws-sdk/clients/s3');
```
[AWS.config](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Config.html)를 사용해서 설정 가능한 모든 AWS SDK 옵션은 [서비스를 초기화](http://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/configuring-the-jssdk.html#Service-Specific_Configuration) 할 때도 설정할 수 있습니다. 
아래와 같은 구문으로 전역 설정을 하면 계속해서 모든 서비스에서 AWS 네임스페이스에 접근할 수 있습니다:
```JavaScript
var AWS = require('aws-sdk/global');
```
다음은 이런 변화를 주면 s3.js파일이 어떻게 바뀌는지를 보여주는 예시입니다.
  
**s3.js**
```JavaScript
// Amazon S3 서비스 클라이언트 가져오기
var S3 = require('aws-sdk/clients/s3');

// 보안 자격 증명과 지역 설정
var s3 = new S3({
    apiVersion: '2006-03-01',
    region: 'REGION', 
    credentials: {/* */}
  });

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
  
이제 npm run build 명령어를 실행하면 아래와 같은 결과를 보여줍니다.
```bash
    Version: webpack 1.13.2
    Time: 720ms
      Asset    Size  Chunks             Chunk Names
    bundle.js  797 kB     0  [emitted]  main
      [0] multi main 28 bytes {0} [built]
      [1] ./browser.js 653 bytes {0} [built]
      [2] ./s3.js 803 bytes {0} [built]
       + 155 hidden modules
```
  
이제 생성된 번들 용량이 2.38 MB 에서 797 KB로 떨어졌습니다.
  
최종 용량을 더 줄이기 위해 webpack에서 생성된 코드를 [축소하도록 설정](https://webpack.github.io/docs/optimization.html)할 수 있습니다!
  
##Node.js 번들 생성하기
webpack을 사용해서 설정에서 target: 'node' 를 지정하면 Node.js에서 작동하는 번들을 생성할 수 있습니다. 
디스크 용량이 제한되어 있는 환경에서 Node.js 어플리케이션을 실행할 때 유용할 수 있습니다.
  
node.js 파일을 생성해서, Node.js 번들을 빌드하도록 프로젝트를 업데이트 해 봅시다. 
이 파일은 browser.js 파일과 거의 동일합니다. 그러나 S3 객체를 DOM 나열하는 대신 콘솔에 출력합니다.
  
**node.js**
```JavaScript
// listObjects 함수를 가져옵니다
var listObjects = require('./s3');
var bucket = 'BUCKET';
// 지정된 버킷에서 listObjects 를 호출합니다
listObjects(bucket, function(err, data) {
  if (err) {
    console.log(err);
  } else {
    console.log('S3 Objects in ' + bucket + ':');
    // 반환된 각각 객체의 키값을 출력합니다
    data.Contents.forEach(function(metadata) {
      console.log('Key: ' + metadata.Key);
    });
  }
});
```
  
다음, node.js를 진입점으로 사용하도록 webpack.config.js 파일을 업데이트하고 webpack이 Node.js 번들을 생성해야 하는 것을 
알 수 있도록 target: "node" 라는 필드를 추가합니다.
  
**webpack.config.js**
```JavaScript
// 파일 경로를 분석하기 위해 path를 포함합니다
var path = require('path');
module.exports = {
  // 어플리케이션의 진입점을 지정합니다.
  entry: [
    path.join(__dirname, 'node.js')
  ],
  // 번들된 코드를 포함하는 출력 파일을 지정합니다
  output: {
    path: __dirname,
    filename: 'bundle.js'
  },
  // Node.js 번들을 생성해야 하는 것을 알려줍니다
  target: "node",
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
  
새로운 번들을 생성하기 위해 npm run build 명령어를 실행하세요. 
커맨드라인에 node bundle.js 명령어를 실행해서 이 코드를 테스트할 수 있습니다. 
이렇게하면 콘솔에 S3 객체 리스트가 출력됩니다!
  
##시도해 보세요!
자바스크립트용 AWS SDK v2.6.1 에서 새로 webpack을 지원하는 것에 대한 의견을 듣고 싶습니다! 시도해 보시고 피드백을 코멘트나 [Github](https://github.com/aws/aws-sdk-js)에 남겨주세요!