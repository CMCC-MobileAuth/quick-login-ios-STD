# 1. 接入指南

sdk技术问题沟通QQ群：609994083</br>
sdk支持版本：iOS8.0及以上</br>
本文档为一键登录SDK5.3.11版本的开发文档</br>


**注意事项：**

1. **认证取号服务必须打开蜂窝数据流量并且手机操作系统给予应用蜂窝数据权限才能使用**
2. **取号请求过程需要消耗用户少量数据流量（国外漫游时可能会产生额外的费用）**
3. **认证取号服务目前支持中国移动2/3/4G和中国电信4G**
4. **（新增）建议全部SDK方法由主线程发起调用，因为SDK的数据容器就是在主线程进行读写操作的，如果外部发起调用SDK方法的线程不是主线程可能会导致多线程对数据容器读写导致崩溃**

## 1.1. 接入流程

**1.申请appid和appkey**

根据《开发者接入流程文档》，前往中国移动开发者社区（dev.10086.cn)，按照文档要求创建开发者账号并申请appid和appkey，并填写应用的包名（bundle ID)。

**2.申请能力**

应用创建完成后，在页面左侧选择一键登录能力，配置应用服务器出口IP地址及验签方式。

## 1.2. 开发流程

**第一步：下载SDK及相关文档**

请在开发者群或官网下载最新的SDK包

**第二步：搭建开发环境**

1. xcode版本需使用9.0以上，否则会报错
2. 导入认证SDK的framework，直接将移动认证`TYRZSDK.framework`拖到项目中
3. 在Xcode中找到`TARGETS-->Build Setting-->Linking-->Other Linker Flags`在这选项中需要添加`-ObjC`</br>
注意:如果以上操作仍然出现unrecognized selector sent to instance找不到方法的报错,则添加更改为_all_load
4. 资源文件:在Xcode中务必导入TYRZResource.bundle到项目中，否则授权界面显示异常（不显示默认图片） `TARGETS-->Build Phases-->Copy Bundle Resources-> 点击 "+" --> Add Other --> TYRZSDK.frameWork --> TYRZResource.bundle -->Open `

**第三步：开始使用移动认证SDK**

**[1] 初始化SDK**

在appDelegate.m文件的`didFinish`函数中添加初始化代码。初始化代码只需要执行一次就可以。

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    [UASDKLogin.shareLogin registerAppId:@"xxxxxxxx" AppKey:@"xxxxxxxx"];

        
    return YES;
}
```

**方法原型：**

```objective-c
- (void)registerAppId:(NSString *)appId AppKey:(NSString *)appKey;
```

**参数说明：**

| 参数   | 类型     | 说明        |
| ------ | -------- | ----------- |
| appId  | NSString | 应用的appid |
| appKey | NSString | 应用密钥    |

<div STYLE="page-break-after: always;"></div>
# 2. 一键登录功能

## 2.1. 准备工作

在中国移动开发者社区进行以下操作：

1. 获得appid和appkey、APPSecret（服务端）；
2. 勾选一键登录能力；
3. 配置应用服务器的出口ip地址
4. 配置公钥（如果使用RSA加密方式）

## 2.2. 流程说明

![](image/login.png)

## 2.3. 取号请求

本方法用于发起取号请求，SDK完成网络判断、蜂窝数据网络切换等操作并缓存凭证scrip。

**取号方法原型**

```objective-c
- (void)getPhoneNumberCompletion:(void(^)(NSDictionary *_Nonnull result))completion;
```

**参数说明：**

| 参数     | 类型  | 说明     |
| -------- | ----- | -------- |
| complete | Block | 取号回调 |

**响应参数：**

| 参数       | 类型     | 说明             |
| ---------- | -------- | ---------------- |
| resultCode | NSString | 返回相应的结果码 |
| desc       | NSString | 调用描述         |

**请求示例代码**

```objective-c
[UASDKLogin.shareLogin getPhoneNumberCompletion:^(NSDictionary * _Nonnull sender) {
     
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

**使用方法：**

通过设置Model属性中SMSAuthOn值确定是否使用短信验证码服务(参考2.6.2)

NO（默认）：不使用SDK提供的短验服务，此时如果用户点击“切换账号”，SDK将返回200060返回码

YES：使用SDK提供的短验服务


## 2.5. 授权请求

应用调用本方法时，SDK将拉起用户授权页面，用户确认授权后，SDK将返回token给应用客户端。可通过返回码200087监听授权页是否成功拉起。

**授权请求方法原型**

```objective-c
- (void)getAuthorizationWithModel:(UACustomModel *)model 
                         complete:(void (^)(id sender))completion;
```

**请求参数：**

| 参数     | 类型          | 说明                              |
| -------- | ------------- | --------------------------------- |
| model    | UACustomModel | 需要配置的Model属性（控制器必传） |
| complete | Block         | 取号回调                          |

**响应参数：**

| 参数       | 类型     | 说明                                                         |
| ---------- | -------- | ------------------------------------------------------------ |
| resultCode | NSString | 返回相应的结果码                                             |
| token      | NSString | 成功时返回：临时凭证，token有效期2min，一次有效，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |


**请求示例代码**

```objective-c
//    NSDecimalNumber *begin = [self.class startLoading];
    UACustomModel *model = [[UACustomModel alloc]init];
    model.currentVC = self;//必传
    model.authPageBackgroundImage = [UIImage imageNamed:@"tooopen_sy_122409821526"];
//    model.numberSize = 10;
//    model.navReturnImg = [UIImage imageNamed:@"tooopen_sy_122409821526"];
//    model.logBtnImgs = @[[UIImage imageNamed:@"12341553737084_.pic_hd"],[UIImage imageNamed:@"12341553737084_.pic_hd"],[UIImage imageNamed:@"12341553737084_.pic_hd"]];
//    model.logBtnImgs = []
//    model.navCustom = YES;
//    model.authViewBlock = ^(UIView *customView) {
//        UIImageView *ima = [[UIImageView alloc]initWithImage:[UIImage imageNamed:@"tooopen_sy_122409821526"]];
//        ima.frame = customView.bounds;
//        [customView addSubview:ima];
//    };
//    model.customSMSFlag = YES;
    [UASDKLogin.shareLogin getAuthorizationWithModel:model complete:^(NSDictionary * _Nonnull sender) {
//        NSDecimalNumber *end = [self.class stopLoading];
//        NSDecimalNumberHandler *subHandler = [self.class roundPlainWithScale:3];
//        NSDecimalNumber *delta = [end decimalNumberBySubtracting:begin withBehavior:subHandler];
//        NSMutableDictionary *result = [sender mutableCopy];
//        result[@"duration"] = delta;
//        [self displayObject:result withTitle:@"一键登录" alertActionHandler:^(UIAlertAction * _Nonnull action) {

//        [self dismissViewControllerAnimated:YES completion:nil];
        [self displayObject:sender];

//        }];
    }];

```


## 2.6. 授权页面设计

为了确保用户在登录过程中将手机号码信息授权给开发者使用的知情权，一键登录需要开发者提供授权页登录页面供用户授权确认。开发者在调用授权登录方法前，必须弹出授权页，明确告知用户当前操作会将用户的本机号码信息传递给应用。

### 2.6.1. 页面规范细则

![](image/authUI.png)

![](image/SMSPage.png)



**注意：**

**1、开发者不得通过任何技术手段，破解授权页，或将授权页面的隐私栏、品牌露出内容隐藏、覆盖。**

**2、登录按钮文字描述必须包含“登录”或“注册”等文字，不得诱导用户授权。**

**3、对于接入移动认证SDK并上线的应用，我方会对上线的应用授权页面做审查，如果有出现未按要求弹出或设计授权页面的，将关闭应用的认证取号服务。**



###  2.6.2Model属性

通过model属性，可以实现：

1、可以允许开发者在授权页面上添加自定义的控件；

2、设置授权页面和短信验证码页面的元素控件的布局

**当前VC，注意：使用一键登录服务时，这个值必传**

```objective-c
@property (nonatomic,strong) UIViewController *currentVC;
```



**授权界面自定义控件View的Block**

| model属性     | 值类型     | 属性说明                 |
| ------------- | ---------- | ------------------------ |
| authViewBlock | UIView *customView, CGRect logoFrame, CGRect numberFrame, CGRect sloganFrame, CGRect loginBtnFrame, CGRect checkBoxFrame, CGRect privacyFrame | 设置授权页应用自定义控件 |

示例：

```model.authViewBlock = ^(UIView *customView, CGRect logoFrame, CGRect numberFrame, CGRect sloganFrame, CGRect loginBtnFrame, CGRect checkBoxFrame, CGRect privacyFrame) {
UIImageView *ima = [[UIImageView alloc]initWithImage:[UIImage imageNamed:@"tooopen_sy_122409821526"]];
        ima.frame = customView.bounds;
        [customView addSubview:ima];
    };
```
**授权界面自定义控件View的Block**

| model属性     | 值类型     | 属性说明                 |
| ------------- | ---------- | ------------------------ |
| presentType | UAPresentationDirection | 授权页面推出动画效果 |


**导航栏状态栏设置**

| model属性      | 值类型             | 属性说明                                                     |
| -------------- | ------------------ | ------------------------------------------------------------ |
| navColor       | UIColor            | 设置导航栏颜色（授权页短验页一致）                           |
| navText        | NSAttributedString | 设置导航栏标题文字、字体大小、颜色                           |
| barStyle       | UIBarStyle         | 状态栏着色样式（授权页短验页一致）                           |
| statusBarStyle | UIStatusBarStyle   | 状态栏着色样式（授权页短验页一致）(隐藏导航栏时设置)         |
| navReturnImg   | UIImage            | 设置导航栏返回按钮图标（授权页短验页一致）                   |
| navControl     | UIBarButtonItem    | 导航栏右侧自定义控件，如果导航栏隐藏了，该项不生效（授权页短验页一致） |
| navCustom      | BOOL               | 隐藏导航栏（授权页短验页一致）                               |

**授权页背景**

| 方法                    | 值类型  | 说明               |
| ----------------------- | ------- | ------------------ |
| authPageBackgroundImage | UIImage | 设置授权页背景图片 |

**授权页logo**

| model属性   | 值类型  | 属性说明                        |
| ----------- | ------- | ------------------------------- |
| logoImg     | UIImage | 设置logo图片                    |
| logoWidth   | CGFloat | 设置logo宽度                    |
| logoHeight  | CGFloat | 设置logo高度                    |
| logoOffsetY | CGFloat | 设置logo相对于标题栏下边缘y偏移 |
| logoHidden  | BOOL    | 隐藏logo                        |

**号码栏**

| model属性       | 值类型             | 属性说明                             |
| --------------- | ------------------ | ------------------------------------ |
| numberText      | NSAttributedString | 手机号码富文本设置（字体大小、颜色） |
| numberOffsetY | CGFloat            | 号码栏Y相对于标题栏下边缘y偏移       |
| numberOffsetX   | NSNumber           | 号码栏X相对于默认值的左右偏移        |

**登录按钮**

| model属性     | 值类型             | 属性说明                                                     |
| ------------- | ------------------ | ------------------------------------------------------------ |
| logBtnText    | NSAttributedString | 设置登录按钮的富文本属性（字体大小、颜色、文案内容）         |
| logBtnImgs    | NSArray            | 设置授权登录按钮三种状态的图片数组，数组顺序为：[0]激活状态的图片；[1] 失效状态的图片；[2] 高亮状态的图片 |
| logBtnOriginX | NSNumber           | 设置登录按钮距离屏幕的左右边距，注意:按钮呈现的宽必须大于屏幕的一半* |
| logBtnHeight  | CGFloat            | 设置登录按钮高h 注意：必须大于40*                            |
| logBtnOffsetY | CGFloat            | 设置登录按钮相对于标题栏下边缘y偏移                          |

**切换账号**

| model属性         | 值类型  | 属性说明                              |
| ----------------- | ------- | ------------------------------------- |
| switchOffsetY     | CGFloat | 设置切换账号相对于标题栏下边缘y偏移   |
| swithAccHidden    | BOOL    | 隐藏切换账号按钮，YES时隐藏“切换账号” |
| switchAccText | NSAttributedString | “切换账号”字体颜色、大小、文本设置                   |

**隐私条款**

| model属性         | 值类型                        | 属性说明                                                     |
| ----------------- | ----------------------------- | ------------------------------------------------------------ |
| appPrivacy        | NSArray<NSAttributedString*>* | APP自定义隐私条款:数组（务必按顺序）要设置NSLinkAttributeName属性可以跳转协议 比如:@[NSAttributedString对象,...] |
| appPrivacyDemo    | NSAttributedString            | 设置隐私的内容模板：，1、全句可自定义但必须保留"&&默认&&"字段表明SDK默认协议,否则设置不生效 2、协议1和协议2的名称要与数组 NSAttributedString1 和 NSAttributedString2 ... 里的名称 一样 3、必设置项（参考SDK的demo） appPrivacyDemo设置内容：登录并同意&&默认&&和&&百度协议&&、&&京东协议2&&登录并支持一键登录 展示：   登录并同意中国移动条款协议和百度协议1、京东协议2登录并支持一键登录 |
| uncheckedImg      | UIImage                       | 设置复选框未选中时图片                                       |
| checkedImg        | UIImage                       | 设置复选框选中时图片                                         |
| checkboxWH        | CGFloat                       | 复选框大小（只能正方形）必须大于12*/                         |
| appPrivacyAlignment | NSTextAlignment               | 设置隐私条款文字内容的方向:默认是居左                        |
| privacyColor      | UIColor                       | 设置隐私条款名称颜色（协议）                                 |
| privacyState      | BOOL                          | 隐私条款check框默认状态<br/>默认:NO                          |
| privacyOffsetY    | NSNumber                      | 设置隐私条款相对于授权页面底部下边缘y偏移                    |
| appPrivacyOriginX | NSNumber                      | 设置隐私协议距离屏幕的左右边距                               |

**授权页slogan**

| model属性     | 值类型             | 属性说明                           |
| ------------- | ------------------ | ---------------------------------- |
| sloganText    | NSAttributedString | slogan文字富文本（颜色、字体大小） |
| sloganOffsetY | CGFloat            | 设置slogan相对于标题栏下边缘y偏移  |

**弹窗授权页**

| model属性            | 值类型                 | 属性说明                                                     |
| :------------------- | :--------------------- | :----------------------------------------------------------- |
| authWindow           | BOOL                   | 窗口模式开关                                                 |
| controllerSize       | CGSize                 | 此属性支持半弹框方式与authWindow不同（此方式为UIPresentationController）设置后自动隐藏切换按钮 |
| cornerRadius         | CGFloat                | 自定义窗口弧度半径 默认是10                                  |
| modalTransitionStyle | UIModalTransitionStyle | 窗口模式推出动画（系统自带）                                 |
| navReturnImgFrame    | CGRect                 | 返回按钮frame设置 只有隐藏导航栏时有效 默认frame为 (20,居中,20 * 系数scaleW,20 * scaleW) |
| scaleH               | CGFloat                | 自定义窗口高-缩放系数(屏幕高乘以系数) 默认是0.5              |
| scaleW               | CGFloat                | 自定义窗口宽-缩放系数(屏幕宽乘以系数) 默认是0.8              |
| webNavReturnImg      | UIImage                | web协议界面导航返回图标(尺寸根据图片大小)                    |

**短信验证码页面**

| model属性          | 值类型             | 属性说明                                                     |
| ------------------ | ------------------ | ------------------------------------------------------------ |
| SMSAuthOn          | BOOL               | SDK短信验证码开关（默认为NO）</br>NO:不使用SDK提供的短验；</br>YES：使用SDK提供的短验 |
| SMSBackgroundImage | UIImage            | 短验界面背景                                                 |
| SMSNavText         | NSAttributedString | 设置短验页的导航栏标题文字、大小、颜色                       |
| SMSLogBtnText      | NSAttributedString | 设置短验页的按钮文字、大小、颜色                             |
| SMSLogBtnImgs      | NSArray            | 设置短验登录按钮三种状态的图片数组，数组顺序为：[0]激活状态的图片；[1] 失效状态的图片；[2] 高亮状态的图片 |
| SMSGetCodeBtnImgs  | NSArray            | 设置获取验证码按钮三种状态的图片数组，数组顺序为：[0]激活状态的图片；[1] 失效状态的图片 |

### 2.6.3. 授权页面的关闭

开发者可以自定义关闭授权页面。

**代码示例**

```objective-c
…………
[self dismissViewControllerAnimated:YES completion:nil];
…………
```
### 2.6.4. 弹窗设置（中间弹窗）

**代码示例（有关model设置,注意控件大小可根据需求去设置,更加详细的上下文可移步到demo查看）**
```objective-c
#pragma mark ----------------------弹窗竖屏:(温馨提示:由于受屏幕影响，小屏幕（5S,5E,5）需要改动字体和另自适应和布局)--------------------
    if (self.authWindow && isPortrait) {
        //⑴居中弹窗 特殊方式 -----务必在设置控件位置时，自行查看各个机型是否正常
        
        //要匹配更多屏幕,此处为控件大小控制设置（建议只更改Logo大小）
        model.authWindow = YES;
        model.cornerRadius = 15;
        //46、居中弹窗开关
        //UIModalTransitionStyleCoverVertical, 下推 0
        //UIModalTransitionStyleFlipHorizontal,翻转 1
        //UIModalTransitionStyleCrossDissolve, 淡出 2
        //47、弹窗退出动画
        model.modalTransitionStyle = 0;
        //logo
        model.logoOffsetY = @50;
        //logo大小(可只设置宽SDK跟进图片比例算出等比高)
        model.logoWidth = 65;
        
        //48、弧度设置
        //49、自定义窗口宽-缩放系数(屏幕宽乘以系数) 默认是0.8 其它比例自行配置
        model.scaleW = 0.8;
        //50、自定义窗口高-缩放系数(屏幕高乘以系数) 默认是0.5 其它比例自行配置
        model.scaleH = 0.5;
        model.logBtnOriginX = @20;
//        model.checkboxWH = @12;
        model.privacyOffsetY = @10;
        model.appPrivacyOriginX = @20;
        model.appPrivacyAlignment = 1;
        //在5、5s、5e下,需要改字体才能适配
        BOOL isSmallScreen = UIScreen.mainScreen.bounds.size.height < 667.0f;
        if (isSmallScreen) {
            model.logoWidth = 50;
            model.logoOffsetY = @30;
            model.numberOffsetY = @90;
            model.sloganOffsetY = @115;
            model.scaleW = 0.7;
            model.privacyOffsetY = @5;
            
        }
        //51、返回按钮frame
        model.navReturnImgFrame = CGRectMake(self.view.frame.size.width *model.scaleW - 10 - 20, 10, 20, 20);
        
        model.authViewBlock = ^(UIView *customView, CGRect logoFrame, CGRect numberFrame, CGRect sloganFrame, CGRect loginBtnFrame, CGRect checkBoxFrame, CGRect privacyFrame) {
            [self setThiirdViewWithCustom:customView isSmall:isSmallScreen frame:loginBtnFrame];
        };
        
        
    }
    
 #pragma mark ----------------------弹窗横屏:(温馨提示:由于受屏幕影响，小屏幕（5S,5E,5）需要改动字体和另自适应和布局)--------------------
    else if(self.authWindow &&!isPortrait){
        CGFloat overallScaleH = UIScreen.mainScreen.bounds.size.height/ 375.f;
        CGFloat overallScaleW = UIScreen.mainScreen.bounds.size.width/ 667.f;
        model.authWindow = YES;
        model.cornerRadius = 10;
        //46、居中弹窗开关
        //UIModalTransitionStyleCoverVertical, 下推 0
        //UIModalTransitionStyleFlipHorizontal,翻转 1
        //UIModalTransitionStyleCrossDissolve, 淡出 2
        //47、弹窗退出动画
        model.modalTransitionStyle = 1;
        BOOL isSmallScreen = UIScreen.mainScreen.bounds.size.height < 375.f;
        model.scaleH = 0.7;
        model.scaleW = 0.7;
        model.logoHidden = YES;
        
        model.numberOffsetX = @(-155 * overallScaleW);
        model.logBtnOffsetY = @(110 * overallScaleH);
        model.numberOffsetY = @((110 + (40 - 23.86)/2) * overallScaleH);
        model.privacyOffsetY = @(55 * overallScaleH);
        model.sloganOffsetY = @(85 * overallScaleH);
        model.navReturnImgFrame = CGRectMake(self.view.frame.size.width *model.scaleW - 25 - 20, 35/2, 20, 20);
        model.authViewBlock = ^(UIView *customView, CGRect logoFrame, CGRect numberFrame, CGRect sloganFrame, CGRect loginBtnFrame,CGRect checkBoxFrame, CGRect privacyFrame) {
            
            CGFloat subViewH = isSmallScreen ? 45 : 55;
            UIImage *imageS = [UIImage imageNamed:@"中国移动.png"];
            CGFloat imageH = isSmallScreen ? 30 : 40;
            CGFloat widthI = imageS.size.width / imageS.size.height * imageH;
            UIImageView *imageV = [[UIImageView alloc]initWithImage:imageS];
            imageV.frame = CGRectMake(10, 10, widthI, imageH);
            [customView addSubview:imageV];
            
            UIView *line = [[UIView alloc]init];
            line.backgroundColor = [UIColor colorWithWhite:0 alpha:0.2];
            line.frame = CGRectMake(0, subViewH, customView.frame.size.width,1);
            [customView addSubview:line];
            
            UIView *thirdView = [[UIView alloc]init];
            thirdView.frame = CGRectMake(0, customView.frame.size.height - subViewH, customView.frame.size.width, subViewH);
            thirdView.backgroundColor = [UIColor colorWithRed:233/255.f green:234/255.f blue:254/255.f alpha:1];
            
            [customView addSubview:thirdView];
            
            //自定义------控件（仅供参考）
            //        CGFloat screenh = customView.frame.size.height;
            UIButton *btnWeChat = [self setBtnWith:@"微信1"];
            UIButton *btnQQ = [self setBtnWith:@"qq1"];
            UIButton *btnWebo = [self setBtnWith:@"微博1"];
            [btnWeChat setImageEdgeInsets:UIEdgeInsetsMake(5, 5, 5, 5)];
            [btnQQ setImageEdgeInsets:UIEdgeInsetsMake(5, 5, 5, 5)];
            [btnWebo setImageEdgeInsets:UIEdgeInsetsMake(5, 5, 5, 5)];
            
            UILabel *label = [[UILabel alloc]init];
            label.text = @"其它登录方式:";
            label.textColor = [UIColor grayColor];
            label.font = [UIFont systemFontOfSize:isSmallScreen ? 12 :15];
            label.textAlignment = NSTextAlignmentLeft;
            CGSize sizeLabel = [label.text sizeWithAttributes:@{NSForegroundColorAttributeName:label.textColor,NSFontAttributeName:label.font}];
            CGRect labelFrame =  label.frame;
            labelFrame.size = sizeLabel;
            label.frame = labelFrame;
            label.frame = CGRectMake(0, 0, thirdView.frame.size.width, label.frame.size.height);
            [thirdView addSubview:label];
            
            CGFloat btnW = subViewH - label.frame.size.height;
            CGFloat margin = 40;
            
            btnWeChat.frame = CGRectMake((thirdView.frame.size.width - 3 *btnW - 2 *margin)/2, CGRectGetMaxY(label.frame),btnW , btnW);
            btnQQ.frame = CGRectMake(CGRectGetMaxX(btnWeChat.frame) + margin, CGRectGetMaxY(label.frame), btnW, btnW);
            btnWebo.frame = CGRectMake(CGRectGetMaxX(btnQQ.frame) + margin, CGRectGetMaxY(label.frame), btnW, btnW);
            [thirdView addSubview:btnWeChat];
            [thirdView addSubview:btnQQ];
            [thirdView addSubview:btnWebo];
            
        };
        
    }
    
```

### 2.6.5. 弹窗设置（边缘弹窗,二选一，authWindow必须为NO,UIPresentationController为主要改VC大小）

**代码示例（有关model设置,注意控件大小可根据需求去设置,更加详细的上下文可移步到demo查看）**
```objective-c
#pragma mark ----------------------边缘弹窗:(温馨提示:authWindow必须为NO,由于受屏幕影响，小屏幕（5S,5E,5）需要改动字体和另自适应和布局)--------------------
    else if(!self.authWindow && isPortrait && self.edgeWindow){
        CGFloat overallScaleH = UIScreen.mainScreen.bounds.size.height/ 667.f;
        CGFloat overallScaleW = UIScreen.mainScreen.bounds.size.width/ 375.f;
        //推出动画由model中的presentType来决定
        model.presentType = 0;
        //53、边缘弹窗方式中的VC尺寸
        model.controllerSize = CGSizeMake(UIScreen.mainScreen.bounds.size.width, UIScreen.mainScreen.bounds.size.height/2);
        model.logoWidth = 45 * overallScaleW;
        model.logoOffsetY = @(15 *overallScaleH);
        model.numberOffsetY = @(70 * overallScaleH);
        model.privacyOffsetY = @((UIScreen.mainScreen.bounds.size.height)/2 - 15);
        
    }

根据原理，此方式也有其它方向设置,可修改VC推出动画实现,并且修改布局

```


## 2.7. 获取手机号码（服务端）

详细请开发者查看移动认证服务端接口文档说明。



# 3. 本机号码校验

## 3.1. 准备工作

在中国移动开发者社区进行以下操作：

1. 获得appid和appkey、APPSecret（服务端）；
2. 勾选本机号码校验能力；
3. 配置应用服务器的出口ip地址
4. 配置公钥（如果使用RSA加密方式）
5. 勾选本机号码校验短验辅助开关（可选）
6. 商务对接签约（未签约应用每个appid每天只能调用1000次）

## 3.2. 使用流程说明

![](image/mobile_auth.png)

## 3.3. 取号请求

详情可参考一键登录的取号请求说明（2.3章）



## 3.4. 本机号码校验请求token

开发者可以在应用内部任意页面调用本方法，获取本机号码校验的接口调用凭证（token）

*本机号码校验暂时不支持联通



**本机号码校验方法原型**

```java
/**
 本机号码校验
 @param complete 回调
 */
- (void)mobileAuthCompletion:(void(^)(NSDictionary *_Nonnull result))completion;

```

**请求参数说明：**

| 参数     | 类型   | 说明     |
| :------- | :----- | :------- |
| complete | String | 方法回调 |

**响应参数：**

| 字段       | 类型   | 含义                                                         |
| ---------- | ------ | ------------------------------------------------------------ |
| resultCode | Int    | 接口返回码，“103000”为成功。                                 |
| token      | String | 成功返回:临时凭证，token有效期2min，一次有效，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |



## 3.5. 本机号码校验（服务端）

详细请开发者查看移动认证服务端接口文档说明。



# 4. 其它SDK请求方法

## 4.1. 获取网络状态和运营商类型

本方法用于获取用户当前的网络环境和运营商

网络类型及运营商（双卡下，获取上网卡的运营商）

**原型**

```objective-c
@property (nonatomic,readonly) NSDictionary<NSString *, NSNumber *> *networkInfo;
```

**响应说明**

| 参数        | 类型         | 说明                                                   |
| ----------- | ------------ | ------------------------------------------------------ |
| networkInfo | NSDictionary | <运营商, 网络类型> |

字典对应的键值：

| 参数        | 类型     | 说明                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| networkType | NSString | 0.无网络;</br>1.数据流量;</br>2.wifi;</br>3.数据+wifi        |
| carrier     | NSNumber | 0.未知（未插sim卡，其它运营商等）;</br>1.中国移动;</br>2.中国联通;</br>3.中国电信 |

## 4.2. 删除临时取号凭证

本方法用于删除取号方法`getPhoneNumberCompletion`成功后返回的取号凭证scrip

**原型**

```objective-c
- (BOOL)delectScrip;
```

**响应说明**

| 参数  | 类型 | 说明                                          |
| ----- | ---- | --------------------------------------------- |
| state | BOOL | 删除结果状态，（YES：删除成功，NO：删除失败） |

<div STYLE="page-break-after: always;"></div>
## 4.3. 自定义请求超时设置

本方法用于设置取号、一键登录、本机号码校验请求的超时时间

**原型**

```objective-c
- (void)setTimeoutInterval:(NSTimeInterval)timeout;
```

**响应说明**

| 参数    | 类型           | 说明                                                         |
| ------- | -------------- | ------------------------------------------------------------ |
| timeout | NSTimeInterval | 设置取号、授权请求和本机号码校验请求时的超时时间，开发者不配置时，默认所有请求的超时时间都为8000，单位毫秒 |

<div STYLE="page-break-after: always;"></div>
# 5. 返回码说明

## 5.1. SDK返回码

| 返回码 | 返回码描述                                                   |
| ------ | ------------------------------------------------------------ |
| 103000 | 成功                                                         |
| 103101 | 请求签名错误                                                 |
| 103102 | 包签名/Bundle ID错误                                         |
| 103108 | 短信验证码错误                                               |
| 103109 | 短信验证码校验超时                                           |
| 103111 | 网关IP错误                                                   |
| 103119 | appid不存在                                                  |
| 103125 | 短验下发时，手机号填写格式错误                               |
| 103211 | 其他错误，（如有需要请联系qq群609994083内的移动认证开发）    |
| 103901 | 短验下发次数已达上限（5次/min,30次/day）                     |
| 103902 | scrip失效                                                    |
| 103911 | token请求过于频繁，10分钟内获取token且未使用的数量不超过30个 |
| 103273 | 预取号联通重定向（暂不支持联通取号）                         |
| 105002 | 移动取号失败                                                 |
| 105003 | 电信取号失败                                                 |
| 105021 | 已达当天取号限额                                             |
| 105302 | appid不在白名单                                              |
| 105313 | 非法请求                                                     |
| 200020 | 用户取消登录                                                 |
| 200021 | 数据解析异常                                                 |
| 200022 | 无网络                                                       |
| 200023 | 请求超时                                                     |
| 200025 | 其他错误（socket、系统未授权数据蜂窝权限等，如需要协助，请加入qq群发问） |
| 200027 | 未开启数据网络                                               |
| 200028 | 网络请求出错                                                 |
| 200038 | 异网取号网络请求失败                                         |
| 200048 | 用户未安装sim卡                                              |
| 200050 | EOF异常                                                      |
| 200060 | 切换账号（未使用SDK短验时返回）                              |
| 200061 | 授权页面异常                                                 |                                         |
| 200064 | 服务端返回数据异常                                           |
| 200072 | CA根证书校验失败                                             |
| 200080 | 本机号码校验仅支持移动手机号                                 |
| 200082 | 服务器繁忙                                                |
| 200086 | ppLocation为空                                               |
| 200087 | 授权页成功拉起                                            |
| 200089 | SDK正在处理                                             |


# 6.常见问题

产品简介

1. 一键登录与本机号码校验的区别？
   - 一键登录有授权页面，开发者经用户授权后可获得号码，适用于注册/登录场景；本机号码校验不返回号码，仅返回待校验号码是否本机的校验结果，适用于所有基于手机号码进行风控的场景。
2. 一键登录支持哪些运营商？
   - 一键登录目前仅支持移动、电信手机号码
3. 移动认证是否支持小程序和H5？
   - 暂不支持
4. 移动认证对于携号转网的号码，是否还能使用？
   - 移动认证SD不提供判断用户是否为携号转网的Api，但提供判断用户当前流量卡运营商的方法。即携号转网的用户仍然能够使用移动认证
5. 移动认证的原理？
   - 通过运营商数据网关获取当前流量卡的号码
6. 一键登录是否支持多语言？
   - 暂不支持
7. 一键登录是否具备用户取号频次限制？
   - 对获取token的频次有限制，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个

能力申请

1. 注册邮件无法激活
   - 由于各公司企业邮箱的限制，请尽量不使用企业邮箱注册，更换其他邮箱尝试；如无法解决问题，需发邮件至平台客服邮件激活：kfptfw@aspirecn.com
2. 服务器IP白名单填写有没有要求？
   - 业务侧服务器接口到移动认证接口访问时，会校验请求服务器的IP地址，防止业务侧用户信息被盗用风险。IP白名单目前同时支持IPv4和IPv6，支持最大4000字符，并支持配置IP段。
3. 安卓和苹果能否使用一个AppID？
   - 需分开创建appid
4. 包签名修改问题？
   - 包名和包签名提交后不支持修改，建议直接新建应用
5. 包签名不一致会有哪些影响？
   - SDK会无法使用

SDK使用问题：

1. 最新的移动服务条款在哪里查询？
   - 最新的授权条款请见：https://wap.cmpassport.com/resources/html/contract.html 
2. 授权页面可以关闭跳转到短信验证页面吗?是否可以选择使用应用原有的短验而不使用SDK自带的？
   - 可以，详见接入文档【使用短信验证码】章节。应用如果没有短验，可以自行去掉。
3. 一键登录sdk的短信验证页能不能单独调用？
   - 不能，短信验证码仅作为网关取号失败后的补充
4. 用户点击授权后，授权页会自动关闭吗？
   - 不能，需要开发者调用一下dissmiss，详情见【finish授权页】章节
5. 同一个token可以多次获取手机号码吗？
   - token是单次有效的，一个token最多只能获取一次手机号。
6. 如何判断调用方法是否成功？
   - 方法调用后SDK会给出返回码，103000为成功，其余为调用失败。建议应用捕捉这些返回码，可用于日常数据分析。



如果未能解决您的问题，请加入sdk技术问题沟通QQ群：609994083。

