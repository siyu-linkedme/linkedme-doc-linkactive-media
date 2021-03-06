# API对接文档

本文档面向LinkActive媒体方，通过调用本文档中的接口来获取拉活的广告资源及回调广告行为。

调用文档接口前，请先联系LinkedME开通LinkActive平台权限，以获取媒体方唯一的linkedme\_key。

> 新增APP唤起微信小程序功能，广告获取接口同APP唤起APP接口一致，返回的参数中包含微信小程序必要的参数，请参考返回参数中红色字体部分；微信小程序的跳转逻辑请参考代码示例部分。

## API接入前提

您的APP需要满足一定条件才能够通过API接入，概括为两点：

* 能检测广告主的APP是否安装，并能直接唤起广告主APP（安装才展示广告，没有安装则不展示）
* 能获取到唤起广告主APP的状态，并回传给LinkActive

如果您的APP能支持以上两点，则可只接入服务端；若不支持则需要客户端做相应发版更新。

具体要求参见：[API接入前提](../../standard.md#api接入前提)

## 整体接入流程示意

![](../../.gitbook/assets/api-flow-1%20%281%29.jpg)

## LinkActive各端交互图及步骤说明

![](../../.gitbook/assets/linkactive-jiao-hu-tu%20%281%29.png)

### 服务端对接说明

建议广告请求过程通过媒体方服务端进行，客户端只处理广告展示及广告行为回调。

* 服务端需要对接 [获取广告]() 的接口（图中2）
* 服务端处理返回的广告结果（图中3）并回传给客户端（图中4）

### 客户端对接说明

* 客户端在用户访问时，向自己服务器请求广告（图中1）
* 客户端需要对服务端返回的广告（图中4）进行处理之后展示，返回结果处理详见 [广告展示规则]()
* 广告行为发生之后，需要调用 [回调广告行为]() 的接口（图中5）通知LinkActive服务端，回调的值参见 [发送回调通知说明]()
* 回调广告行为之后处理LinkActive服务端返回的结果（图中6）

#### 广告展示规则

LinkActive的接口请求广告时，返回的是按照优先级排序的每个广告主对应的广告素材，需要根据返回的字段“check\_install\_status”处理每个广告素材是否保留：如果该条广告check\_install\_status值为 1，需要判断用户安装状态，app未安装则删除该素材；如果该条广告check\_install\_status值为0，则不需判断app安装状态，直接展示。

为了防止一个设备对同一广告主产生两次拉活，在完成拉活之后需要做如下处理： 客户端从当次请求的广告列表中删除该条广告，并将其余广告向前顺移。具体流程如下图所示。

![](../../.gitbook/assets/guang-gao-zhan-shi-chu-li-liu-cheng%20%281%29.png)

包括如下几个主要步骤： 1. 判断广告资源是否为空，若为空则自行处理 2. 判断该广告是做拉活还是拉活兼拉新，如果只是拉活则需要判断广告主APP是否安装，将未安装的APP对应的广告删除；如果是拉活兼拉新，则不需判断广告主APP的安装状态 3. 获得有效广告列表，并按照顺序给用户做展示 4. 为了防止用户重复点击广告，删除已被点击的广告，并将广告列表中的其他广告向前顺移

#### 注意事项

* 拿到广告列表后首先需要处理将有效的广告保留下来，将无效（需要拉活但实际检测未安装的APP）的广告删除
* 用户点击广告后，客户端就应该把该次广告立即删除，以免产生二次点击
* 注意处理返回的广告为空，或者用户点击广告后无有效广告的情况

## 分平台逻辑及代码示例

![](../../.gitbook/assets/mini_program.jpg)

### Android端逻辑

1. 调用“/ad/openapi/v2/get\_ad”接口获取广告列表数据\(建议服务端调用\)，LinkedME可能返回多条广告；获取数据后，根据check\_install\_status字段的值确认是否需要判断安装状态，如需要则通过pkg\_name逐条判断应用是否已安装，最终获得有效广告列表，顺次显示广告；若均无效则不展示广告。
2. 用户点击广告唤起APP 当用户点击广告时，通过uri scheme唤起APP\(如果第一步展示了未安装的APP广告，跳转到h5地址，建议同时后台下载apk包\)。
3. 调用“/ad/openapi/v2/record\_status”接口向LinkedME服务器发送广告行为通知。

### Android端代码示例

```java
// adInfoArrayList对象为调用get_ad接口获取的广告列表，AdInfo为广告实体
ArrayList<AdInfo> adInfoArrayList = new ArrayList<>();
for (int i = 0; i < adInfoArrayList.size(); i++) {
    AdInfo adInfo = adInfoArrayList.get(i);
    // 不同的广告类型走不同的处理逻辑
    // 广告类型：0:普通广告（默认）2:微信小程序广告
    if (adInfo.getAdType() == 2) {
        // 2:小程序广告，此广告可直接展示
        // 展示广告后，调用track的imp_urls字段中的链接上报状态
        break;
    }else if (adInfo.getAdType() == 0){
        // 0:普通广告
        // 判断应用是否需要检查App的安装状态
        if ("1".equals(adInfo.getCheckInstallStatus())) {
            // 需要检查安装状态
            if (isAppInstalled(this, adInfo.getPackageName())) {
                // 根据返回的包名检查广告主App是否安装，安装则此广告可展示
                // 展示广告后，调用track的imp_urls字段中的链接上报状态
                break;
            }
        } else {
            // 不需要检查安装状态
            // 此广告可展示
            // 展示广告后，调用track的imp_urls字段中的链接上报状态
            break;
        }
    }
}

//广告点击
ad_click.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view){
    // 点击广告后，调用track的deeplink_urls字段中的链接上报状态
    String uriString = "lkmedemo://?click_id=G4LCXAjn7";
    String packageName = "com.microquation.linkedme";
    String h5_url = "http://www.linkedme.cc";
    String apk_url = "https://github.com/WFC-LinkedME/LinkedME-Android-Deep-Linking-Demo/blob/master/LinkedME-Android-Demo.apk?raw=true";
    String user_name = "gh_b83a75769dc2";
    String path = " pages/details?id=3782&from=linkedme&imei=<Android手机imei号>&idfa=<iOS手机idfa号> ";

    try {
        if(adInfo.getAdType() == 2)){
            // 2:微信小程序广告
            WXLaunchMiniProgram.Req req = new WXLaunchMiniProgram.Req();
            req.userName = user_name; // 微信小程序原始id
            req.path = path;  //拉起小程序页面的可带参路径，不填默认拉起小程序首页
            req.miniprogramType = WXLaunchMiniProgram.Req.MINIPTOGRAM_TYPE_RELEASE;// 打开正式版
            // api为IWXAPI对象，参考移动应用拉起小程序Android端代码示例：https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=21526646437Y6nEC&token=&lang=zh_CN
            api.sendReq(req);
            // 拉活APP后，调用track的active_urls字段中的链接上报状态
        }else if(adInfo.getAdType() == 0)){
            Intent intent = Intent.parseUri(uriString, Intent.URI_INTENT_SCHEME);
            intent.setPackage(packageName);
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            ResolveInfo resolveInfo = DemoActivity.this.getPackageManager().resolveActivity(intent, PackageManager.MATCH_DEFAULT_ONLY);
            if (resolveInfo != null) {
                startActivity(intent);
                // 拉活APP后，调用track的active_urls字段中的链接上报状态
                } else {
                openAppWithPN(packageName, uriString, h5_url, apk_url);
            }
        }
    } catch (URISyntaxException ignore) {
        openAppWithPN(packageName, uriString, h5_url, apk_url);
        }
    }
});

/**
* 通过包名唤起APP
* @param packageName 包名
* @param uriString uri scheme
* @param h5_url h5链接
* @param apk_url apk下载地址
*/
private void openAppWithPN(String packageName, String uriString, String h5_url, String apk_url) {
    //如果通过uri scheme没有唤起APP，则尝试包名唤起APP
    Intent resolveIntent = DemoActivity.this.getPackageManager().getLaunchIntentForPackage(packageName);
    // 启动目标应用
    if (resolveIntent != null) {
        resolveIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        resolveIntent.setData(Uri.parse(uriString));
        DemoActivity.this.startActivity(resolveIntent);
        // 拉活APP后，调用track的active_urls字段中的链接上报状态
    } else {
        // 建议未唤起APP打开h5页面的同时下载apk，引导用户安装
        if (!TextUtils.isEmpty(h5_url)) {
            // 未拉活APP后，调用track的openstore_urls字段中的链接上报状态
            openH5Url(h5_url);
        }
        if (!TextUtils.isEmpty(apk_url)) {
            // 应用内开启服务下载apk文件或通过外部浏览器下载apk文件
            // 未拉活APP后，调用track的download_urls字段中的链接上报状态
        }
    }
}

/**
* 打开h5链接
* @param h5_url h5链接
*/
private void openH5Url(String h5_url) {
    // 应用内WebView打开h5页面或在外部浏览器中打开h5页面
    // 若在应用内WebView中打开h5地址，h5地址可能是一个引导用户下载apk的地址，需要注意处理点击h5页面内apk下载链接的情况；
    // 若在外置浏览器中打开则无需处理。
}

/**
 * 根据包名判断是否安装了APP
 *
 * @param pkgName 包名
 * @return true:安装 false:未安装
 */
public static boolean isAppInstalled(Context context, String pkgName) {
   PackageInfo packageInfo = null;
   try {
       packageInfo = context.getPackageManager().getPackageInfo(pkgName, 0);
   } catch (PackageManager.NameNotFoundException ignore) {
       packageInfo = null;
   }
   if (packageInfo == null) {
       return false;
   } else {
       return true;
   }
}
```

### iOS端逻辑

1. 从SSP后台的应用管理中拿到APP对应的广告主Url Scheme，并写入配置文件中（为了判断广告主的APP是否安装）
2. （**方案一**）调用“/ad/openapi/v2/get\_ad”接口获取广告列表数据\(建议服务端调用\)，LinkedME可能返回多条广告；获取数据后，根据check\_install\_status字段的值确认是否需要判断安装状态，如需要则通过scheme逐条判断应用是否已安装，最终获得有效广告列表，顺次显示广告；若均无效则不展示广告。 （**方案二**）媒体方可实时或周期性扫描白名单，在调用“/ad/openapi/v2/get\_ad”接口请求广告时将白名单中已经安装的APP list通过ad\_app\_install\_list字段传给LinkActive，LinkActive根据返回的APP列表来返回已安装的广告主的广告；媒体根据返回的广告列表顺次展示广告。
3. 用户点击广告，通过scheme唤起APP\(如果第二步展示了未安装的APP广告，点击后跳转到AppStore；\)
4. 调用“/ad/openapi/v2/record\_status”接口向LinkedME服务器发送广告行为通知。

### iOS端代码示例

iOS广告主app安装状态获取（通过url scheme白名单）

```swift
- (void) appInstallStatus:(void (^)(NSDictionary* dict))block{
    NSMutableDictionary * appStatus = [[NSMutableDictionary alloc]init];
    NSArray* ret = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"LSApplicationQueriesSchemes"];
    if (ret) {
        for (NSString * scheme in ret) {
            if ([scheme isKindOfClass:[NSString class]]) {
                [[UIApplication sharedApplication] openURL:[NSURL URLWithString:scheme] options:@{} completionHandler:^(BOOL success) {
                    [appStatus setValue:[NSString stringWithFormat:@"%d",success] forKey:scheme];
                    if (appStatus.count == ret.count) {
                        block(appStatus);
                    }
                }];
            }
        }
    }
}

调用
  [self appInstallStatus:^(NSDictionary * dict) {
        NSLog(@"%@",dict);
    }];
```

```text
//check_install_status字段使用示例代码
// 检查应用是否安装，以判断是否需要显示广告
if (!OWS_MAN.checkInstallStatus) {
    [[UIApplication sharedApplication] openURL:[NSURL URLWithString:dict[@"seatbid"][index][@"ad_content"][@"download_url"]] options:@{} completionHandler:^(BOOL success) {
    if (success) {
//打开appstore
        [self extractReport:dict indexPath:index status:@"openstore_urls"];
    }else{
        NSLog(@"操作失败!");
    }
}];
}
//这里可以做打开AppStore动作
block(NO);
/*
通过Url Schemes唤起App
@param scheme url schemes
@param adid appid
*/
- (void)openScheme:(NSString *)scheme AndAdid:(NSUInteger)adid{
UIApplication *application = [UIApplication sharedApplication];
NSURL *URL = [NSURL URLWithString:scheme];
    if ([application respondsToSelector:@selector(openURL:options:completionHandler:)]) {
        [application openURL:URL options:@{}
        completionHandler:^(BOOL success) {
            [self recordStatus:[NSString stringWithFormat:@"%d",success] withAdid:adid];
                if (success) {
//拉活成功,此处通知服务器唤起了APP，status为13
                }else{
//拉活失败,此处通知服务器未唤起APP，status为14
//这里可以做打开h5页和打开AppStore动作
                }
            }];
            } else {
                BOOL success = [application openURL:URL];
//App拉活失败,此处通知服务器未唤起APP，status为14
//判断是否在app内打开AppStore
                [self showStoreProductWithAdid:adid];
                }
}
```

### 发送回调通知说明

1）展示广告，status值为11  
2）点击广告，status值为12  
3）唤起APP，status值为13  
4）点击广告，没有唤起APP，status值为14  
5）点击广告，去下载APP，status值为15

## 接口说明

### 获取广告接口

获取广告接口

#### 接口协议

* 接口地址：[http://a.lkme.cc/ad/openapi/v2/get\_ad](http://a.lkme.cc/ad/openapi/v2/get_ad)
* 协议：http
* 方法：POST
* 请求头：ContentType:application/json

#### 请求参数

* 字段类型说明  

  required : 必须存在且不为空；    

  recommended : 推荐字段，通常也不为空；  

  optional : 可选字段 \(默认类型\)

**BidRequest对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| search\_id | String | recommended | 媒体方每次请求的唯一id，用于追踪请求，媒体方生成 |
| device | Object | required | 设备信息 |
| app | Object | required | 媒体APP信息 |
| user | Object | optional | 用户信息 |
| site | Object | optional | 媒体站点信息 |
| tmax | int | optional | 超时时间，单位为毫秒值 |
| test | int | optional | 值为0或1 0：live 1：test 默认为0 |

**device对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| imei | String | required （Android） | 安卓设备的imei原值，如果是Android系统则必填 |
| android\_id | String | optional | 安卓设备的AndroidID原值 |
| idfa | String | required （iOS） | 苹果设备的idfa原值，如果是iOS系统则必填 |
| os | String | required | 设备系统 Android或者iOS |
| osv | String | optional | 操作系统版本 |
| mac | String | recommended | 设备的mac地址 |
| w | int | optional | 屏幕宽：像素 |
| h | int | optional | 屏幕高：像素 |
| ppi | int | optional | 每英寸上的像素数 |
| make | String | optional | 设备厂商 |
| model | String | optional | 设备型号 |
| ua | String | recommended | 浏览器的User Agent |
| ip | String | recommended | 用户的公网ip地址 |
| carrier | String | optional | 运营商 0：unknown 1：中国移动 2：中国联通 3：中国电信 |
| connectiontype | int | optional | 联网方式 0：未知 1：Ethernet 2：wifi 3：未知蜂窝网络 4：2G 网络 5：3G 网络 6：4G网络 |
| geo | Object | optional | 地理位置信息 |
| ad\_app\_install\_list | Object | optional | 安装了广告主app的用户才请求广告 Android: 广告主app包名列表, 用';'分号分隔 iOS: 白名单里的scheme列表, 用';'分号分隔 |

**app对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| linkedme\_key | String | required | 媒体APP标识，由LinkedME提供 |
| ad\_position\_id | String | required | 媒体广告位标识，由LinkedME提供 |
| app\_name | String | required | 媒体APP名称 |
| ver | String | optional | 媒体APP版本号 |
| wechat\_ver | String | optional | 微信APP版本号，未安装则传空字符串，若广告位包含微信小程序唤起则该字段值必传，否则无法返回小程序广告 |

**user对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| id | String | optional | 媒体给用户的标识 |
| age | String | optional | 出生年月日，如：19960101 |
| gender | String | optional | 性别 M：表示男性 F：表示女性 O：未知 不填充表示未知 |
| user\_tag | String | optional | 用户的兴趣，用逗号分隔 |

**site对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| page | String | optional | 页面url |
| ref | String | optional | 来源url |
| keywords | String | optional | 页面内容关键字，以逗号分隔 |

**geo对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| lon | double | optional | 经度 取值范围-180.000000~+180.000000， 负值表示西经 |
| lat | double | optional | 纬度 取值范围-90.000000~+90.000000 负值表示南纬 |

#### 返回结果

**bidresponse对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| search\_id |  | recommended | 请求里的search\_id，用于追踪日志 |
| seatbid | bid对象Array | required | 返回广告数组 |
| track | Object对象 | required | 上报链接 |
| nurl | String | optional | 如果参与竞价，竞价成功的回调接口 |

**bid对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| check\_install\_status | int | required | 是否判断该广告主的app安装状态， 如果判断，安装了才展示广告； 如果不判断，直接展示广告； 0：不判断 1：判断 |
| price | float | required | 出价，单位为分 |
| chargetype | String | required | 计价类型 cpm cpc cpa |
| adid | String | required | 广告id |
| cid | String | required | 创意id |
| ad\_content | Object | required | 创意内容 |
| ad\_type | int | required | 广告类型 0:普通广告（默认） 1:全部广告类型（实际不会返回该类型） 2:微信小程序广告 |

**ad\_content对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| deeplink | String | required | 唤起App的scheme |
| img | String Array | optional | 图片地址；数组，可以返回多张图片 |
| h5\_url | String | optional | 落地页地址 |
| download\_url | String | optional | iOS是AppStore地址 Android是apk下载地址 |
| pkg\_name | String | required\(Android\) | 包名,如果是Android系统则必填 |
| title | String | optional | 标题 |
| sub\_title | String | optional | 描述 |
| content | String | optional | 保留 |
| app\_appid | String | required | 微信小程序绑定的应用的appid |
| mini\_appid | String | required | 微信小程序appid |
| user\_name | String | required | 微信小程序原始id |
| path | String | required | 拉起小程序页面的可带参路径（例：pages/details?id=3782&from=linkedme&imei=&idfa=） |

**track对象**

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| imp\_urls | String Array | required | 展示广告 |
| deeplink\_urls | String Array | required | 点击scheme上报（安装的情况） |
| active\_urls | String Array | optional | 拉活上报 |
| not\_active\_urls | String Array | optional | iOS点击了，但是没有拉活 |
| openstore\_urls | String Array | optional | 进入应用商店 |
| download\_urls | String Array | optional | 开始下载上报 |
| download\_end\_urls | String Array | optional | 下载完成上报 |
| install\_end\_urls | String Array | optional | 安装完成上报 |

#### 请求示例

```java
{
    search_id：“a8c943717c19c5b2”,
    device:{
        imei:”869611020101188”,
        android_id:”d01f6f17f9dfcf52”,
        idfa:“”,
        mac:”c0:9f:05:bf:b0:98”,
        ua:”xx”,
        ip:”117.135.233.101”,
        os:”android”,
        osv:”6.0”,
        carrier:”中国电信”,
        connectiontype:0,
        w:800,
        h:1280,
        ppi:285,
        make:”Samsung”,
        model:”SM-N9100”,
        geo:{
            lat:210.441812
            lon:41.403381
            }
           },
        app:{
            linkedme_key:”4c6d903b4b44eab7c990e0ce6f9c1e78”
            ad_position_id:”1234_1”
            ver:”5.8.8.1”
        },
        user:{
            id:”xx”,    //用户标识
            age:”19960101”,    //出生年月日
            gender:”M”,    //性别
            user_tag:”food,movie,shopping”    //兴趣，用逗号分隔
        },
        site:{
            page:”xx”,    //页面url
            ref:”xx”,    //来源url
            keywords:”搞笑段子”    //站点的关键字，用逗号分隔
        },
        test:1
}
```

#### 返回示例

```java
{
    search_id:”a8c943717c19c5b2”    //请求里的search_id
    rid:”1a599cf113e60881fbb337d06ef107aa”,    //linkedme生成的，用于追踪日志
    seatbid:[{
        check_install_status:1,    //是否需要判断安装状态
        price:5.0,
        chargetype:”cpm”,
        adid:”11102_0”,    //广告id
        cid:128    //创意id
        ad_content:{
            deeplink:”sinaweibo://detail?mblogid=mid&luicode=10000360&lfid=Linkedme”,
            img:[“http://img.lkme.cc/xx.png”,”http://img.lkme.cc/xxx.png”],
            h5_url:”http://m.weibo.cn/status/mid”,
            download_url:”http://www.weibo.com/xx.apk”,
            pkg_name:”com.sina.weibo”,
            title:”xx”,
            sub_title:”xxx”,
            content:””
        }}],
     track:{
            imp_urls:[“http://xx”],//曝光上报
            deeplink_urls:[“http://xx”],//点击上报
            active_urls:[“http://xx”],//拉活上报
            not_active_urls:[“http://xx”],//未拉活上报
            download_urls:[“http://xx”],//下载上报
            openstore_urls:[“http://xx”]//openstore上报
        },
    nurl:”“http://xx””
}
```

### 广告行为上报接口

当广告被曝光、点击、拉活、下载和安装时，调用上报接口。  
该接口不需要单独对接 , 在getAd接口返回的内容中已经有对应的链接了\(track对象中\) , 媒体方将对其中的{ADID} {CID} {TIMESTAMP} 替换之后,直接上报就可以。

#### 接口协议

* 接口地址：[http://a.lkme.cc/ad/openapi/v2/record\_status](http://a.lkme.cc/ad/openapi/v2/record_status)
* 协议：http
* 方法: `GET`

#### 请求参数

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| search\_id | String | required | 请求广告接口里的search\_id |
| bidid | String | required | 返回广告结果里的bidid |
| timestamp | long | required | 广告行为的发生时间，即当前时间的毫秒数，如1521528544 |
| imei | String | required （Android） | 安卓设备的imei原值，如果是Android系统则必填 |
| android\_id | String | optional | 安卓设备的AndroidID原值 |
| idfa | String | required （iOS） | 苹果设备的idfa原值，如果是iOS系统则必填 |
| os | String | required | 设备系统 Android或者iOS |
| linkedme\_key | String | required | 媒体APP标识，由LinkedME提供 |
| ad\_position\_id | String | required | 媒体广告位标识，由LinkedME提供 |
| adid | String | required | 广告id |
| cid | String | required | 创意id |
| status | int | required | 上报类型 11  曝光 12  点击深度链接 13  拉活 14  iOS系统下，点击广告，但是没有唤起APP 15  开始下载 16  下载完成 17  安装完成  |
| test | int | optional | 值为0或1 0：live 1：test 默认为0 |

### 调用成功接口

如果返回的广告参与竞价，广告竞价成功后回调通知。

#### 接口协议

* 接口地址：[http://a.lkme.cc/ad/openapi/v2/winnotice](http://a.lkme.cc/ad/openapi/v2/winnotice)
* 协议：http
* 方法: `GET`

#### 请求参数

| 字段 | 类型 | 是否必填 | 描述 |
| :--- | :--- | :--- | :--- |
| search\_id | String | required | 请求广告接口里的search\_id |
| bidid | String | required | 返回广告结果里的bidid |
| price | float | required | 竞价成功价格（单位为分） |

## 其他注意事项

**1，接入准备** 接入前需要先获取linkedme\_key和ad\_position，这两个字段在接口调用时来标记当前app及广告位。 请先[申请LinkActive后台权限](https://www.linkedme.cc/linkactive.html)

**2，联调需要准备哪些内容** 联调请准备您测试设备的设备ID（Android平台为imei，iOS平台为idfa）并提供广告位的尺寸标注示例图；联调时间请与LinkedME协商。

**3，点击是否就是唤起**

Android平台，要求根据包名检测APP的安装状态，展示广告则一定是APP已安装，点击后一定会唤起APP。注意：唤起一定要用scheme，而不要用包名唤起。

iOS平台，需要在配置文件里加入广告主的Url Schemes，才能判断广告主的APP是否安装，如果媒体方加入了配置，并且写了判断是否安装的逻辑，那么展示的广告一定是已安装的，点击一定能唤起APP。

**4，广告列表的顺序** LinkActive平台返回的是多个广告，并且广告是有一定优先级的，所以不能对广告列表的顺序进行更改。

**5，建议客户端的回调接口由服务端返回，方便后期优化升级** 如果回调接口写死在客户端，每次改动都需要客户端发版。并且建议API的调用添加开关。

**6，建议同时请求LinkActive的广告和自己的广告，避免降低广告的填充率** LinkActive的广告会出现两种情况：1. 直接返回的是空，没有广告；2.返回广告，但是广告主的APP用户没有安装，也不会展示；在两种情况下可以展示自己的广告，以免降低媒体方的广告填充率。

