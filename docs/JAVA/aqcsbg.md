# 安全测试报告处理方案
## 密码复杂度低
### 涉及功能
账户管理新增密码，个人中心修改密码
### 解决方案
前端+后端使用正则匹配输入的密码，对复杂度低的密码做驳回处理
## 跨站点请求伪造(CSRF)
### 涉及功能
所有接口
### 解决方案
后端对请求做Referer认证，确认请求是否合理， 同时所有接口做token认证，失效或者不存在，拒绝请求
## 敏感数据
### 涉及功能
所有包含敏感数据的接口和页面
### 解决方案
后端已做敏感数据脱敏，可配置脱敏字段
## 越权问题
### 涉及功能
所有接口
### 解决方案
对接口的入口做统一处理，查询该token的权限，防止越权
## accessTokent未失效
### 涉及功能
所有接口
### 解决方案
手动退出和异常退出登录后，发起请求，清除token
## Oracle Application Server PL/SQL 未授权的 SQL 查询执行
### 涉及功能
所有接口
### 解决方案
开发/测试/验收环境均没有此问题，appscan只是扫描到请求200，但是实际接口的响应值为500，并在响应中描述了错误信息，实践中我们用的DB也并非Oracle
## SQL盲注
### 涉及功能
/gateway/organization
/organization/agency
### 解决方案
appscan扫描到了请求200，认为存在sql注入的可能性，实际接口对参数做了校验，并且返回500，并且我们在开发过程中已经扼杀了sql注入的可能性
## 补充
对于appscan扫描结果不正确的补充， 在我们的架构中结合了restful风格，对接口返回值做了统一封装，http请求成功后，需要查看响应值里的code码和msg信息
## 后期安全措施补充
### ip黑白名单
登录限制ip，使用可配置的字典表保存黑白名单的ip地址
### 数据安全
涉及敏感数据的接口前后端做RSA非对称加密处理
## APP安全计划
   ### 移动客户端程序安全  
#### 安装包签名和证书 
   1. 漏洞说明：检测客户端是否经过正确签名。检测 app 移动客户端安装包是 
否正确签名，通过签名，可以检测出安装包在签名后是否被修改过。 
   2. 测试结果：安全
   3. 测试步骤：com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 使用 keytool 查看 META-INF\CERT.RSA 证书文件发现证书是否为 debug 证书，签名算法是否为 弱算法。使用命令 keytool -printcert -file CERT.RSA
#### 应用程序数据可备份 
   1. 漏洞说明：Android 2.1 以上的系统可为 App 提供应用程序数据的备份和恢 复功能，该由 AndroidMainfest.xml 文件中的 allowBackup 属性值控制，其默认 值为 true。当该属性没有显式设置为 false 时,攻击者可通过 adb backup 和 adb restore 对 App 的应用数据进行备份和恢复,从而可能获取明文存储的用户敏感信息。 
   2. 测试结果： 中危 
   3. 测试步骤：使用 Android Studio 打开 apk 文件查看 AndroidMainfest.xml 中 allowBackup 属性。
 #### 移动客户端程序保护
   1. 漏洞说明：app 代码未保护，可能面临被反编译的风险。反编译是将二进 
制程序转换成人们易读的一种描述语言的形式。反编译的结果是应用程序的代 
码，这样就暴露了客户端的所有逻辑，比如与服务端的通讯方式，加解密算法、密钥，转账业务流程、软键盘技术实现等等。攻击者可以利用这些信息窃 
取客户端的敏感数据，包括手机号、密码；截获与服务器之间的通信数据；绕 
过业务安全认证流程，直接篡改用户账号信息；对服务器接口发起攻击等。 
   2. 测试结果：高危 测试步骤： 
使用 7-zip 将 apk 文件中 classes.dex 解至测试文件夹，使用 dex2jar 反编译，使 用 jadx-gui 查看反编译后文件内容。
   3. 安全建议：建议移动客户端进行加壳处理防止攻击者反编译 app 移动客户 端，同时混淆 app 移动客户端代码。
 #### debug 模式
   1. 漏洞说明：客户端软件 AndroidManifest.xml 中的 android:debuggable="true" 标记如果开启，可被 Java 调试工具例如 jdb 进行调试，获取和篡改用户敏感信 息，甚至分析并且修改代码实现的业务逻辑，我们经常使用 android.util.Log 来 打印日志，软件发布后调试日志被其他开发者看到，容易被反编译破解。 
   2. 测试结果：安全 
   3. 测试步骤：使用 Android Studio 打开 apk 文件查看 AndroidMainfest.xm 中 android:debuggable 属性，是否为 true。
 #### 应用完整性校验
   1. 漏洞说明：测试客户端程序是否对自身完整性进行校验。攻击者能够通过 
反编译的方法在客户端程序中植入自己的木马，客户端程序如果没有自校验机 
制的话，攻击者可能会通过篡改客户端程序窃取手机用户的隐私信息。 
   2. 测试结果：中危
   3. 测试步骤：将 apk 反编译，修改 apk 内容，重新对 apk 进行打包签名。查 看重新打包后的 apk 是否可以安装运行。 
在手机上卸载已安装的应用，安装已重新签名的软件，打开后发现内容已 被篡改。
重新安装 app 发现 app 能正常打开，并内内容已被篡改

 ### 组件安全
 #### Activity
   1. 漏洞说明：Android 每一个 Application 都是由 Activity、Service、content Provider 和 Broadcast Receiver 等 Android 的基本组件所组成，其中 Activity 是 实现应用程序的主体，它承担了大量的显示和交互工作，甚至可以理解为一个 
“界面”就是一个 Activity。没有对调用 activity 的组件进行权限验证，可造成权 
限绕过，钓鱼欺诈。 
   2. 测试结果：中危
   3. 测试步骤：使用 drozer 对 app 的 Activity 进行检测 
使用 run app.activity.info -acom.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 获取 activity 信息。 执行 run app.activity.start –component
com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love “页面”检测是否可调用响应页面查看 androidmanifast.xml文件发现com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love.activity.WelcomeActi vity 为 app 启动入口
 用com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love.activity.User_DetailsActivity 手机弹出了个人信息页面
   4. 漏洞危害：可能被系统或者第三方的应用程序直接调出并使用。组件导出 
可能导致登录界面被绕过、信息泄露、数据库 SQL 注入、DOS、恶意调用等风 
险
   5. 安全建议： 
      1) app 内使用的私有 Activity 不应配置 intent-filter，如果配置了 intent-filter 需设 置 exported 属性为 false； 
      2) 使用默认 taskAffinity；
      3) 使用默认 launchMode；
      4) 启动 Activity 时不设置 intent 的 FLAG_ACTIVITY_NEW_TASK 标签；
      5)  谨慎处理接收的 intent 以及其携带的信息；
      6)  签名验证内部（in-house）app；
      7)  当 Activity 返回数据时候需注意目标 Activity 是否有泄露信息的风险；
      8)  目的 Activity 十分明确时使用显示启动；
      9)  谨慎处理 Activity 返回的数据，目的 Activity 返回的数据有可能是恶意应用
      伪造的；
      10)  验证目标 Activity 是否恶意 app，以免受到 intent 欺骗，可用 hash 签名验证；
      11)  When Providing an Asset Secondhand, the Asset should be Protected with the Same Level of Protection；
      12)  尽可能的不发送敏感信息，应考虑到启动 public Activity 中 intent 的信息均有
      可能被恶意应用窃取的风险。
#### Service
   1. 漏洞说明：一个 Service 是没有界面且能长时间运行于后台的应用组件．其 
它应用的组件可以启动一个服务运行于后台，即使用户切换到另一个应用也会 
继续运行．另外，一个组件可以绑定到一个 service 来进行交互，即使这个交互 是进程间通讯也没问题．例如，一个 service 可能处理网络事物，播放音乐，执 行文件 I/O，或与一个内容提供者交互，所有这些都在后台进行。 
测
   2. 测试结果：安全
   3. 测试步骤：使用 drozer 进行 service 检测
使用 run app.service.info -a com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 查看 service 信息
#### Broadcast Reciever
   1. 漏洞说明：Broadcast Recevier 广播接收器是一个专注于接收广播通知信 息， 并做出对应处理的组件。很多广播是源自于系统代码的──比如，通知时区 改
变、电池电量低、拍摄了一张照片或者用户改变了语言选项。应用程序也可 以
进行广播──比如说，通知其它应用程序一些数据下载完成并处于可用状态。 
应用程序可以拥有任意数量的广播接收器以对所有它感兴趣的通知信息予以响 
应。所有的接收器均继承自 BroadcastReceiver 基类。 广播接收器没有用户界 面。然而，它们可以启动一个 activity 来响应它们收到的信息，或者用 NotificationManager 来通知用户。通知可以用很多种方式来吸引用户的注意力 
──闪动背灯、震动、播放声音等等。一般来说是在状态栏上放一个持久的图 
标，用户可以打开它并获取消息。
   2. 测试结果：安全
   3. 测试步骤：使用 drozer 对 Broadcast Recevier 进行检测
使用 run app.broadcast.info -a 
com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 查看 Broadcast Recevier 信息。使用 run app.broadcast.send --component com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love “ 页面 ” 对组件发 送不完整 intent 来测试安全问题。
#### Content Provider
   1. 漏洞说明：Android Content Provider 存在文件目录遍历安全漏洞，该漏洞 源于对外暴露 Content Provider 组件的应用，没有对 Content Provider 组件的访 问进行权限控制和对访问的目标文件的 Content Query Uri 进行有效判断，攻击 者利用该应用暴露的 Content Provider 的 openFile()接口进行文件目录遍历以达 到访问任意可读文件的目的。在使用 Content Provider 时，将组件导出，提供 了 query 接口。由于 query 接口传入的参数直接或间接由接口调用者传入，攻 击者构造 sql injection 语句，造成信息的泄漏甚至是应用私有数据的恶意改写 
和删除。
   2. 测试结果：安全
   3. 测试步骤：使用 drozer 对 Content Provider 进行检测 使
用 run scanner.provider.finduris -a com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 检测可以访问页面
#### WebView  代码执行检查
   1. 漏洞说明：Android 系统通过 WebView.addJavascriptInterface 方法注册可供 JavaScript 调用的 Java 对象，以用于增强 JavaScript 的功能。但是系统并没有对 注册 Java 类的方法调用的限制。导致攻击者可以利用反射机制调用未注册的其 它任何 Java 类，最终导致 JavaScript 能力的无限增强。攻击者利用该漏洞可以 
   根据客户端执行任意代码
   2. 测试结果：安全
   3. 测试步骤：用 drozer 的 run scanner.misc.checkjavascriptbridge -a com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 命令检查 webview 中 addJavascriptInterface 的使用是否存在安全隐患。（“-a”参数后加包名），返 回结果为 not vulnerable，即为安全
   测试结果：返回结果为 not vulnerable，即为安全。
#### WebView 不校验证书检测
   1. 漏洞说明：调用了 android/webkit/SslErrorHandler 类的 proceed 方法,可能导 致 WebView 忽略校验证书的步骤。
   2. 测试结果：高危
   3. 测试步骤： 使用 android/webkit/SslErrorHandler 类的 proceed 方法可 
   忽略证书异常，继续访问服务端，在不导入根证书情况下检测是否可以获取数 
   据包
   安全建议：
       1) 不要调用 android.webkit.SslErrorHandler 的 proceed 方法； 
       2) 当发生证书认证错误时，采用默认的处理方法 SslErrorHandler.cancel()，停 止加载问题页面
#### WebView 密码明文保存检测
   1. 漏洞说明：在使用 WebView 的过程中忽略了 WebView setSavePassword， 当用户选择保存在 WebView 中输入的用户名和密码，则会被明文保存到应用 数据目录的 databases/webview.db 中。如果手机被 root 就可以获取明文保存的 
   密码，造成用户的个人敏感数据泄露。 
   2. 测试结果：安全
   3. 测试步骤： 使用 root 过的手机进入 
   /data/data/com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love 查看是 否存在 webview.db 文件。
### 敏感信息安全
#### 敏感信息文件和字符串检查
   1. 漏洞说明：测试移动客户端私有目录下的文件及 APP 自身代码是否存在敏 感信息泄露的情况，如在 app 私有目录下存储明文或简单加密密码，身份认 证，
   用户访问记录，敏感参数等敏感信息
   2. 测试结果：低危
   3. 测试步骤：将 
   /data/data/com.jinhuitai.marriage_and_love.jinhuitai_marriage_and_love/文件夹和拷   贝到 pc 端，查看是否存在明文信息或者敏感数据
   漏洞危害：可能泄露 APP 自身关键信息（如私钥）、服务端接口信息以及 
   用户信息等。安全建议：  
        1) 敏感信息不要直接硬编码在 APP 代码中； 
        2) 敏感信息在本地存储时进行加密。
#### 程序目录权限检查
   1. 漏洞说明：测试移动客户端私有目录下的文件权限是否设置正确，非 root 
   账户是否可以读，写，执行私有目录下的文件。
   2. 测试结果：安全
   3. 测试步骤：使用 root 过的手机进入/data/data/cn.swiftpass.enterprise.intl 查看 该文件夹内文件夹和文件权限，非 root 账户是否可以读，写，app 私有目录下的文件
#### logcat 日志检查
   1. 漏洞说明：调试日志函数可能输出重要的日志文件，其中包含的信息可能 
   导致客户端用户信息泄露，暴露客户端代码逻辑等，为发起攻击提供便利
   2. 测试结果：安全
   3. 测试步骤：使用 adb logcat 将系统日志导出，查看日志中 spay 日志是否存 在 username，password，session，token，key，cookie 等敏感数据。
   测试结果：搜索关键字未发现敏感数据。
#### 敏感数据文件残留检测
   1. 漏洞描述：移动客户端卸载后，文件系统中残留与用户相关的个人信息及 
   敏感数据等，易导致用户信息泄露。
   2. 测试结果：安全
   3. 测试步骤：卸载移动客户端后，查看移动客户端目录是否有残留与用户相 
   关的个人信息及敏感数据。
### 密码软键盘安全
#### 键盘劫持测试
   1. 漏洞说明：测试移动客户端程序在密码等输入框是否使用自定义软键盘。 安卓应用中的输入框默认使用系统软键盘，手机安装木马后，木马可以通过替 换系统软键盘，记录应用的密码。
   2. 测试结果：安全
   3. 测试步骤：安装安卓按键记录工具（Hackers Keylogger.apk），在设置中更 换输入法，启动 APP，查看测试键盘记录
#### 键盘随机布局测试
   1. 漏洞说明：测试移动客户端是否使用随机布局的密码软键盘。
   危险等级：低危
   2. 测试步骤：手工使用 APP 的键盘，查看键盘布局是否随机。
   3. 漏洞危害：网银等支付类应用，若使用固定位置的密码软键盘，则用户的 
   密码有可能被攻击者记录。
   4. 安全建议：建议移动客户端对自定义软键盘进行随机化处理，同时在每次 
   点击输入框时都进行随机初始化
#### 屏幕录像测试
   1. 漏洞说明：测试通过连续截图，是否可以捕捉到用户密码输入框的密码。
   危险等级：低危
   2. 测试步骤：通过系统截图软件对 app 进行截图。
   3. 漏洞危害：网银等支付类应用，若密码输入界面可被截屏，则用户的密码 有
   可能被攻击者记录。 
   4. 安全建议：在 Activity 的 onCreate()方法的 Layout 初始化部分加入以下代 
   码 Window win = getWindow();win.addFlags(WindowManager.LayoutParams.FLAG_SECURE);防止截屏
### 安全策略
#### 密码复杂度
   1. 漏洞说明：测试移动客户端程序是否检查用户输入的密码，禁止用户设置 弱 口令。
   2. 测试结果：安全 
   3. 测试步骤：测试能否将用户密码修改为弱口令
#### 帐户锁定策略
   1. 漏洞说明：测试移动客户端是否限制登录尝试次数。防止木马使用穷举法 
   暴力破解用户密码。 
   2. 测试结果：安全
   3. 测试步骤：尝试输错 5 次密码之后是否会封锁账号.在测试过
   程中，使用错误的用户密码进行登录操作，当登录失 
   败五次后，仍然能正常的尝试密码，不过抓包发现密码加密，不能暴力破解，因此此项为安全。
#### 会话安全设置
   1. 漏洞说明：测试移动客户端在一定时间内无操作后，是否会使会话超时并 要求重新登录。超时时间设置是否合理。
   2. 测试结果：低危 
   3. 验证步骤：十分钟无操作后查看会话是否失效，是否需要 密码才能使用
   十分钟无操作后查看会话是否失效，是否需要 密码才能使用。
   ：十分钟无操作后，再打开 app 会话不会失效，
   不需要密码。
   4. 漏洞危害：会话超时设置不合理会增加用户帐户的安全风险。 
   5. 安全建议：
   建议在移动客户端编写会话安全设置的逻辑，当 10 分钟或 20 分钟无操作时自动退出登录状态或是关闭移动客户端。
   建议在移动客户端编写会话安全设置的逻辑，当 10 分钟或 20 分钟无操作时自动退出登录状态或是关闭移动客户端。
#### 界面切换保护
   1. 漏洞描述：检查移动客户端程序在切换到后台或其他应用时，是否能恰当 响应（如清除表单或退出会话），防止用户敏感信息泄露。 
   2. 测试结果：安全 
   3. 测试步骤：在登陆界面填写登陆名和密码，然后切出，再
   进入客户端，看 输入的登陆名和密码是否清除。
#### 登录界面设计
   1. 漏洞说明：登录时返回“用户名不存在”或“密码错误”等详细的登录错 
   误提示信息，有利于攻击者获取注册用户信息。 
   2. 测试结果：安全 
   3. 测试步骤：尝试使用不存在的用户名登陆与存在的用户名及错误密码登 陆，
   检测是否可根据界面提示进行用户名枚举。
   4. 漏洞危害：攻击者可通过该漏洞使用用户名字典遍历 APP 用户。 
   5. 安全建议：建议用户名或密码输入错误均提示“用户名或密码错误”，若 
   移动客户端同时还希望保证客户使用的友好性，可以在登陆界面通过温馨提示 
   的方式提示输入错误次数，密码安全策略等信息，以防用户多次输入密码错误 
   导致账号锁定。 
#### UI 敏感信息泄漏
   1. 漏洞说明：检查移动客户端的各种功能，看是否存在敏感信息泄露问题。 
   2. 测试结果：安全 
   3. 测试步骤：查看所有页面信息无敏感数据。 测试结果：查看所有页面未发
   现敏感信息泄露
#### 帐号登录限制
   1. 漏洞说明：测试能否在两个设备上同时登录同一个帐号。 
   2. 测试结果：安全
   3.  验证步骤：测试发现不能使用两台设备登录同一个账号。
#### 安全退出
   1. 漏洞说明：验证移动客户端在用户退出登录状态时是否会和服务器进行通 
   信以保证退出的及时性。 
   2. 测试结果：低危 
   3. 验证步骤： 获取订单处数据包，然后点击安全退出，发现 app 向服务器发 送 logout 数据包，使用订单处数据包再次请求查看身份验证是否失效。
#### 验证码安全性
   1. 漏洞说明：验证移动客户端使用的验证码的安全性。 测
   试结果：低危 
   2. 测试步骤： 使用同一个携带验证码的数据包提交两次以上看服务器是否拒 
   绝请求。
   3. 漏洞危害：如图片验证码在客户端验证，或图片验证码强度弱可被程序识 
   别，会造成验证码失效。 
   4. 安全建议：建议移动客户端从移动客户端处获取图形验证码相关信息，并 且图形验证码符合图形验证码安全策略。
#### 密码修改验证
   1. 漏洞说明：验证移动客户端进行密码修改时的安全性。
   2. 测试结果：安全
   3. 测试步骤：尝试在修改密码时使用错误的旧密码，检测服务端是否验证旧 密码的正确性。
### 通信安全
#### 通信加密检测
   1. 漏洞说明：验证移动客户端和服务器之间的通信是否使用加密信道。 
   2. 测试结果：高危
   3. 测试步骤：使用 burp 抓包，是否使用 http，或者使用 wireshark 抓包是否使 用 http。
   4. 漏洞危害：用户存在被中间人攻击的风险。 
   5. 安全建议：建议移动客户端同
   服务器进行通信时信道使用 SSL 加密信道进 行传输，同时保证加密信道的本身安全（SSLV2，SSLV3 已被证明存在漏洞，
#### 关键数据加密和校验检测
   1. 漏洞说明：验证移动客户端和服务器之间的关键数据（请求中 data 部分） 
   交互是否使用加密信道，检测能否对关键数据进行篡改。 
   2. 测试结果：安全 测试步骤：修改数据包中部分数据，测试是否对数据包的
   部分进行签名。
#### SSL 证书有效性检测
   1. 漏洞说明：验证移动客户端和服务器之间是否存在双向验证的机制，同时 
   确认此机制是否完善，服务器是否以白名单的方式对发包者的身份进行验证。 
   2. 测试结果：因服务端采用 http 通信，因此不存在此项。
### 进程安全
#### 内存访问和修改检测
   1. 漏洞说明：通过对移动客户端内存的访问，木马将有可能会得到保存在内 
   存中的敏感信息（如登录密码，帐号等）。测试移动客户端内存中是否存在的 
   敏感信息（卡号、明文密码等等）。 
   2. 测试结果：安全 
   3. 测试步骤：使用 memspector.apk 搜索内存敏感信息。测试结果：在内存中未搜索到相关数据。
#### 本地端口开放检测
   1. 漏洞说明：通常使用 PF_UNIX、PF_INET、PF_NETLINK 等不同 domain 的 socket 来进行本地 IPC 或者远程网络通信，这些暴露的 socket 代表了潜在的 本地或远程攻击面，历史上也出现过不少利用 socket 进行拒绝服务、root 提权 或者远程命令执行的案例。特别是 PF_INET 类型的网络 socket，可以通过网络 与 Android 应用通信，其原本用于 linux 环境下开放网络服务，由于缺乏对网 络调用者身份或者本地调用者 id、permission 等细粒度的安全检查机制，在实 现不当的情况下，可以突破 Android 的沙箱限制，以被攻击应用的权限执行命 
   令，通常出现比较严重的漏洞。 
   2. 测试结果：安全 
   3. 测试步骤：使用 adb shell  运行 busybox netstat -anp 查看开放端口。
#### 外部动态加载 DEX 安全风险检测
   1. 漏洞说明：Android 系统提供了一种类加载器 DexClassLoader，其可以在 运行时动态加载并解释执行包含在 JAR 或 APK 文件内的 DEX 文件。外部动态 加载 DEX 文件的安全风险源于：Android4.1 之前的系统版本容许 Android 应用 将动态加载的 DEX 文件存储在被其他应用任意读写的目录中(如 sdcard)，因此 不能够保护应用免遭恶意代码的注入；所加载的 DEX 易被恶意应用所替换或 者代码注入，如果没有对外部所加载的 DEX 文件做完整性校验，应用将会被 
   恶意代码注入，从而执行的是恶意代码。 
   2. 测试结果： 安全 
   3. 测试步骤：在源码中搜索 DexClassLoader，查看 AndroidMainfest.xml 包 
   package 值相对应路径下的文件中是否含有 DexClassLoade()函数调用。查看 
   AndroidMainfest.xml 中 sdk 最低版本是否小于 16。检查在 sdcard 目录中是否 
存在 app 加载的 dex 文件。
### 业务安全
#### 水平越权访问及修改漏洞
   1. 漏洞说明：访问个人信息，修改认证信息时，都只使用了单一的身份认证 
   (UserId)，导致了水平越权访问和越权修改，且 UserId 前几位为手机的运营商 位置和运营商信息，减小了爆破 UserId 的难度。 
   漏洞 URL：test-api.jhtai8.com 
   2. 危险等级：高危
   3. 漏洞危害:  
         1. 导致任意用户敏感信息泄露   
         2. 导致任意用户信息被恶意修改或删除
   4. 修复建议： 
   在 web 层的逻辑中做鉴权，检查提交 CRUD 请求的操作者（通过 session 信息 
   得到）与目标对象的权限所有者（查数据库）是否一致，如果不一致则阻断

