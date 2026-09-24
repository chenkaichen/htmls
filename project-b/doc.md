# B面接入文档

## 一、zip说明
1、ClinkNativeDeviceInfo.swift - flutter<->iOS桥接方法

2、Constant.swift - appList

3、SecurityManager.swift - 安全

4、robov - B面

**说明：<4放入flutter项目lib文件夹下，123放入iOS工程中>**

## 二、注意事项

### 2.1 A/B面判断
登录接口返回的userGroup字段 0->B else->A

### 2.2 X-Device-Id字段
A/B面的请求header中的X-Device-Id字段必须为同一个值

### 2.3 启动时权限展示判断
此字段需要存入userdefault，因为如果用户删除应用重新安装的情况下，不会弹出此页面，会导致adjustSDK不能注册

### 2.4 facebook说明
需要在应用商城中有发布过才能接入，在Info.plist文件中加入参数如[4.4](#4.4)，然后在AppDelegate文件中加入方法
```
        ApplicationDelegate.shared.application(
            application,
            didFinishLaunchingWithOptions: launchOptions
        )  
```
## 三、B面需要修改

### 3.1 客服
需要设置客户信息
```
    await _invokeFreshchat('identifyUser', <String, String>{
      'externalId': userId.trim(),
      'restoreId': 'restore_$userId',
    });

    final cleanEmail = email?.trim();

    final displayName =
        '${AppPublicConstants.appName}-IOS-${phone != null ? '0${phone}' : cleanEmail}';

    await _invokeFreshchat('setUser', <String, String?>{
      'firstName': displayName,
      'email': cleanEmail,
      'phoneCountryCode': '84',
      'phoneNumber': phone,
    });

    await _invokeFreshchat('setUserProperties', <String, String?>{
      'user_id': userId.trim(),
      'platform': 'IOS',
      'country': 'Vietnam',
      if (phoneParts != null) 'phone': phone,
      if (cleanEmail != null && cleanEmail.isNotEmpty) 'email': cleanEmail,
    });
```

### 3.2 webview_controller需要修改

#### 3.2.1 退出登录
需要跟A面一致，跳转至登录页面
```
  Future<void> _handleLogout() async {
   
  }
```

#### 3.2.1 打开客服
需要直接跳转聊天页面，非列表
```
  Future<void> _handleOpenCustomerService(Map<String, dynamic>? params) async {
   
  }
```
isSingleOpen == true 跳转聊天界面，默认为false不影响A面
```
 Future<CustomerSupportResult> openConversations({
    bool isSingleOpen = false,
  }) async {
    if (!config.isConfigured) return CustomerSupportResult.notConfigured;

    await init();
    try {
      runZonedGuarded(
        () => Freshchat.showConversations(
          filteredViewTitle: 'Hỗ trợ Clink',
          tags: isSingleOpen ? ['vn_session'] : [],
        ),
        (error, _) => debugPrint('FreshchatService: mở chat lỗi - $error'),
      );
      return CustomerSupportResult.opened;
    } catch (error) {
      debugPrint('FreshchatService: mở chat lỗi - $error');
      return CustomerSupportResult.failed;
    }
  }
```

## 四、外壳需要修改

### 4.1 新增文件
把zip解压的一下三个文件copy到项目中
```
ClinkNativeDeviceInfo.swift
Constant.swift
SecurityManager.swift
```

### 4.2 Podfile文件修改

#### 4.2.1 新增依赖
```
pod 'FBSDKCoreKit'
pod 'IOSSecuritySuite'
```
#### 4.2.2 新增代码
```
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= [
        '$(inherited)',
        'PERMISSION_CAMERA=1',
        'PERMISSION_PHOTOS=1',
      ]
    end
  end
end
```
  
### 4.3 AppDelegate文件修改
```
    func didInitializeImplicitFlutterEngine(_ engineBridge: FlutterImplicitEngineBridge) {
        GeneratedPluginRegistrant.register(with: engineBridge.pluginRegistry)
        ClinkNativeDeviceInfo.register(messenger: engineBridge.applicationRegistrar.messenger())
    }
```

<a id="4.4"></a>
### 4.4 Info.plist文件新增
```
	<key>LSApplicationQueriesSchemes</key>
	<array>
		<string>vneid</string>
		<string>zalo</string>
        <string>momo</string>
        <string>feonline2</string>
        <string>hcvn</string>
        <string>tnex</string>
        <string>hpo</string>
        <string>dudu</string>
        <string>hdsaison</string>
        <string>timo</string>
        <string>lfvn</string>
        <string>tiktok</string>
        <string>grab</string>
        <string>googlemaps</string>
        <string>cake.vn</string>
	</array>
	
		<key>CFBundleURLTypes</key>
	<array>
		<dict>
			<key>CFBundleTypeRole</key>
			<string>Editor</string>
			<key>CFBundleURLName</key>
			<string>facebook_scheme</string>
			<key>CFBundleURLSchemes</key>
			<array>
				<string>fb1602104891710141</string>
			</array>
		</dict>
	</array>
	
		<key>FacebookAdvertiserIDCollectionEnabled</key>
	<true/>
	<key>FacebookAppID</key>
	<string>1602104891710141</string>
	<key>FacebookAutoLogAppEventsEnabled</key>
	<true/>
	<key>FacebookClientToken</key>
	<string>9a628f7940b0b969725992c46b77b99d</string>
	<key>FacebookDisplayName</key>
	<string>Clink</string>
```



