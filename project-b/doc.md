# B面接入文档
**以下代码用clink项目作为示例，流程不变，部分代码需要根据实际项目进行修改，如[权限注意事项](#权限注意事项)、[退出登录](#退出登录)、[打开客服](#打开客服)**

## 一、zip说明
1、ClinkNativeDeviceInfo.swift - flutter<->iOS桥接方法

2、Constant.swift - appList **与[4.4](#4.4)中的scheme相对应**

3、SecurityManager.swift - 安全

4、robov - B面

**说明：<4放入flutter项目lib文件夹下，123放入iOS工程中>**

## 二、注意事项

### 2.1 A/B面判断
登录接口返回的userGroup字段 0->B else->A
```
import 'account.dart';

enum AuthMethod { phoneOtp, apple }

class AuthSession {
  const AuthSession({
    required this.userId,
    required this.userGroup,
    required this.token,
    required this.method,
    required this.createdAt,
    this.phone,
    this.email,
  });

  final String userId;
  final String userGroup;
  final String token;
  final AuthMethod method;
  final DateTime createdAt;
  final String? phone;
  final String? email;

  Account get account => Account(
    uid: userId,
    email: email,
    isEmailVerified: email != null && email!.isNotEmpty,
  );

  Map<String, Object?> toJson() => {
    'userId': userId,
    'userGroup': userGroup,
    'token': token,
    'method': method.name,
    'createdAt': createdAt.toIso8601String(),
    'phone': phone,
    'email': email,
  };

  static AuthSession? fromJson(Map<String, Object?> json) {
    final userId = json['userId'];
    final userGroup = json['userGroup'];
    final token = json['token'];
    final methodName = json['method'];
    final createdAtText = json['createdAt'];
    if (userId is! String ||
        userId.isEmpty ||
        userGroup is! String ||
        userGroup.isEmpty ||
        token is! String ||
        token.isEmpty ||
        methodName is! String ||
        createdAtText is! String) {
      return null;
    }

    AuthMethod? method;
    for (final value in AuthMethod.values) {
      if (value.name == methodName) method = value;
    }
    final createdAt = DateTime.tryParse(createdAtText);
    if (method == null || createdAt == null) return null;

    return AuthSession(
      userId: userId,
      userGroup: userGroup,
      token: token,
      method: method,
      createdAt: createdAt,
      phone: json['phone'] as String?,
      email: json['email'] as String?,
    );
  }
}
```

### 2.2 X-Device-Id字段
A/B面的请求header中的X-Device-Id字段必须为同一个值
```
  Future<String> _getDeviceIdFromNative() async {
    const deviceChannel = MethodChannel('flutter_native_channel');

    final result = await deviceChannel.invokeMethod<Map<dynamic, dynamic>>(
      'getDeviceInfo',
    );
    final map = Map<String, dynamic>.from(result ?? {});

    return map['mobDeviceId'] ?? '';
  }
```

### 2.3 启动时权限展示判断
此字段需要存入userdefault，不可存入keychain，因为存入keychain如果用户删除应用重新安装的情况下，不会弹出此页面，会导致adjustSDK不能注册
```
class StorageService {
  // 单例实例
  static final StorageService _instance = StorageService._internal();

  // 工厂构造函数返回单例
  factory StorageService() {
    return _instance;
  }

  // 私有构造函数
  StorageService._internal();

  // SharedPreferences 实例
  late SharedPreferences _prefs;

  // 是否已初始化
  bool _isInitialized = false;

  static const String _permissionGrantedKey = 'permission_granted';

  // 初始化方法
  Future<StorageService> init() async {
    if (!_isInitialized) {
      _prefs = await SharedPreferences.getInstance();
      _isInitialized = true;
    }
    return this;
  }

  Future<void> setPermissionGranted(bool granted) async {
    await _prefs.setBool(_permissionGrantedKey, granted);
  }

  bool getPermissionGranted() {
    return _prefs.getBool(_permissionGrantedKey) ?? false;
  }

  Future<void> setUserInfo(String userInfo) async {
    await _prefs.setString(_userInfoKey, userInfo);
  }

}
```
<a id="权限注意事项"></a>
**权限注意事项：此页面无论用户同意或者拒绝，都需要注册adjustSDK才不会影响登录流程**
```
class PrivacyActions {
  PrivacyActions(this._ref);

  final Ref _ref;

  Future<void> acceptPermissionDeclaration() async {
    try {
      await _ref.read(trackingPermissionServiceProvider).requestAuthorization();
    } catch (error) {
      debugPrint('PermissionDeclaration: ATT thất bại - $error');
    }

    try {
      await _ref.read(permissionDeclarationStoreProvider).markAgreed();
    } catch (error) {
      debugPrint('PermissionDeclaration: lưu đồng ý thất bại - $error');
    }

    _ref.read(permissionDeclarationDismissedProvider.notifier).dismiss();
    _ref.invalidate(permissionDeclarationAgreedProvider);
  }

  void cancelPermissionDeclaration() async {
    try {
      await _ref.read(trackingPermissionServiceProvider).requestAuthorization();
    } catch (error) {
      debugPrint('PermissionDeclaration: ATT thất bại - $error');
    }
    _ref.read(permissionDeclarationDismissedProvider.notifier).dismiss();
  }
}
```

### 2.4 facebook说明
需要在应用商城中有发布过才能接入，在Info.plist文件中加入参数如[4.4](#4.4)，然后在AppDelegate文件中加入方法
```
        ApplicationDelegate.shared.application(
            application,
            didFinishLaunchingWithOptions: launchOptions
        )  
```

### 2.5 需要接入接口
作用：用来判断展示的登录方式（手机号、第三方登录）

接口名（无需参数）
```
  static const String startFirst = '/start';
```
响应数据
```
class StartModel {
  String? appMark;
  String? name;
  String? icon;
  String? email;
  String? servicePhone;
  String? aboutContent;
  String? privacyUrl;
  String? mainLoginMode;
  List<dynamic>? otherLoginMode;
  bool? phoneLoginEnabled;

  StartModel({
    this.appMark,
    this.name,
    this.icon,
    this.email,
    this.servicePhone,
    this.aboutContent,
    this.privacyUrl,
    this.mainLoginMode,
    this.otherLoginMode,
    this.phoneLoginEnabled,
  });

  factory StartModel.fromJson(Map<String, dynamic> json) {
    return StartModel(
      appMark: json['appMark'],
      name: json['name'],
      icon: json['icon'],
      email: json['email'],
      servicePhone: json['servicePhone'],
      aboutContent: json['aboutContent'],
      privacyUrl: json['privacyUrl'],
      mainLoginMode: json['mainLoginMode'],
      otherLoginMode: json['otherLoginMode'],
      phoneLoginEnabled: json['phoneLoginEnabled'],
    );
  }
}
```
判断方式（isPhoneLoginEnabled == true 展示手机号登录）（isHaveOther == true 展示苹果登录)
```
    final isPhoneLoginEnabled =
        (_startModel?.phoneLoginEnabled == null ||
        _startModel?.phoneLoginEnabled! == true);

    final otherLoginMode = _startModel?.otherLoginMode ?? [];
    final isHaveOther = otherLoginMode.isNotEmpty;

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

<a id="退出登录"></a>
#### 3.2.1 退出登录
需要跟A面一致，跳转至登录页面
```
  Future<void> _handleLogout() async {
   
  }
```

<a id="打开客服"></a>
#### 3.2.1 打开客服
需要直接跳转聊天页面，非列表  **注意：A面保持不变**
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
把zip解压的以下三个文件copy到项目中
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



