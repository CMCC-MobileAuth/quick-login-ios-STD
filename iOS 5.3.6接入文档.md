# 1. 接入指南

sdk技术问题沟通QQ群：609994083</br>

**注意事项：**

1. **认证取号服务必须打开蜂窝数据流量并且手机操作系统给予应用蜂窝数据权限才能使用**
2. **取号请求过程需要消耗用户少量数据流量（国外漫游时可能会产生额外的费用）**
3. **认证取号服务目前支持中国移动2/3/4G和中国电信4G**

## 1.1. 接入流程

**1.申请appid和appkey**

根据《开发者接入流程文档》，前往中国移动开发者社区（dev.10086.cn)，按照文档要求创建开发者账号并申请appid和appkey，并填写应用的包名（bundle ID)。

**2.申请能力**

应用创建完成后，在能力配置页面上，勾选应用需要接入的能力类型，如一键登录，并配置应用的服务器出口IP地址。（如果在服务端需要用非对称加密方法对一些重要信息进行加密处理，请在能力配置页面填写RSA加密的公钥）

## 1.2. 开发流程

**第一步：下载SDK及相关文档**

请在开发者群或官网下载最新的SDK包

**第二步：搭建开发环境**

1. xcode版本需使用9.0以上，否则会报错
2. 导入认证SDK的framework，直接将移动认证`TYRZSDK.framework`拖到项目中
3. 在Xcode中找到`TARGETS-->Build Setting-->Linking-->Other Linker Flags`在这选项中需要添加`-ObjC`

**第三步：开始使用移动认证SDK**

**[1] 初始化SDK**

在appDelegate.m文件的`didFinish`函数中添加初始化代码。初始化代码只需要执行一次就可以。

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [TYRZUILogin initializeWithAppId:APPID appKey:APPKEY];
        
    return YES;
}
```

**方法原型：**

```objective-c
+ (void)initializeWithAppId:(NSString *)appId appKey:(NSString *)appKey;
```

**参数说明：**

| 参数   | 类型     | 说明        |
| ------ | -------- | ----------- |
| appID  | NSString | 应用的appid |
| appKey | NSString | 应用密钥    |

<div STYLE="page-break-after: always;"></div>

# 2. 一键登录功能

## 2.1. 准备工作

在中国移动开发者社区进行以下操作：

1. 获得appid和appkey；
2. 勾选一键登录能力；
3. 配置应用服务器的出口ip地址
4. 配置公钥（如果使用RSA加密方式）

## 2.2. 流程说明

![](image/login.png)

## 2.3. 取号请求

本方法用于发起取号请求，SDK完成网络判断、蜂窝数据网络切换等操作并缓存凭证scrip。

**取号方法原型**

```objective-c
+ (void)preGetPhonenumberWithTimeout:(NSTimeInterval)timeout 
    					  completion:(void(^)(id sender))complete;
```

**参数说明：**

| 参数     | 类型           | 说明                                         |
| -------- | -------------- | -------------------------------------------- |
| timeout  | NSTimeInterval | 自定义取号超时时间（默认8000毫秒），单位：ms |
| complete | Block          | 取号回调                                     |

**响应参数：**

| 参数       | 类型     | 说明             |
| ---------- | -------- | ---------------- |
| resultCode | NSString | 返回相应的结果码 |
| desc       | NSString | 调用描述         |

**请求示例代码**

```objective-c
[TYRZUILogin preGetPhonenumberWithTimeout:8000 
 						       completion:^(id sender) {
        NSString *resultCode = sender[@"resultCode"];
        NSMutableDictionary *result = [NSMutableDictionary dictionaryWithDictionary:sender];
        if ([resultCode isEqualToString:CLIENTSUCCESSCODECLIENT]) {
            NSLog(@"预取号成功");
        } else {
            NSLog(@"预取号失败");
        }
        [self showInfo:result];
    }];
```

## 2.4. 使用短信验证码（可选）

SDK提供短信验证码作为网关取号的补充功能，短验功能只有在网关取号失败时才能使用。

**注意：**

1. 目前短信验证码只支持移动和电信手机号码
2. 无网络时，不提供短验服务
3. 未获取`READ_PHONE_STATE`授权时，不提供短验服务

**短信验证码开关原型：**

```objective-c
+ (void)enableCustomSMS:(BOOL)state;
```

**参数说明：**

| 参数  | 类型 | 说明                                                         |
| ----- | ---- | ------------------------------------------------------------ |
| state | BOOL | 是否使用开发者自定义短验（YES：使用；NO：不使用），默认值为NO |

**短信验证使用场景（enableCustomSMS打开前提）：**

一、开发者调用2.5中的授权请求方法失败时，自动跳转到短信验证码页面；

二、开发者在授权页面切换，其中

“切换账号”按钮隐藏时，无法在授权页面跳转到短验页面；

“切换账号”按钮显示时，

1. 当enableCustomSMS设置为打开时，点击“切换账号”，跳转到SDK短验
2. 当enableCustomSMS设置为关闭时，点击“切换账号”，SDK将返回200060返回码，应用根据该返回码自行实现页面的跳转。

![](image/sms_logic.png)

示例代码如下：

```objective-c
[TYRZUILogin getTokenExpWithController:self timeout:-1 complete:^(id sender) {
       
    NSLog(@"显示登录:%@",sender);
        NSString *resultCode = sender[@"resultCode"];
        self.token = sender[@"token"];
        NSMutableDictionary *result = [sender mutableCopy];
        NSLog(@"result = %@",result);
        if ([resultCode isEqualToString:CLIENTSUCCESSCODECLIENT]) {
            [self dismissViewControllerAnimated:YES completion:nil];
            result[@"result"] = @"获取token成功";
            
            //用户点击了“切换账号”（customSMS为YES才返回）
        }else if ([resultCode isEqualToString:@"200060"]){
            
            UINavigationController *nav = sender[@"NavigationController"];
            UIViewController *vc = [[UIViewController alloc]init];
            
            
            //导航栏push模式，可以跳回授权页面
            [nav pushViewController:vc animated:YES];

            //present模式，无法跳回授权页面
            //[self presentViewController:vc animated:YES completion:nil];
        }
        else {
            [self dismissViewControllerAnimated:YES completion:nil];
            result[@"result"] = @"获取token失败";
        }
            [self showInfo:result];        
    }];

```


## 2.5. 授权请求

应用调用本方法时，SDK将拉起用户授权页面，用户确认授权后，SDK将返回token给应用客户端。

**授权请求方法原型**

```objective-c
+ (void)getTokenExpWithController:(UIViewController *)vc
                          timeout:(NSTimeInterval)timeout
                         complete:(void (^)(id sender))complete;
```

**请求参数：**

| 参数     | 类型             | 说明                                         |
| -------- | ---------------- | -------------------------------------------- |
| vc       | UIViewController | 开发者调用一键登录方法时所在的页面控制器     |
| timeout  | NSTimeInterval   | 自定义取号超时时间（默认8000毫秒），单位：ms |
| complete | Block            | 取号回调                                     |

**响应参数：**

| 参数       | 类型     | 说明                                                         |
| ---------- | -------- | ------------------------------------------------------------ |
| resultCode | NSString | 返回相应的结果码                                             |
| openId     | NSString | 成功时返回：用户身份唯一标识                                 |
| token      | NSString | 成功时返回：临时凭证，token有效期2min，一次有效，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |


**请求示例代码**

```objective-c
[TYRZUILogin getTokenExpWithController:self 
 						       timeout:8000 
 							  complete:^(id sender) {
        						
                              //SDK响应时，客户端执行的逻辑
                                ……………… 
                                ………………
                              //可参考demo示例代码                   
    }
];

```


## 2.6. 授权页面设计

为了确保用户在登录过程中将手机号码信息授权给开发者使用的知情权，一键登录需要开发者提供授权页登录页面供用户授权确认。开发者在调用授权登录方法前，必须弹出授权页，明确告知用户当前操作会将用户的本机号码信息传递给应用。

### 2.6.1. 页面规范细则

![](image/authPage.png)

![](image/SMSPage.png)



**注意：开发者不得通过任何技术手段，将授权页面的隐私栏、品牌露出内容隐藏、覆盖，对于接入移动认证SDK并上线的应用，我方会对上线的应用授权页面做审查，如果有出现未按要求设计授权页面，将隐私栏、品牌等UI隐去不可见的设计，我方有权将应用的登录功能下线。**

### 2.6.2. 修改授权页布局

开发者通过调用授权页面布局方法customUIWithParams，根据2.6.1所示规则配置授权页面默认元素，并且允许开发者在授权页面上添加自定义的控件和事件。

**授权页面默认元素修改**

创建一个UACustomModel 类，设置好类的属性（属性名可参考UACustomModel.h文件查看或者查看2.6.3查看model属性）,然后将设置好的model实例作为参数传进 customUIWithParams方法里

**开发者自定义控件**

customUIWithParams将把授权页面customAreaView回调给开发者，开发者成功获取该页面后，可以在页面上添加自定义的元素和添加事件。

注意：开发者定义的元素无法覆盖授权页面的默认元素

**方法原型**

```objective-c
+ (void)customUIWithParams:(UACustomModel *)model
               customViews:(void(^)(UIView *customAreaView))customViews;
```

**参数说明**

| 参数        | 类型          | 说明                                      |
| ----------- | ------------- | ----------------------------------------- |
| model       | UACustomModel | 用于配置页面默认元素的类，具体可参考2.4.3 |
| customViews | UIView        | 开发者自定义控件                          |

**布局示例代码**

```objective-c
UACustomModel *model = [[UACustomModel alloc]init];
	model.navReturnImg = [UIImage imageNamed:@"delete.png"];
    model.logoImg = [UIImage imageNamed:@"(friend_quan)_[图片]SFont.CN"];
    model.logoWidth = 100;
    model.logoHeight = 120;
    model.numberColor = [UIColor blackColor];
    model.navText = [[NSAttributedString alloc]initWithString:@"你好" attributes:@{}];
  	...............
	...............
	...............
    [TYRZUILogin customUIWithParams:model 
     					customViews:^(UIView *customAreaView) {
        UIImageView *iamgeView = [[UIImageView alloc]initWithImage:[UIImage imageNamed:@"WechatIMG16.jpeg"]];
//        [customAreaView addSubview:iamgeView];
                        }
    ];

```

### 2.4.3. Model属性

**授权页导航栏**

| model属性    | 值类型             | 属性说明               |
| ------------ | ------------------ | ---------------------- |
| navColor     | UIColor            | 设置导航栏颜色         |
| navText      | NSAttributedString | 设置导航栏标题文字     |
| barStyle     | UIBarStyle         | 状态栏着色样式         |
| navReturnImg | UIImage            | 设置导航栏返回按钮图标 |
| navControl   | UIBarButtonItem    | 导航栏右侧自定义控件   |

**授权页logo**

| model属性   | 值类型  | 属性说明                        |
| ----------- | ------- | ------------------------------- |
| logoImg     | UIImage | 设置logo图片                    |
| logoWidth   | CGFloat | 设置logo宽度                    |
| logoHeight  | CGFloat | 设置logo高度                    |
| logoOffsetY | CGFloat | 设置logo相对于标题栏下边缘y偏移 |
| logoHidden  | BOOL    | 隐藏logo                        |

**号码栏**

| model属性       | 值类型  | 属性说明                              |
| --------------- | ------- | ------------------------------------- |
| oldStyle        | BOOL    | 显示旧版号码栏样式，YES时显示旧版样式 |
| numberColor     | UIColor | 手机号码字体颜色                      |
| numFieldOffsetY | CGFloat | 号码栏Y偏移量                         |

**登录按钮**

| model属性       | 值类型   | 属性说明                                                     |
| --------------- | -------- | ------------------------------------------------------------ |
| logBtnText      | NSString | 设置登录按钮文字                                             |
| logBtnTextColor | UIColor  | 设置登录按钮文字颜色                                         |
| logBtnImgs      | NSArray  | 设置授权登录按钮三种状态的图片数组，数组顺序为：[0]激活状态的图片；[1] 失效状态的图片；[2] 高亮状态的图片 |
| logBtnOffsetY   | CGFloat  | 设置登录按钮相对于标题栏下边缘y偏移                          |

**切换账号**

| model属性         | 值类型  | 属性说明                              |
| ----------------- | ------- | ------------------------------------- |
| switchOffsetY     | CGFloat | 设置切换账号相对于标题栏下边缘y偏移   |
| swithAccHidden    | BOOL    | 隐藏切换账号按钮，YES时隐藏“切换账号” |
| swithAccTextColor | UIColor | “切换账号”字体颜色                    |

**隐私条款**

| model属性       | 值类型  | 属性说明                                                     |
| --------------- | ------- | ------------------------------------------------------------ |
| appPrivacyOne   | NSArray | 设置开发者隐私条款1名称和URL(名称,url)                       |
| appPrivacyTow   | NSArray | 设置开发者隐私条款2名称和URL(名称,url)                       |
| appPrivacyColor | NSArray | 设置隐私条款名称颜色和协议文字颜色(基础文字颜色，协议文字颜色) |
| uncheckedImg    | UIImage | 设置复选框未选中时图片                                       |
| checkedImg      | UIImage | 设置复选框选中时图片                                         |
| privacyOffsetY  | CGFloat | 设置隐私条款相对于授权页面底部下边缘y偏移                    |

**授权页slogan**

| model属性       | 值类型  | 属性说明                          |
| --------------- | ------- | --------------------------------- |
| sloganTextColor | UIColor | 设置移动slogan文字颜色            |
| sloganOffsetY   | CGFloat | 设置slogan相对于标题栏下边缘y偏移 |

**短信验证码页面**

| model属性          | 值类型             | 属性说明                                                     |
| ------------------ | ------------------ | ------------------------------------------------------------ |
| SMSNavText         | NSAttributedString | 设置短验页的导航栏标题文字                                   |
| SMSLogBtnText      | NSString           | 设置短验页的按钮文字                                         |
| SMSLogBtnImgs      | NSArray            | 设置短验登录按钮三种状态的图片数组，数组顺序为：[0]激活状态的图片；[1] 失效状态的图片；[2] 高亮状态的图片 |
| SMSLogBtnTextColor | UIColor            | 设置短验页的按钮文字颜色                                     |

### 2.4.4. 授权页面的关闭

开发者可以自定义关闭授权页面。

**代码示例**

```objective-c
…………
[self dismissViewControllerAnimated:YES completion:nil];
…………
```

## 2.7. 获取手机号码（服务端）

开发者获取token后，需要将token传递到应用服务器，由应用服务器发起获取用户手机号接口的调用。

调用本接口，必须保证：

1. token在有效期内。（2分钟）
2. token还未使用过。
3. 应用服务器出口IP地址在开发者社区中配置正确。
4. 如果使用RSA加密，确保应用的公钥在开发者社区正确填写。

**接口说明：**

请求地址：https://www.cmpassport.com/unisdk/rsapi/loginTokenValidate

协议： HTTPS 

请求方法： POST+json,Content-type设置为application/json

**参数说明：**

1、json形式的报文交互必须是标准的json格式

2、发送时请设置content type为 application/json

3、参数类型都是String

**请求参数**

| 参数                | 是否必填 | 说明                                                         |
| :------------------ | :------: | :----------------------------------------------------------- |
| version             |    是    | 填2.0                                                        |
| msgid               |    是    | 标识请求的随机数即可(1-36位)                                 |
| systemtime          |    是    | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| strictcheck         |    是    | 暂时填写"0"，填写“1”时，将对服务器IP白名单进行强校验（后续将强制要求IP强校验） |
| appid               |    是    | 业务在统一认证申请的应用id                                   |
| expandparams        |    否    | 扩展参数                                                     |
| token               |    是    | 需要解析的凭证值。                                           |
| sign                |    是    | 当**encryptionalgorithm≠"RSA"**时，sign = MD5(appid + version + msgid + systemtime + strictcheck + token + appkey)（注：“+”号为合并意思，不包含在被加密的字符串中），输出32位大写字母；</br>当**encryptionalgorithm="RSA"**，业务端RSA私钥签名（appid+token）, 服务端使用业务端提供的公钥验证签名（公钥可以在开发者社区配置）。 |
| encryptionalgorithm |    否    | 推荐使用。开发者如果需要使用非对称加密算法时，填写“RSA”。（当该值不设置为“RSA”时，执行MD5签名校验） |

**响应参数**

| 参数         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| inresponseto | 对应的请求消息中的msgid                                      |
| systemtime   | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| resultcode   | 返回码                                                       |
| msisdn       | 表示手机号码，如果加密方式为RSA，应用需要用私钥进行解密      |

## 2.8. 本机号码校验（服务端）

开发者获取token后，需要将token传递到应用服务器，由应用服务器发起本机号码校验接口的调用。

调用本接口，必须保证：

1. token在有效期内（2分钟）
2. token还未使用过
3. 应用服务器出口IP地址在开发者社区中配置正确。

对于本机号码校验，需要注意：

1. 本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限，详情可咨询商务。
2. 签订合同后，将不在提供每天免费的测试次数。

**接口说明：**

请求地址： https://www.cmpassport.com/openapi/rs/tokenValidate

协议： HTTPS

请求方法： POST+json,Content-type设置为application/json

**参数说明：**

1、json形式的报文交互必须是标准的json格式

2、发送时请设置content type为 application/json

3、参数类型都是String

**请求参数**

| 参数          | 层级  | 是否必填                     | 说明                                                         |
| ------------- | ----- | ---------------------------- | ------------------------------------------------------------ |
| **header**    | **1** | 是                           |                                                              |
| version       | 2     | 是                           | 版本号,初始版本号1.0,有升级后续调整                          |
| msgId         | 2     | 是                           | 使用UUID标识请求的唯一性                                     |
| timestamp     | 2     | 是                           | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId         | 2     | 是                           | 应用ID                                                       |
| **body**      | **1** | 是                           |                                                              |
| openType      | 2     | 否，requestertype字段为0时是 | 运营商类型：</br>1:移动;</br>2:联通;</br>3:电信;</br>0:未知  |
| requesterType | 2     | 是                           | 请求方类型：</br>0:APP；</br>1:WAP                           |
| message       | 2     | 否                           | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| expandParams  | 2     | 否                           | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |
| keyType       | 2     | 否                           | 手机号码加密方式：</br>0:默认phonenum采用sha256加密，sign采用HMACSHA256算法</br>1:RSA加密（暂未支持）</br>（注：keyType=1时，phonenum和sign均使用RSA，keyType不填或非1、0时按keyType=0处理） |
| phoneNum      | 2     | 是                           | 待校验的手机号码的64位sha256值，字母大写。（手机号码 + appKey + timestamp， “+”号为合并意思）（注：建议开发者对用户输入的手机号码的格式进行校验，增加校验通过的概率） |
| token         | 2     | 是                           | 身份标识，字符串形式的token                                  |
| sign          | 2     | 是                           | 签名，HMACSHA256( appId + msgId + phonNum + timestamp + token + version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中，appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |

**响应参数**

| 参数         | 层级  | 说明                                                         |
| ------------ | ----- | :----------------------------------------------------------- |
| **header**   | **1** |                                                              |
| msgId        | 2     | 对应的请求消息中的msgid                                      |
| timestamp    | 2     | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId        | 2     | 应用ID                                                       |
| resultCode   | 2     | 平台返回码                                                   |
| **body**     | **1** |                                                              |
| resultDesc   | 2     | 平台返回码                                                   |
| message      | 2     | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| accessToken  | 2     | 使用短验辅助服务的凭证，当resultCode返回为001时，并且该appid在开发者社区配置了短验辅助功能时返回该参数。accessToken有效时间为5min，一次有效。 |
| expandParams | 2     | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |

**请求示例代码：**

```
{
    body =     {
        openType = 1;
        phoneNum =0A2050AC434A32DE684745C829B3DE570590683FAA1C9374016EF60390E6CE76;
        requesterType = 0;
        sign = 87FCAC97BCF4B0B0D741FE1A85E4DF9603FD301CB3D7100BFB5763CCF61A1488;
        token = STsid0000001517194515125yghlPllAetv4YXx0v6vW2grV1v0votvD;
    };
    header =     {
        appId = 3000******76;
        msgId = f11585580266414fbde9f755451fb7a7;
        timestamp = 20180129105523519;
        version = "1.0";
    };
}
```

<div STYLE="page-break-after: always;"></div>

<div STYLE="page-break-after: always;"></div>

##2.9. 获取网络状态和运营商类型

本方法用于获取用户当前的网络环境和运营商

**原型**

```objective-c
+(NSDictionary*)getNetworkType;
```

**响应说明**

| 参数        | 类型         | 说明                                                   |
| ----------- | ------------ | ------------------------------------------------------ |
| networkType | NSNumber | 0.无网络;</br>1.数据流量;</br>2.wifi;</br>3.数据+wifi  |
| carrier     | NSNumber | 0.未知;</br>1.中国移动;</br>2.中国联通;</br>3.中国电信 |

## 2.10. 删除临时取号凭证

本方法用于删除取号方法`getPhoneNumberWithTimeout`成功后返回的取号凭证scrip

**原型**

```objective-c
+(BOOL)delectScrip;
```

**响应说明**

| 参数  | 类型 | 说明                                          |
| ----- | ---- | --------------------------------------------- |
| state | BOOL | 删除结果状态，（YES：删除成功，NO：删除失败） |

<div STYLE="page-break-after: always;"></div>

# 3. 服务端接口说明

## 3.1. 获取手机号码接口

业务平台或服务端携带用户授权成功后的token来调用认证服务端获取用户手机号码等信息。

### 3.1.1. 接口说明

**请求地址：**https://www.cmpassport.com/unisdk/rsapi/loginTokenValidate

**协议：** HTTPS 

**请求方法：** POST+json,Content-type设置为application/json

**注意：开发者需到开发者社区填写服务端出口IP地址后才能正常使用**

</br>

### 3.1.2. 参数说明

1、json形式的报文交互必须是标准的json格式

2、发送时请设置content type为 application/json

3、参数类型都是String

**请求参数**

| 参数                | 是否必填 | 说明                                                         |
| :------------------ | :------: | :----------------------------------------------------------- |
| version             |    是    | 填2.0                                                        |
| msgid               |    是    | 标识请求的随机数即可(1-36位)                                 |
| systemtime          |    是    | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| strictcheck         |    是    | 暂时填写"0"，填写“1”时，将对服务器IP白名单进行强校验（后续将强制要求IP强校验） |
| appid               |    是    | 业务在统一认证申请的应用id                                   |
| expandparams        |    否    | 扩展参数                                                     |
| token               |    是    | 需要解析的凭证值。                                           |
| sign                |    是    | 当**encryptionalgorithm≠"RSA"**时，sign = MD5(appid + version + msgid + systemtime + strictcheck + token + appkey)（注：“+”号为合并意思，不包含在被加密的字符串中），输出32位大写字母；</br>当**encryptionalgorithm="RSA"**，业务端RSA私钥签名(appid+token), 服务端使用业务端提供的公钥验证签名（公钥可以在开发者社区配置）。 |
| encryptionalgorithm |    否    | 推荐使用。开发者如果需要使用非对称加密算法时，填写“RSA”。（当该值不设置为“RSA”时，执行MD5签名校验） |

**响应参数**

| 参数         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| inresponseto | 对应的请求消息中的msgid                                      |
| systemtime   | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| resultcode   | 返回码                                                       |
| msisdn       | 表示手机号码，如果加密方式为RSA，应用需要用私钥进行解密      |

### 3.1.3. 示例

**请求示例**

```
{
    appid = 3000******76; 
    msgid = 335e06a28f064b999d6a25e403991e4c;
    sign = 213EF8D0CC71548945A83166575DFA68;
    strictcheck = 0;
    systemtime = 20180129112955435;
    token = STsid0000001517196594066OHmZvPMBwn2MkFxwvWkV12JixwuZuyDU;
    version = "2.0";
}
```

**响应示例**

```
{
    inresponseto = 335e06a28f064b999d6a25e403991e4c;
    msisdn = 14700000000;
    resultCode = 103000;
    systemtime = 20180129112955477;
}
```

## 3.2. 本机号码校验

开发者获取token后，需要将token传递到应用服务器，由应用服务器发起本机号码校验接口的调用。

调用本接口，必须保证：

1. token在有效期内（2分钟）
2. token还未使用过
3. 应用服务器出口IP地址在开发者社区中配置正确。

对于本机号码校验，需要注意：

1. 本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限，详情可咨询商务。
2. 签订合同后，将不在提供每天免费的测试次数。

### 3.2.1. 接口说明

请求地址： https://www.cmpassport.com/openapi/rs/tokenValidate

协议： HTTPS

请求方法： POST+json,Content-type设置为application/json

### 3.2.2. 参数说明

1、json形式的报文交互必须是标准的json格式

2、发送时请设置content type为 application/json

3、参数类型都是String

**请求参数**

| 参数          | 层级  | 约束                         | 说明                                                         |
| ------------- | ----- | ---------------------------- | ------------------------------------------------------------ |
| **header**    | **1** | 必选                         |                                                              |
| version       | 2     | 必选                         | 版本号,初始版本号1.0,有升级后续调整                          |
| msgId         | 2     | 必选                         | 使用UUID标识请求的唯一性                                     |
| timestamp     | 2     | 必选                         | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId         | 2     | 必选                         | 应用ID                                                       |
| **body**      | **1** | 必选                         |                                                              |
| openType      | 2     | 否，requestertype字段为0时是 | 运营商类型：</br>1:移动;</br>2:联通;</br>3:电信;</br>0:未知  |
| requesterType | 2     | 必选                         | 请求方类型：</br>0:APP；</br>1:WAP                           |
| message       | 2     | 否                           | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| expandParams  | 2     | 否                           | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |
| keyType       | 2     | 否                           | 手机号码加密方式：</br>0:默认phonenum采用sha256加密，sign采用HMACSHA256算法</br>1:RSA加密（暂未支持）</br>（注：keyType=1时，phonenum和sign均使用RSA，keyType不填或非1、0时按keyType=0处理） |
| phoneNum      | 2     | 必选                         | 待校验的手机号码的64位sha256值，字母大写。（手机号码 + appKey + timestamp， “+”号为合并意思）（注：建议开发者对用户输入的手机号码的格式进行校验，增加校验通过的概率） |
| token         | 2     | 必选                         | 身份标识，字符串形式的token                                  |
| sign          | 2     | 必选                         | 签名，HMACSHA256(appId + msgId + phonNum + timestamp + token + version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中,appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |

**响应参数**

| 参数         | 层级  | 约束 | 说明                                                         |
| ------------ | ----- | :--- | :----------------------------------------------------------- |
| **header**   | **1** | 必选 |                                                              |
| msgId        | 2     | 必选 | 对应的请求消息中的msgid                                      |
| timestamp    | 2     | 必选 | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId        | 2     | 必选 | 应用ID                                                       |
| resultCode   | 2     | 必选 | 平台返回码                                                   |
| **body**     | **1** | 必选 |                                                              |
| resultDesc   | 2     | 必选 | 平台返回码                                                   |
| message      | 2     | 否   | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| accessToken  | 2     | 否   | 使用短验辅助服务的凭证，当resultCode返回为001时，并且该appid在开发者社区配置了短验辅助功能时返回该参数。accessToken有效时间为5min，一次有效。 |
| expandParams | 2     | 否   | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |

### 3.2.3. 示例

```
{
    body =     {
        openType = 1;
        phoneNum =0A2050AC434A32DE684745C829B3DE570590683FAA1C9374016EF60390E6CE76;
        requesterType = 0;
        sign = 87FCAC97BCF4B0B0D741FE1A85E4DF9603FD301CB3D7100BFB5763CCF61A1488;
        token = STsid0000001517194515125yghlPllAetv4YXx0v6vW2grV1v0votvD;
    };
    header =     {
        appId = 3000******76;
        msgId = f11585580266414fbde9f755451fb7a7;
        timestamp = 20180129105523519;
        version = "1.0";
    };
}
```

<div STYLE="page-break-after: always;"></div>

# 4. 返回码说明

## 4.1. SDK返回码

| 返回码 | 返回码描述                                   |
| ------ | -------------------------------------------- |
| 103000 | 成功                                         |
| 103108 | 短信验证码错误                               |
| 103125 | 手机号码格式错误                             |
| 200014 | 手机号码不存在                               |
| 200020 | 用户取消登录                                 |
| 200021 | 数据解析异常                                 |
| 200022 | 无网络                                       |
| 200023 | 请求超时                                     |
| 200025 | 其他错误（socket、系统未授权数据蜂窝权限等） |
| 200027 | 未开启数据网络                               |
| 200028 | 网络请求出错                                 |
| 200030 | 没有初始化参数                               |
| 200038 | 异网重定向失败                               |
| 200039 | 电信取号接口返回失败                         |
| 200048 | 未安装sim卡                                  |
| 200050 | EOF异常                                      |
| 200060 | 切换账号（未使用SDK短验时返回）              |
| 200061 | 授权页面异常                                 |
| 200062 | 预取号不支持联通                             |
| 200063 | 预取号不支持电信                             |


## 4.2. 获取手机号码接口返回码

| 返回码 | 返回码描述               |
| ------ | ------------------------ |
| 103000 | 返回成功                 |
| 103101 | 签名错误                 |
| 103113 | token内容错误            |
| 103119 | appid不存在              |
| 103133 | sourceid不合法           |
| 103211 | 其他错误                 |
| 103412 | 无效的请求               |
| 103414 | 参数校验异常             |
| 103811 | token为空                |
| 104201 | token失效或不存在        |
| 105018 | 用户权限不足             |
| 105019 | 应用未授权（开发者社区） |

## 4.3. 本机号码校验接口返回码

| 返回码 | 说明                       |
| ------ | -------------------------- |
| 000    | 是本机号码（纳入计费次数） |
| 001    | 非本机号码（纳入计费次数） |
| 002    | 取号失败                   |
| 003    | 调用内部token校验接口失败  |
| 004    | 加密手机号码错误           |
| 102    | 参数无效                   |
| 124    | 白名单校验失败             |
| 302    | sign校验失败               |
| 303    | 参数解析错误               |
| 606    | 验证Token失败              |
| 999    | 系统异常                   |
| 102315 | 次数已用完                 |