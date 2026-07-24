## 一、架构模式：
Flutter 客户端采集多维环境信号 → 上报后台；后台综合打分判定A / B

原则：客户端只采集、组装、上报；客户端不做任何 A/B 判定，所有判断交给服务端

## 二、工作机制
**整体链路：启动页 → 权限协议状态校验 → 登录页 → 首页**

流程分支逻辑：
1. App冷启动，展示启动页；
2. 校验本地记录：用户是否已同意权限声明
   - ✅ **已同意权限声明**：直接跳转至【登录页】
   - ❌ **未同意权限声明**：展示【权限声明页面】
     - 权限声明页点击【同意】：唤起ATT授权弹窗获取IDFA，完成后进入登录页
     - 权限声明页点击【取消】：**不唤起ATT、不获取IDFA**，直接进入登录页
3. 登录页
     - 采集：IP + 设备当前国家码 + 设备当前语言 + 设备当前时区 + 设备当前版本号 + 设备名称 + IDFV + [当前版本号](#当前版本号) + [安全环境检测](#安全环境检测) + [appList](#用户已安装应用列表)
     - 上报：请求服务器接口获取requestId
4. 登录行为
     - 采集：[审核账号](#审核账号) + requestId
     - 登录：请求服务器接口获取登录状态及判定字段userGroup
5. 登录成功，首页路由分流
     - B面：userGroup = 0 || userGroup = 12
     - A面：else

**注意：** B面H5链接加密

## 三、名词解释
<a id="安全环境检测"></a>
### 3.1 安全环境检测
1. 是否越狱
2. 是否开启代理/VPN/中间人抓包
3. 是否注入Frida
4. 是否附加调试器
5. 是否模拟器

**区分自动化测试环境、阻止逆向研究员抓包观察 B面。以上五点只要有一点为true，都应视为审核环境**

<a id="用户已安装应用列表"></a>
### 3.2、已安装应用列表
用户已安装应用列表 **非系统级应用**

**鉴于Otter Keep、Hedgy都是因为加了非合规方式获取应用列表而导致被封禁，是否在此次使用合规方式获取**

#### 3.2.1 非合规方式

方式简单，获取到的信息最全，但风险最高

```json
{
    "switch": "off",
    "data": [
        {
            "installTime": 1781591762000,
            "genre": "",
            "genreId": "0",
            "pkg": "com.tencent.xin",
            "diskUse": "2756001792",
            "isSearch": "1",
            "appId": "0",
            "teamId": "88L2Q4487U",
            "type": "0",
            "versionName": "8.0.74",
            "source": "",
            "lngId": "",
            "versionCode": "8.0.74.34",
            "name": "微信",
            "updateTime": 0
        },
        {
            "installTime": 1781591763000,
            "genre": "Developer Tools",
            "genreId": "6026",
            "pkg": "developer.apple.wwdc-Release",
            "diskUse": "23654400",
            "isSearch": "1",
            "appId": "640199958",
            "teamId": "9JA89QQLNQ",
            "type": "0",
            "versionName": "11.0.2",
            "source": "com.apple.AppStore",
            "lngId": "2",
            "versionCode": "1102.3.1",
            "name": "Apple Developer",
            "updateTime": 1781492400000
        }
    ]
}
```

#### 3.2.1 合规方式

方式麻烦，获取到的信息单一，需要手动收集其他APP的scheme，比较耗时，有风险但程度低

```json
{
    "data": [
        {
            "name": "wechat",
            "scheme": "wechat",
            "pkg": "com.tencent.xin",
            "isInstall": true
        },
        {
            "name": "ÁgilDinero",
            "scheme": "agildin",
            "pkg": "com.agildin.app.mx",
            "isInstall": true
        }
    ],
    "switch": "on"
}
```

<a id="当前版本号"></a>
### 3.3、当前版本号

应用当前版本号：用来判断是否正在审核中，如果当前版本号等于审核版本号，一定进入A面

<a id="审核账号"></a>
### 3.4、审核账号

用来给审核人员审查版本更新  **一定进入A面**