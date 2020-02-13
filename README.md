# Cognito Identity Pool + IoT Core 实现 Mobile 端用户对设备权限的精细化控制


此文章发表于AWS中国区官网blog：https://aws.amazon.com/cn/blogs/china/cognito-identity-pool-iot-core-realize-mobile-user-control/  

## 免责说明
建议测试过程中使用此方案，生产环境使用请自行考虑评估。
当您对方案需要进一步的沟通和反馈后，可以联系 nwcd_labs@nwcdcloud.cn 获得更进一步的支持。
欢迎联系参与方案共建和提交方案需求, 也欢迎在 github 项目issue中留言反馈bugs。


## 场景概述
目前，越来越多的Iot厂商会开发自己的APP，使得终端用户可以通过APP绑定自己的设备，检测自己设备的实时情况，并且对设备做即时的控制。在此场景下，Mobile端的终端用户应该只能发布消息到自己的设备。用户和设备的关系可能是一对多或者多对多,比如同一个user拥有多个设备，用户A只能发布消息到/userA/deviceX的topic中；多个设备可能受同时受多人控制，此时用户B和C都有权限发布消息到/device2/temp下。本文考虑到此两种场景并做展开，利用Cognito和AWS Iot Core，实现终端对设备的精细化控制权限管理。


## 架构图
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/cognito-with-iot-core-architecture.png)

通过cognito user pool，无需自己coding，即可轻松实现用户的注册、登录、注销等基本操作。Cognito Identity Pool可以与cognito user pool或是其他第三方账号(如google，facebook)做对接，利用IAM Role实现对AWS资源的精细化控制。本文同时使用cognito User Pool和cognito identity Pool，实现对Iot Core的访问管理。终端用户通过cognito user pool的用户池，获得登录token，通过此登录成功的token，拿到cognito Identity Pool Authorized Role的身份，使得他有权访问Iot Core并且只能发布消息到自己的设备控制topic。用户的登录ID和设备之间的绑定关系存储在AWS NoSQL数据库DynamoDB当中，用户只能发布消息到自己的Iot设备。


## 先决条件
0. 拥有AWS账号
1. 安装 [AWS Iot JavaScript SDK](https://github.com/aws/aws-iot-device-sdk-js#install)
2. 安装[Browserify](http://browserify.org/)将AWS IoT JavaScript SDK(Nodejs版本)转化为浏览器可以直接引用的JavaScript包. 

## 操作步骤

### 第一步：资源配置

#### 0. 创建dynamoDB table
创建名称为iot的table，将IdentityId为主键，其他的保留默认值即可. 此table主要用来维护用户id和设备id之间的mapping关系。
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/create-dynamodb-table.png)

#### 1. 创建cognito用户池user pool
输入user pool名称（如cognito-user-pool-for-iot），review defaults, 并根据需求做自定义修改（如可以修改necessary attributes，密码长度等），此demo均利用默认值。


#### 2. 创建并配置应用客户端app client
选择应用客户端，取消generate client secret的选项
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/create-app-client.png)

在左侧APP-Integration项目下，需要我们修改的有2个地方，一是APP client setting，修改callback URL以及scope token作用范围，二是自定义domain name（需要全region唯一）
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/app-client-config.png)
注意：localhost:8000仅在测试环境中使用。实际生产环境，这里的callback请修改为https的网址，注意：不支持http协议。请勿写入http://ip等形式。

记下userPoolID和app Client ID，在下一步骤中会用到。

#### 3. 创建cognito identity pool  
命名完毕后，对于authentication provider, 选择Cognito. 输入上一步记下的两个ID：user pool ID(User Pools → demo-pool → General Settings → Pool ID)以及app client ID(User Pools → demo-pool → App Integration → App client settings → demo-app-client→ ID)， 同时我们可以勾选允许unauth用户访问。
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/create-identity-pool.png)


#### 4. 设置cognito identity pool权限：授予Iot和DDB的访问权限
在点击创建后，进入到权限设置页面。可以在这里直接设置，也可以后续在IAM role当中随时更新policy。   
Identity pool会自动创建两个role，一种为unauthorized，即未登录仅游客身份的用户，可自行为此类用户授予一些基本的浏览权限；另外一种为authorized，成功验证身份的用户。   

在本文当中，我们主要实现的功能为登录用户可以并且只能访问到隶属于自己的设备，未登录用户仅能网页浏览并且有登录选项。 因此，对于授权用户我们赋予对用户自己的id下的设备有着“iot:Connect”, “iot:Publish”, “iot:Subscribe”, “iot:Receive”, “iot:GetThingShadow”的允许权限，以及访问dynamoDB，和修改dynamoDB的权限。${cognito-identity.amazonaws.com:sub}为从cognito-user-pool传递过来的变量，标记用户的identityID。每个user不同且唯一。我们通过此值来限定不同user之间的访问权限。

* 对于authorized users.  请下载[IAM policy](https://github.com/nwcdlabs/iot-mobile-control-using-cognito/blob/master/policy-rules/cognito-auth-policy.json)并且替换以下参数：
  - 请将<Your-AWS-Account-ID>替换为自己的12位ID（去掉<>两个尖括号）
  - 请将<region-code>去掉尖括号替换为自己使用的区域（去掉<>两个尖括号），如日本为ap-northeast-1，virginia为us-east-1,如使用其他region，请务必替换为自己的对应代码，其他region-code请参考[此页](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html). 本文使用日本区域ap-northeast-1做演示。 resource完整示例为arn:aws:iot:us-west-1:123456789102:client/${cognito-identity.amazonaws.com:sub}。

* Unauth-Role的policy如下

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mobileanalytics:PutEvents",
        "cognito-sync:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

#### 5. 在Iot中添加Iot policy：为Iot授予访问权限
在Iot core当中安全--策略(policy)页面，添加策略，命名cognito-identity-general-policy，具体权限如下。Cognito将通过attachPolicy命令为自己授予这条policy，使得每个Auth user有权限访问Iot且仅能发消息给自己的设备。此段policy也可以在[这里](https://github.com/nwcdlabs/iot-mobile-control-using-cognito/blob/master/policy-rules/iot-policy.json)找到

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:<your-region-code>:<account-id>:client/${cognito-identity.amazonaws.com:sub}"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": "arn:aws:iot:<your-region-code>:<account-id>:topic/${cognito-identity.amazonaws.com:sub}/*"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Subscribe",
      "Resource": "arn:aws:iot:<your-region-code>:<account-id>:topic/${cognito-identity.amazonaws.com:sub}/*"
    }
  ]
}

```

至此，通过cognito连接Iot并只能访问自己资源已经配置完毕，可以用自己的web页面嵌入此功能做验证或者测试。
本文用一个demo前端界面演示效果。

### 第二步：前端demo代码

此网页实现三个功能，一是登录，注册，是由cognito user pool来实现的；   

二是设备绑定，此网页模拟了用户拿到设备之后，手输设备号完成绑定的过程，在实际APP当中，这一步通常是由扫二维码的形式来实现绑定的，因web网页版不好模拟扫码，故用手输的方式；    

三是消息收发。点击已有设备，即模拟一次消息传输的过程。页面还有一个button是验证发送到其他topic会出现什么情况。

[>>>>>完整代码请点击这里<<<<<](hhttps://github.com/nwcdlabs/iot-mobile-control-using-cognito)下载。

1. (可选步骤，已经附此最终文件bundle.js)JS引入AWSIotDeviceSdk浏览器版本

（1）安装[AWSIotDeviceSdk](https://github.com/aws/aws-iot-device-sdk-js#install) Nodejs
```
#AWSIotDeviceSdk.js
var AwsIot = require('aws-iot-device-sdk');
window.AwsIot = AwsIot; // make it global
```
（2）在Shell当中运行browserify,将nodejs转化为前端可引入的JS

```
terminal> browserify path/to/AWSIotDeviceSdk.js -o bundle.js
```

2. 修改前端代码, 替换参数 
    
在[index.html](https://github.com/nwcdlabs/iot-mobile-control-using-cognito/blob/master/index.html)中修改以下内容：   

(1) 在function initCognitoSDK() 中替换所有的<your-region-code>，<your-app-client-id>，<your-custom-domain>,<identity-provider-id>,<your-user-pool-id>为自己的值，这一步是设置cognito的过程。

```
AWS.config.region ='<your-region-code>';   //替换为自己的region-code

    var authData = { 
      ClientId:'<your-app-client-id>', // APP client id here, 如5smxxxc7lcetqvao6ed967sq01
      AppWebDomain : '<your-custom-domain>', // 去掉 "https://" 部分. 样例格式：xxx.auth.ap-northeast-1.amazoncognito.com
      TokenScopesArray : ['openid','email'], // like ['openid','email','phone']...
      RedirectUriSignIn : 'http://localhost:8000/',   //只供本地测试，实际生产请写https网址，且必须要和前面控制台中cognito callback设置保持一致，不然会报redirect_mismatch的错误
      RedirectUriSignOut : 'http://localhost:8000/',   //只供本地测试，实际生产请写https网址，且必须要和前面控制台中cognito callback设置保持一致，不然会报redirect_mismatch的错误
      IdentityProvider : '<identity-provider-id>', //替换为自己的identity pool id
          UserPoolId : '<your-user-pool-id>',   //user pool ID,example: ap-northeast-1_410bH7K8x
          AdvancedSecurityDataCollectionFlag : false
    };

    //在onsuccess回调函数当中
    AWS.config.credentials = new AWS.CognitoIdentityCredentials({
        IdentityPoolId: '<identity-provider-id>',//identity provider id,example: ap-northeast-1: xxxxxxxx-xxxx-xxxx-xxxxxx
        Logins: login
    });

```
user pool ID在User Pools → demo-pool → General Settings → Pool ID    
app client ID在User Pools → demo-pool → App Integration → App client settings → demo-app-client→ ID可找到    

(2)如果iot policy为自己命名的，则attachPolicy("cognito-identity-general-policy", principal)第一个参数替换为自己的iot policy name。通过此操作，当前用户在拿到token后，可以实现与iot core的互通。

(3)在function connect(principal)当中替换<your-iot-endpoint>   

```
  device = AwsIot.device({
      clientId: clientID,
      host: '<your-iot-endpoint>',   //example: xxxxxx.iot.<your-region-code>f.amazonaws.com
      protocol: 'wss',
      accessKeyId: AWS.config.credentials.accessKeyId,   
      secretKey: AWS.config.credentials.secretAccessKey,
      sessionToken: AWS.config.credentials.sessionToken  
  });

```

此iot-endpoint可以在iot core-setting当中找到：
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/iot-endpoint.png)


(4)在function userButton(auth)当中替换自己的<your-custom-domain>，<your-region-code>，<your-client-id>，<your-call-back-url>，实现页面跳转。
```
  function userButton(auth) {
    var state = document.getElementById('signInButton').innerHTML;
    if (state === "Sign Out") {

      //*************************需自行修改，替换为自己的域名，clientid以及回调地址************************//
      document.getElementById("signInButton").href="https://<your-custom-domain>.auth.<your-region-code>.amazoncognito.com/logout?response_type=code&client_id=<your-client-id>&logout_uri=<your-call-back-url>";
      document.getElementById("signInButton").innerHTML = "Sign In";
      auth.signOut();
      showSignedOut();

    } else {
      //auth.getSession();
      //*************************需自行修改，替换为自己的域名，clientid以及回调地址************************//
      document.getElementById("signInButton").href="https://<your-custom-domain>.auth.<your-region-endpoint>.amazoncognito.com/login?response_type=code&client_id=<your-client-id>&redirect_uri=your-call-back-url>";
      
    }
      
    }

```

(5)在function publishMessage(env)当中，可以选择是否设置qos参数。这两种参数会在后续的实验中有不同的效果，我们先不设置qos试试看（默认）。


3. 搭建测试服务器

有两种方式：   

(1) localhost方式
在shell当中运行
```
python -m SimpleHTTPServer
```
在浏览器当中输入http://0.0.0.0:8000 或者 localhost:8000 进行验证。建议打开浏览器的developer tools查看日志以及MQTT传输过程。

注意：这种场景下，在点击sign in之后，因浏览器安全等级不同，有些浏览器可能会显示connecting... unable to connect websocket的error提示，这是因为页面停留在原http页面，无法自动进行证书验证，此时需要在浏览器新tab当中手动输入https://xxx（复制原本wss://xxxx后面的url）进行手动的加载证书的操作。之后再刷新原localhost:8000即可正常加载。


(2) 在实际生产当中，请直接打开https://your-own-domain进行测试。
留意：如果用自己的域名，请务必保持cognito--APPclientSetting中以及代码里面所有的callback回调地址也都改为your own domain，否则会报redirect_dismatch的错误


### 验证说明
 
（1）新建的cognito user pool是没有用户的，可以在页面验证用户注册和用户登录的过程，或者直接在cognito user pool当中手动创建新用户也可以。     
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/sign-in.png)

（2）原始dynamoDB当中没有数据，可以通过点击add a new device的按钮来模拟设备绑定的过程。这时可以输入一串字符（如iphone-15341）,点击submit按钮，等待几秒钟，在页面最下方即出现设备列表iphone-15341 publish。    
![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/register-new-device.png)

这时dynamodb写入一条新数据，代表新增一条identityID与device之间的绑定关系的记录。当用户下一次登录时，会直接展示这些device lists。   
这个表的结构如图所示：   

![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/iot-dynamodb-table.png)

（3）进入Iot-test页面，订阅#（通配符，即订阅所有topic）。在web页面点击刚刚出现的xxxx publish的按钮，可以在console当中看到实时的消息推送，此时Iot连接并且发布消息已成功。前端页面会显示发送出去的message的topic和具体内容。

![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/web_device_list.png)

![](https://salander.s3.cn-north-1.amazonaws.com.cn/public/cognito-with-iot-core/iot-subscribe-message.png)


（4）web页面的”Demo Unauthed situation“这个按钮，是模拟当前用户如果要发送不在权限范围内的情形，这个按钮会发送到名为test的topic。这时候我们点击此button，会出现两种不同的情况：   

* 如果在publishMessage当中，不设置Qos(默认代码)，这时候MQTT不会验证传输是否成功，尽管web页面会显示发送成功，然而在Iot的console当中，会发现这条消息实际是未送达且永远不会被送达的。

* 如果设置{qos:1}（代码改动如下） 

```
  function publishMessage(env) {
    var topic = env.target.topic;
    var msg = env.target.msg;
    //device.publish(topic,msg, function (err) {
    device.publish(topic,msg, { qos: 1 }, function (err) {
          if (err) {
              console.log("failed to publish iot message! ",topic);
              console.error(err);
          } else {
              console.log("published to TopicName: ", topic);
              openTab("messagedetails");
              showMessage("Message Published", "Topic: "+topic , "Message: "+msg);
          }
      });

  }
            
```   

因权限设置问题，Iot仍然无法收到这条消息，但是web页面会不断重连尝试重新发送，根据官方解释，Iot会尝试长达一个小时的重传，此时在点击demo unauthed situation的按钮后，页面会出现卡顿，打开developer tool会发现不停的reconnect尝试重传。此时点击其他publish的button也没有反应。


>注：实际生产中，因为不会有这样一个unauth test的button，因此可以设置qos:1。
>引用文档："AWS IoT will retry delivery of unacknowledged quality-of-service 1 (QoS 1) publish requests to a client 
>for up to one hour. If AWS IoT does not receive a PUBACK message from the client after one hour, it will drop the 
>publish requests." 


## 参考链接：

https://aws.amazon.com/cn/blogs/iot/configuring-cognito-user-pools-to-communicate-with-aws-iot-core/


