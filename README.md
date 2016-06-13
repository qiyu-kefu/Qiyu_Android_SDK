# 网易七鱼 Android SDK 开发指南

## 简介

网易七鱼 Android SDK 是一个 Android 端客服系统访客解决方案，既包含了客服聊天逻辑管理，也提供了聊天界面，开发者可方便的将客服功能集成到自己的 APP 中。

## 快速集成

只需简单 4 步，即可将客服功能加入你的 APP（使用 Android Studio）：

1. 下载 SDK。

2. 解压缩 SDK，将得到的 unicorn 文件夹作为库工程模块导入到你的工程中，并添加模块依赖。
> 如果你使用的 IDE 是 Eclipse，还需要将 AndroidManifest 文件中的内容拷贝到你的主工程的 manifest 文件中，将 assets 文件夹的内容拷贝你的主工程的 assets 目录中。

3. 在你的 Application 类的 `onCreate` 函数中，加入以下初始化代码：

	```java
    public class YourApplication extends Application {
   	public void onCreate() {
   		// ... your codes
   		Unicorn.init(this, "appKey", options(), new UnicornImageLoader()); // appKey 可以在七鱼管理系统-\>设置-\>APP接入 页面找到
   		// ... your codes
   	}

   	// 如果返回值为null，则全部使用默认参数。
   	private YSFOptions options() {
   		YSFOptions options = new YSFOptions();
           options.statusBarNotificationConfig = new StatusBarNotificationConfig();
           options.savePowerConfig = new SavePowerConfig();
           return options;
   }
   ```
   
   上面代码中，UnicornImageLoader 可根据你 APP 中图片加载模块做自定义实现，以免 SDK 中引入第三方图片管理库后造成与 APP 的冲突或者浪费。在 demo 中，实现了依赖于 UniversalImageLoader 的 UILImageLoader。其代码以及依赖于 fresco 的实现代码可参考 [图片加载](#图片加载) 一节。
   
4. 在你的 APP 的合适页面添加客服入口按钮，并在响应函数中加入如下代码：

	```java
	String title = "聊天窗口的标题";
	// 设置访客来源，标识访客是从哪个页面发起咨询的，用于客服了解用户是从什么页面进入三个参数分别为来源页面的url，来源页面标题，来源页面额外信息（可自由定义）
	// 设置来源后，在客服会话界面的"用户资料"栏的页面项，可以看到这里设置的值。
	ConsultSource source = new ConsultSource(sourceUrl, sourceTitle, "custom information string");
	// 请注意： 调用该接口前，应先检查Unicorn.isServiceAvailable(), 如果返回为false，该接口不会有任何动作
	Unicorn.openServiceActivity(context, // 上下文
                            title, // 聊天窗口的标题
                            source // 咨询的发起来源，包括发起咨询的url，title，描述信息等
                             );
	```

在打开的页面中，用户就可以咨询客服了。

## SDK包具体内容

```
sdk
├── libs
│   ├── qiyu-sdk-2.0.0.jar
│   └── android-support-v4.jar
├── res
│   └── ***
└── assets
    └── ***
```

上面文件中，qiyu-sdk-2.0.0.jar 是网易七鱼的 SDK 包 (版本号可能有所不同)，res 和 assets 为 SDK 所依赖的资源文件。

android-support-v4.jar 为工程依赖的外部库，所需最低版本为22.2.0。如果你的 APP 也依赖了这个 jar 包，可以将你的工程中的依赖移除，或者将这个库移动到一个更基础的库工程中做依赖。如果是使用 Android Studio 接入，SDK 工程的 build.gradle 文件已经添加了依赖，无需理会这个文件。

## 混淆配置

如果你的 apk 最终会经过代码混淆，请在 proguard 配置文件中加入以下代码:

```
-dontwarn com.qiyukf.**
-keep class com.qiyukf.** {*;}
```

## 初始化

网易七鱼 SDK 需要接收消息推送，因此有一个后台进程，进程名为 "packageName:core"。我们知道，Application 的 onCreate 在各个进程中都会被调用，包括 UI 主进程和七鱼的推送进程。在实现 Application 的 onCreate 时，如果需要在 onCreate 中调用除 init 接口外的其他接口，应先判断当前所属进程，并只有在当前是 UI 进程时才调用。SDK 的 init 接口无需做额外判断，SDK 会自动识别是否需要初始化。另外，要注意不要在主进程外的其他进程中再调用 Unicorn 提供的接口（`init` 除外）。判断当前进程是否是在主进程的代码示例如下：
   
```java
public static boolean inMainProcess(Context context) {
    String packageName = context.getPackageName();
    String processName = SystemUtil.getProcessName(context);
    return packageName.equals(processName);
}
   
/**
 * 获取当前进程名
 * @param context
 * @return 进程名
 */
public static final String getProcessName(Context context) {
    String processName = null;
   
    // ActivityManager
    ActivityManager am = ((ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE));
   
    while (true) {
        for (ActivityManager.RunningAppProcessInfo info : am.getRunningAppProcesses()) {
            if (info.pid == android.os.Process.myPid()) {
                processName = info.processName;
                break;
            }
        }
   
        // go home
        if (!TextUtils.isEmpty(processName)) {
            return processName;
        }
  
        // take a rest and again
        try {
            Thread.sleep(100L);
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
    }
}
```
   
初始化包括两个部分，一是本地数据的初始化，二是从七鱼的服务器获取一些配置信息操作。本地数据的初始化是同步的，主要是检查本地是否已经有可以聊天的账号，如果有了，还会做一些缓存数据的初始化操作。如果没有，才会进入到第二部分，去七鱼服务器获取聊天账号。第二部分是异步操作，会在后台自动完成。由于网络操作具有一些不确定性，调用 init 返回 true 之后，并不一定就可以立即和客服聊天了。我们建议，在打开客服聊天窗口前，先调用一次 `Unicorn.isServiceAvailable()` 检查是否已经有了聊天账号，如果返回 false， 则不应该进入聊天界面。当然，由于获取聊天账号的行为只会在安装 APP 后第一次初始化时才会有，在日常的使用中，这个接口都是会返回 true 的。

## 新消息提醒

当用户不处在聊天界面时，收到客服的消息，APP 应当在通知栏或者聊天入口给出提醒。

通知栏提醒可以显示最近一条消息的内容，并提供给用户快速进入 APP 的入口。要打开通知栏提醒功能，只需给 `YSFOptions` 的 `statusBarNotificationConfig` 域赋予非 null 值即可。同时，通过定制该域的各配置项，还能实现提醒开关，免打扰等功能。

要实现点击通知栏提醒直接跳转到会话窗口的功能，需要设置 `StatusBarNotificationConfig` 的 `notificationEntrance`，并在对应的 Activity 里添加处理。如果没有设置 `notificationEntrance`，则是在 `AndroidManifest` 中设置的入口 Activity 中处理。示例代码如下：

```java
Intent intent = getIntent();
if (intent.hasExtra(NimIntent.EXTRA_NOTIFY_CONTENT)) {
    // 打开客服窗口
     Unicorn.openServiceActivity(context, title, source);
    // 最好将intent清掉，以免从堆栈恢复时又打开客服窗口
    setIntent(new Intent());
}
```

在聊天入口的地方，APP 可以给出是否有未读消息，以及未读数的提示。APP 可以通过添加一下监听来跟踪未读数变化，更新界面，反馈给用户：

```java
// 添加未读数变化监听，add 为 true 是添加，为 false 是撤销监听。退出界面时，必须撤销，以免造成资源泄露
private UnreadCountChangeListener listener = new UnreadCountChangeListener() { // 声明一个成员变量
        @Override
        public void onUnreadCountChange(int count) {
            // 在此更新界面, count 为当前未读数，
            // 也可以用 Unicorn.getUnreadCount() 获取总的未读数
        }
        
private void addUnreadCountChangeListener(boolean add) {
    Unicorn.addUnreadCountChangeListener(listener, add);
}
```

默认情况下，只有访客在聊天界面时，才不会有通知栏提醒，其他界面以及 APP 在后台时，都会有消息提醒。如果当 APP 在前台时，不需要通知栏提醒新消息，可以调用`Unicorn.toggleNotification(false)`关闭消息提醒，然后在 APP 退到后台时，调用`Unicorn.toggleNotification(true)`重新打开。

## 定制界面

### 客服窗口UI自定义

为了咨询客服窗口的界面风格能与集成七鱼 SDK 的 APP 能够整体统一，七鱼 SDK 提供了简洁的 UI 自定义配置选项。
配置选项接口名为 UICustomization，配置参数放在 `YSFOptions` 的 `uiCustomization` 变量中，开发者可在初始化 SDK 或者在运行时任意时候修改配置，当需要与 SDK 提供的默认界面不一样表现的地方，就修改对应的项，否则不赋值即可，界面会保留默认表现。修改各设置项后，都需要等到下次进入会话界面才会看到相应的更改。

各配置项说明如下：

| 参数名 | 类型 | 参数说明 | 取值说明 |
| :-----| :-----| :-----| :-----|
| msgBackgroundUri | String | 客服消息窗口背景图片设置 | uri（支持格式见表后）|
| msgBackgroundColor | int | 客服消息窗口颜色。如果同时设置 uri 和颜色，优先使用 uri | 32 位颜色值 |
| msgListViewDividerHeight | int | 消息列表消息项间距 | 单位为 pixel |
| hideLeftAvatar | boolean | 是否隐藏左侧(客服消息)头像 | 默认为 false，不隐藏 |
| hideRightAvatar | boolean | 是否隐藏右侧(访客消息)头像 | 默认为 false，不隐藏 |
| avatarShape | int | 头像形状风格 | 0为圆形头像，1为方形头像 |
| leftAvatar | String | 左侧 （客服消息）头像图片 uri | uri（支持格式见表后） |
| rightAvatar | String | 右侧 （访客消息）头像图片 uri | uri（支持格式见表后） |
| tipsTextColor | int | 提示类消息的字体颜色（包括分配客服消息，消息时间标签等）| 32 位颜色值 |
| tipsTextSize | float | 提示类消息的字体大小（包括分配客服消息，消息时间标签等）| 单位为 sp |
| msgItemBackgroundLeft | int | 左边消息项背景, 最好是 selector，同时影响文本和语音消息。 | drawable resId |
| msgItemBackgroundRight | int | 右边消息项背景, 最好是 selector，同时影响文本和语音消息。 | drawable resId |
| audioMsgAnimationLeft | int | 左侧语音消息播放时候的动画 animation-list,没有播放时显示最后一帧 | drawable resId |
| audioMsgAnimationRight | int | 右侧语音消息播放时候的动画 animation-list,没有播放时显示最后一帧 | drawable resId |
| textMsgColorLeft | int | 左侧文本消息颜色 | 32 位颜色值 |
| hyperLinkColorLeft | int | 左侧文本消息中超链接的颜色 | 32 位颜色值 |
| textMsgColorRight | int | 右侧文本消息颜色 | 32 位颜色值 |
| hyperLinkColorRight | int | 右侧文本消息中超链接的颜色 | 32 位颜色值 |
| textMsgSize | float | 文本消息字体大小 | 单位为sp |
| inputTextColor | int | 底部消息输入框的字体颜色 | 32 位颜色值 |
| inputTextSize | float | 底部消息输入框的字体大小 | 单位为sp |
| topTipBarBackgroundColor | int | 顶部提示栏(没有客服，排队状态等)背景色 | 32 位颜色值 |
| topTipBarTextColor | int | 顶部提示栏文字颜色 | 32 位颜色值 |
| topTipBarTextSize | float | 顶部提示栏文字大小 | 单位为 sp |
| titleBackgroundResId | int | 标题栏背景图 | drawable resId|
| titleBarStyle | int | 标题栏风格，影响标题和标题栏上按钮的颜色 | 目前支持： 0浅色系，1深色系|
| buttonBackgroundColorList | int | 发送，选择，预览等按钮的颜色 | ColorStateList，可参考 SDK 的 ysf\_button\_color\_state\_list |
| buttonTextColor | int | 发送，选择，预览等按钮的文字颜色 | 32 位颜色值 |

> 图片 uri 支持的格式由开发者根据自己使用的图片加载框架定义。但必须要支持 file://， http:// 和 https:// 这3种。

### 其他界面

SDK 的所有资源文件名均以 nim 或者 ysf 作为前缀，colors 和 strings 中的常量也以 nim 或 ysf 为前缀，以避免污染开发者的资源名字空间。

聊天界面的根 layout 文件为 ysf\_message\_fragment.xml, 通过修改该文件，以及其引用的各子 layout 文件，可以修改聊天界面的框架布局。通过修改其中引用的素材资源，可以修改界面的上各图标，字体，背景等。

SDK 自带的会话界面为 ServiceMessageActivity, 其 theme 为 ysf_window_theme, 如果需要修改标题栏样式，可以修改该 style。 该 style 位于 ysf_styles.xml 中。

如果修改 SDK 中的资源文件，以后升级 SDK 时要注意重新替换，以免又回到默认界面上。

请注意，由于 SDK 在代码中申请了 `Window.FEATURE_CUSTOM_TITLE`，该 feature 与其他 title 样式会冲突，因此修改 style 时不能再指定 `android:windowNoTitle`。如果需要使用这种样式，可参照下面以 fragment 的方式集成。

SDK 还提供了以 fragment 嵌入的方式集成会话界面，开发者可以更灵活的使用 SDK。示例代码如下：

```java
String title = "聊天窗口的标题"; // 标题
ConsultSource source = new ConsultSource(sourceUrl, sourceTitle, "custom information string"); // 访客来源信息
// 构造一个 ViewGroup，用于放置sdk的评价和人工客服按钮。该控件推荐放在标题栏右边。可以用以下两种方式:
// 1. 将 container 放到 layout 文件中
// LinearLayout sdkIconContainer = (LinearLayout)findViewById(R.id.xxx);
// 2. 动态构建，动态添加
// LinearLayout sdkIconContainer = new LinearLayout(this);
// sdkIconContainer.setOrientation(LinearLayout.HORIZONTAL);
// 构造好后，还需要将 ViewGroup 添加到你的 Activity 中
ServiceMessageFragment fragment = Unicorn.newServiceFragment(title, source, sdkIconContainer);
FragmentManager fm = getSupportFragmentManager();
FragmentTransaction transaction = fm.beginTransaction();
transaction.replace(containerId, fragment); // 将 fragment 放到对应的 containerId 中。containerId 为 ViewGroup 的 resId
try {
    transaction.commitAllowingStateLoss();
} catch (Exception e) {

}
```


## 跟踪用户访问记录

用户在使用你的 APP 时，SDK 提供了接口用于跟踪其浏览路径，用于后期用户行为分析以及主动营销。接口为：

```java
// 两个参数分别为访问内容的 uri，和该 uri 对应的简单描述
Unicorn.trackUserAccess(uri, description);
```

## 关联商户用户信息

七鱼 SDK 允许 APP 的用户以匿名方式向客户咨询，但如果 APP 希望客服知道咨询的用户的身份信息，可以通过 SDK 提供的 `setUserInfo` 接口告诉给客服。 该接口包含两个功能：
1. 关联用户账户。调用过该接口后，客服即可知道当前用户是谁，并可以调看该用户之前发生过的访问记录。通过该接口，SDK还会把 APP 端相同用户 ID 的咨询记录整合在一起。如果调用 `setUserInfo` 接口前是匿名状态，那么匿名状态下的聊天记录也会被整合到新设置的这个用户下面。
2. 提供用户的详细资料。通过设置参数 `YSFUserInfo` 的 data 字段， APP 能把用户的详细信息告诉给客服，这些信息会显示在客服会话窗口的用户信息栏中。该字段具有很强的可扩展性，具体请见本节后面的描述。

为了区分不同的 APP 用户，当 APP 端用户注销后，需要先调用 `setUserInfo(null)`，告诉 SDK，SDK 此时会关闭前一个用户的聊天记录，并重新分配一个新的聊天账号，以和之前的用户区分。

**注意，该接口不允许直接从一个账号切换到另外一个账号。**

为了方便开发者调用，这个接口没有设置回调接口，开发者也无需在用户资料发生变更后就调用该接口。开发者可在每次打开客服窗口时调用该函数，设置访客资料。SDK会缓存上次设置的资料，只有当资料发生改动时SDK才会上传，以节省流量。

访客信息为一个 json 数组，该数组中的 item 目前可包含以下字段：
- key: 数据项的名称，用于区别不同的数据。
- index: 用于排序，显示数据时数据项按index值升序排列；不设定index的数据项将排在后面；index相同或未设定的数据项将按照其在 JSON 中出现的顺序排列。
- label: 该项数据显示的名称。
- value: 该数据显示的值，类型不做限定，根据实际需要进行设定。
- href: 超链接地址。若指定该值，则该项数据将显示为超链接样式，点击后跳转到其值所指定的 URL 地址。
- hidden: 是否隐藏该item。目前仅对mobile和email有效。
 
现在，以下3个key由七鱼使用，他们的排序和标签名是固定的，不能指定index和label。
- real\_name: 用户姓名。
- mobile\_phone：用户手机号，可以隐藏。
- email：用户的邮箱账号，可以隐藏。
 
设置访客信息的示例如下：

```java
YSFUserInfo userInfo = new YSFUserInfo();
userInfo.userId="uid";
userInfo.data="[{"key":"real_name", "value":"土豪"},
                          {"key":"mobile_phone", "hidden":true},
                          {"key":"email", "value":"13800000000@163.com"},
                          {"index":0, "key":"account", "label":"账号", "value":"zhangsan" , "href":"http://example.domain/user/zhangsan"},
                          {"index":1, "key":"sex", "label":"性别", "value":"先生"},
                          {"index":5, "key":"reg_date", "label":"注册日期", "value":"2015-11-16"},
                          {"index":6, "key":"last_login", "label":"上次登录时间", "value":"2015-12-22 15:38:54"}]";
Unicorn.setUserInfo(userInfo);
```

## 自定义客服分配

在打开客服咨询窗口或者构造 `ServiceMessageFragment` 时，接口中有一个参数是 `ConsultSource`，通过该参数，我们可以对分配客服的流程做一些自定义操作。

### 指定客服（组）

在 2.0 之前的版本中，请求客服时会根据企业在管理后台上对 APP 客服分配的设置，来决定首先接入机器人还是人工客服，如果管理后台开启了机器人，则首先会接入机器人。如果管理后台也允许接入人工客服，那么用户可以通过回复 'RG' 或者 '人工客服' 一类的关键字切换到人工客服，界面上也能展现一个人工客服的入口，用户点击此入口，也能接入人工客服。

在2.0版本中，七鱼加入了访客分配的逻辑。如果APP端开启了访客分配，那么当切换到人工客服时，会首先给出一个客服分组选择的入口，用户可以自主选择某个客服或者客服分组咨询。

在2.2版本中，SDK在 `ConsultSource` 增加了指定客服或者客服组的参数，开发者可以在访客进入咨询界面前就指定好要为其服务的客服，例如从订单页面打开咨询界面时为其指定售后客服，在商品页面打开时，为其指定售前客服。在 `ConsultSource` 中，`staffId` 为客服 ID，`groupId` 为客服组 ID，其值可以在管理后台设置页面的「访客分配」页面中找到。如果这两个参数都没有设置，则按照2.0版本的逻辑分配客服，如果指定了其中一个，则只会分配指定的客服或者在指定的客服分组中分配，如果指定的客服不在线，或者指定的客服分组中没有客服在线，则会提示用户客服不在线。如果同时指定 `staffId` 和 `groupId`，以 `staffId` 为准，忽略 `groupId`。

### 汇报商品信息

在打开咨询窗口时，还可以带上用户当前正在浏览的商品或订单信息。在 `ConsultSource` 中，设置字段为 `productDetail`，其类型为 `ProductDetail`。可以设置的信息有：

|字段|意义|备注|
|:----|:----|:----|
|title|商品标题|长度限制为 100 字符，超过自动截断。|
|desc|商品详细描述信息。|长度限制为 300 字符，超过自动截断。|
|note|商品备注信息（价格，套餐等）|长度限制为 100 字符，超过自动截断。|
|picture|缩略图图片的 url。|该 url 需要没有跨域访问限制，否则在客服端会无法显示。<br> 长度限制为 1000 字符， 超长不会自动截断，但会发送失败。|
|url|商品信息详情页 url。|长度限制为 1000 字符，超长不会自动截断，但会发送失败。|
|show| 是否在访客端显示商品消息。|默认为false，即客服能看到此消息，但访客看不到，也不知道该消息已发送给客服。|

## 图片加载

图片加载基本上每个 APP 都会用到，各个 APP 也都会有自己的实现或者依赖的第三方库。为了避免 SDK 引入第三方图片库后，与 APP 自身依赖的图片库冲突，或者与 APP 用了不同的库造成浪费，从 2.0.0 版本开始，七鱼 SDK 需要由 APP 实现一个图片加载接口，并传给 YSFOptions。 下面是两个常用的第三方图片加载库的接口实现示例，开发者可以直接使用。
图片加载参数中的 uri 格式，只要是 APP 选用的图片框架支持即可。但请注意，必须支持本地文件 uri （file://）和 网络文件 uri (http:// 或者 https://) 这两种格式。

### UniversalImageLoader实现

该实现依赖 universal-image-loader 库，需要在工程的 build.gradle 文件中添加依赖：

```java
compile 'com.nostra13.universalimageloader:universal-image-loader:1.9.5'
```

如果使用的 IDE 是 Eclipse，则需要在 libs 中添加 universal-image-loader 的 jar 包。

```java
public class UILImageLoader implements UnicornImageLoader {
    private static final String TAG = "UILImageLoader";

    @Override
    public Bitmap loadImageSync(String uri, int width, int height) {
        DisplayImageOptions options = new DisplayImageOptions.Builder()
                .cacheInMemory(true)
                .cacheOnDisk(false)
                .bitmapConfig(Bitmap.Config.RGB_565)
                .build();

        // check cache
        boolean cached = MemoryCacheUtils.findCachedBitmapsForImageUri(uri, ImageLoader.getInstance().getMemoryCache()).size() > 0
                || DiskCacheUtils.findInCache(uri, ImageLoader.getInstance().getDiskCache()) != null;
        if (cached) {
            Bitmap bitmap = ImageLoader.getInstance().loadImageSync(uri, new ImageSize(width, height), options);
            if (bitmap == null) {
                Log.e(TAG, "load cached image failed, uri =" + uri);
            }
            return bitmap;
        }

        return null;
    }

    @Override
    public void loadImage(String uri, int width, int height, final ImageLoaderListener listener) {
        DisplayImageOptions options = new DisplayImageOptions.Builder()
                .cacheInMemory(true)
                .cacheOnDisk(false)
                .bitmapConfig(Bitmap.Config.RGB_565)
                .build();

        ImageLoader.getInstance().loadImage(uri, new ImageSize(width, height), options, new SimpleImageLoadingListener() {
            @Override
            public void onLoadingComplete(String imageUri, View view, Bitmap loadedImage) {
                super.onLoadingComplete(imageUri, view, loadedImage);
                if (listener != null) {
                    listener.onLoadComplete(loadedImage);
                }
            }

            @Override
            public void onLoadingFailed(String imageUri, View view, FailReason failReason) {
                super.onLoadingFailed(imageUri, view, failReason);
                if (listener != null) {
                    listener.onLoadFailed(failReason.getCause());
                }
            }
        });
    }
}
```

### fresco实现

fresco 库本身提供了一整套图片缓存和加载的功能，由于 SDK 中部分 ImageView 需要做自定义的绘制，因此只会用到其下载，解码和缓存的逻辑。
该实现依赖 fresco 库，需要在工程的 build.gradle 文件中添加依赖：

```java
compile 'com.facebook.fresco:fresco:0.9.0'
```

如果使用的 IDE 是 Eclipse，则需要在 libs 中添加 fresco 的 jar 包。

```java
public class FrescoImageLoader implements UnicornImageLoader {
    @Override
    public Bitmap loadImageSync(String uri, int width, int height) {
        return null;
    }

    @Override
    public void loadImage(String uri, int width, int height, final ImageLoaderListener listener) {
        ImageRequestBuilder builder = ImageRequestBuilder.newBuilderWithSource(Uri.parse(uri));
        if (width > 0 && height > 0) {
            builder.setResizeOptions(new ResizeOptions(width, height));
        }
        ImageRequest imageRequest = builder.build();

        ImagePipeline imagePipeline = Fresco.getImagePipeline();
        DataSource<CloseableReference<CloseableImage>> dataSource = imagePipeline.fetchDecodedImage(imageRequest, context);

        dataSource.subscribe(new BaseBitmapDataSubscriber() {
            @Override
            public void onNewResultImpl(@Nullable Bitmap bitmap) {
                if (listener != null && bitmap != null) {
                    listener.onLoadComplete(bitmap.copy(Bitmap.Config.RGB_565, true));
                }
           }

           @Override
           public void onFailureImpl(DataSource dataSource) {
               if (listener != null) {
                   listener.onLoadFailed(dataSource.getFailureCause());
               }
          }
     },
     UiThreadImmediateExecutorService.getInstance());
}
```

### Picasso实现

该实现依赖 picasso 库，需要在工程的 build.gradle 文件中添加依赖：

```java
compile 'com.squareup.picasso:picasso:2.5.2'
```

如果使用的 IDE 是 Eclipse，则需要在 libs 中添加 picasso 的 jar 包。

```java
public class PicassoImageLoader implements UnicornImageLoader{

    @Nullable
    @Override
    public Bitmap loadImageSync(String uri, int width, int height) {
        return null;
    }

    @Override
    public void loadImage(String uri, int width, int height, final ImageLoaderListener listener) {
        RequestCreator requestCreator = Picasso.with(DemoCache.context).load(uri);
        if (width > 0 && height > 0) {
            requestCreator = requestCreator.resize(width, height);
        }
        requestCreator.into(new Target() {
            @Override
            public void onBitmapLoaded(Bitmap bitmap, Picasso.LoadedFrom from) {
                if (listener != null) {
                    listener.onLoadComplete(bitmap);
                }
            }

            @Override
            public void onBitmapFailed(Drawable errorDrawable) {
                if (listener != null) {
                    listener.onLoadFailed(null);
                }
            }

            @Override
            public void onPrepareLoad(Drawable placeHolderDrawable) {

            }
        });
    }
}
```

## 省电策略

七鱼 SDK 作为用户端的 IM 工具，需要能够及时收到客服回复的消息。同时，就算用户没有主动咨询过客服，客服也可以主动发起会话，进行主动营销。因此，在 2.0 版本之前，SDK一旦初始化之后，会一直保持长连接，定时与服务器交换心跳。

但是，毕竟用户咨询以及客服主动营销都是不太常用的操作，为此保持一个长连接，性价比不高。同时，我们了解到，很多客户的 APP 本身就自带了推送模块，或者继承了第三方的推送 SDK，由此就存在了两条长连接，浪费了用户的电量和流量。因此，在2.0版本中，我们针对 SDK 的长连接策略做了更改，以优化在用户没有发起咨询时的电量消耗。同时，为了保持前向兼容性，默认情况下，SDK 仍旧会保持长连接，如果开发者需要开启省电特性，需要在 `YSFOptions` 中设置 `savePowerConfig`。

在咨询客服时，SDK 会保持长连接以便及时收到推送。当一次会话结束后，用户极有可能还会继续发起咨询会话，同时客服也很有可能会主动发起会话，因此，一次会话结束后，长连接可以在继续保持一段时间(`SavePowerConfig.activeDelay`)，以便及时收到消息，同时也可以减少重新连接造成的消耗。如果在这段时间内，再也没有开始过会话，则在计时结束后即可转入省电模式。根据用户配置，省电模式分为两种状态：
1. 推送模式：用户需要在七鱼的管理后台配置推送服务器，且 `SavePowerConfig.customPush` 设置为 true。此时，如果有新消息，会自动转到后台配置的推送通道上。由于推送不受 SDK 控制，因此收到推送后 SDK 不会自动建立长连接，需要用户进入咨询界面后，才会建立长连接。开启自定义推送后，还需要设置推送的设备 ID：`SavePowerConfig.deviceIdentifier`。在七鱼推送的消息结构体中，会包含该字段。 
2. 轮询模式：SDK 每隔 `SavePowerConfig.checkInterval` 秒去服务器检测一次有没有新消息。如果有新消息，会自动建立长连接，并收取新消息，弹出通知。

### 配置推送服务器地址

客服发送消息给用户，而用户此时已经转入推送模式后，消息将被推送给开发者的服务器端，然后再由开发者推送到 APP 端。
要配置推送服务器，请使用管理员帐号登录七鱼管理后台，在「设置」 -> 「APP设置」 -> 「添加/编辑APP」中设置。

![推送服务器配置](./images/android/android_push_setting.png)

>开发者服务器收到推送请求后，应对立即返回一个 **空字符串** 告诉七鱼服务器。七鱼服务器发出 POST 请求后，如果在 10 秒内收不到响应，或者收到的响应非空，会断掉连接，并且重新发起请求，总共重试 2 次。 如果连续 10 次都推送失败，该推送服务器 url 将被暂停推送 1 个小时。

### 推送消息数据结构

当有消息需要推送时，七鱼服务器会向开发者设置的服务器地址发送推送消息，方法类型为 POST，数据格式为 JSON 。

POST 请求会包含以下参数：

|参数	|参数说明|
|:----|:----|
|nonce	|随机数字符串|
|time|	当前 UTC 时间戳，从 1970 年 1 月 1 日 0 点 0 分 0 秒开始到现在的秒数|
|checksum|SHA1(AppSecret + nonce + time), 三个参数拼接的字符串，进行SHA1哈希计算，转化成16进制字符(String，小写)|

其中，checksum 可用于校验该请求是否由七鱼服务器发出，出于安全性考虑，还应根据 time 参数校验 checksum 的有效期，超过一定时间（比如 5 分钟）的请求也应判定为非法。请确认发起请求的服务器是与标准时间同步的，比如有NTP服务。

**重要提示: 本文档中提供的所有接口均面向开发者服务器端调用，用于计算CheckSum的AppSecret开发者应妥善保管,可在应用的服务器端存储和使用，但不应存储或传递到客户端，也不应在网页等前端代码中嵌入。**

计算 checksum 的 java 示例代码如下：

 ```java
 public class QiyuPushCheckSum {

    private static final char[] HEX_DIGITS = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'};

    public static String encode(String appSecret, String nonce, String time) {
        String content = appSecret + nonce + time;
        try {
            MessageDigest messageDigest = MessageDigest.getInstance("sha1");
            messageDigest.update(content.getBytes());
            return getFormattedText(messageDigest.digest());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    private static String getFormattedText(byte[] bytes) {
        int len = bytes.length;
        StringBuilder buf = new StringBuilder(len * 2);
        for (int j = 0; j < len; j++) {
            buf.append(HEX_DIGITS[(bytes[j] >> 4) & 0x0f]);
            buf.append(HEX_DIGITS[bytes[j] & 0x0f]);
        }
        return buf.toString();
    }
}
 ```
 
POST 请求的内容为 JSON 格式，编码格式为 UTF-8，其各个字段说明如下：

|字段	|字段说明|
|:----|:----|
|content|推送的消息内容|
|time|消息的发出时间|
|messageId|消息的唯一 ID，当消息有重发时，开发者的推送服务器可依据此去重。|
|staffName|说话的客服昵称|
|deviceIdentifier|消息发送对象用户的 deviceIdentifier|
|package|消息接收对象 APP 的 package name|

## 

## 自定义响应事件

为了让用户在使用七鱼 SDK 时拥有更大的灵活性，我们将持续添加更多的事件自定义响应接口。

### URL链接点击响应

如果用户或者客服发送的文本消息中带有 URL 链接，SDK 将会解析出该链接。用户点击这个链接后，SDK 默认会打开系统的浏览器，并访问这个 URL。同时 SDK 提供了一个配置选项，允许应用自己处理这个点击事件，用于在应用的内置浏览器中打开链接，以及做钓鱼网址过滤等等场景。

设置自定义的链接点击响应需要在初始化 SDK 时，为 `YSFOptions` 的 `onMessageItemClickListener` 赋值：

```java

// 初始化代码
// YSFOptions options = new YSFOptions();
OnMessageItemClickListener messageItemClickListener = new OnMessageItemClickListener() {

    // 响应 url 点击事件
    public void onURLClicked(Context context, String url) {
        // 打开内置浏览器等动作
    }
}
options.onMessageItemClickListener = messageItemClickListener;

// ... 其他初始化代码
//Unicorn.init(this, "你的appid", options, new UILImageLoader());
```