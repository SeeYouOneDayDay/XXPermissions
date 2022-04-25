# 权限申请框架阅读记

## 杂记

### 内部传递用的是Context

1. 若是activity,转成activity
2. 若是ContextWrapper,则ContextWrapperObj.getBaseContext()
3. 否则返回null

### 检查模式`mCheckMode`检查的是什么

1. 检查存储权限是否符合规范
   - 若无声明存储权限(读、写、管理)

   - 是否适配了分区存储--内部自定义的(ScopedStorage)
        ``` xml
        <!-- 表示当前项目已经适配了分区存储特性 -->
        <meta-data
            android:name="ScopedStorage"
            android:value="true" />
        ```

   - 解析清单,获取`application`的属性标签`requestLegacyExternalStorage`

        -  这里存储兼容是这样的。 
        - 兼容安卓10: requestLegacyExternalStorage  
        - 兼容安卓11: preserveLegacyExternalStorage
        - https://developer.android.com/about/versions/11/privacy/storage?hl=zh-cn#scoped-storage
        - https://developer.android.com/training/data-storage/use-cases?hl=zh-cn#opt-out-in-production-app
        - https://developer.android.com/training/data-storage/use-cases?hl=zh-cn#if_your_app_targets

   - 目标版本10、未声明`requestLegacyExternalStorage`,未声明管理权限`MANAGE_EXTERNAL_STORAGE`则抛异常

   - 如目标版本`targetSdkVersion`是11和以上、未声明管理权限`MANAGE_EXTERNAL_STORAGE`则抛异常

2. 检查定位权限是否符合规范

    -   兼容安卓12和以上版本，若申请精确定位信息`ACCESS_FINE_LOCATION`，必须同时申请粗略位置`ACCESS_COARSE_LOCATION`, 否则抛出异常
        -   https://developer.android.google.cn/about/versions/12/approximate-location
    -   不包含后台定位，退出
    -   申请粗略位置，不申请精确位置，抛出异常
        -   实践证明。后台可以没粗略权限，必须要精确权限，不然不能工作。——12无该问题
    -   检查需要包含三种定位一种（粗略定位`ACCESS_COARSE_LOCATION`、精确定位`ACCESS_FINE_LOCATION`、后台定位`ACCESS_BACKGROUND_LOCATION`）
    -   后台定位权限不能和其他权限一起申请，如无后台定位权限需求，慎选

3. `targetSdkVersion`和权限检测，确保低版本不申请高版本权限

    -   个人感觉，直接申请也无不可，不影响



### 解析清单相关

1.   通过`AssetManager.addAssetPath(String apkPath)`来加载apk(`context.getApplicationInfo().sourceDir`获取apk路径)，获取cookie(结果)
     1.   其他方法一: AssetManager.findCookieForPath
     2.   其他方法二:AssetManager.addAssetPathInternal
     3.   其他方法三:AssetManager.addOverlayPath
2.   通过`AssetManager.openXmlResourceParser`获取xml解析器
3.   检查加载的包名(`context.getPackageName`)和解析到的包名(`manifest`中的`package`)是否一致



### xml中检查权限

1.   解析清单相关方式解析xml中声明的权限及对应权限的最大版本(`maxSdkVersion`)
2.   如果解析xml解析为空，则通过`PackageManager.getPackageInfo().requestedPermissions`获取权限列表
     1.   思考:什么场景下会为空呢？
3.   遍历权限
     1.   通知栏权限(`NOTIFICATION_SERVICE`)–不需要申请–自定义权限？
     2.   通知栏监听权限(`BIND_NOTIFICATION_LISTENER_SERVICE`--4.3之后才有的特殊权限)
     3.   最小版本`minSdkVersion`小于12，蓝牙权限: 
          1.   `BLUETOOTH_SCAN`-->检查权限`BLUETOOTH_SCAN`,12之前，还需要定位权限`ACCESS_COARSE_LOCATION`
          2.   `BLUETOOTH_CONNECT`–>检查权限`BLUETOOTH`
          3.   `BLUETOOTH_ADVERTISE`-->检查权限`BLUETOOTH_ADMIN`
     4.   最小版本小于11，存储
          1.   `MANAGE_EXTERNAL_STORAGE`–>检查权限`READ_EXTERNAL_STORAGE`,检查权限`WRITE_EXTERNAL_STORAGE`
     5.   最小版本小于10，活动步数`ACTIVITY_RECOGNITION`-->检查权限`BODY_SENSORS`
     6.   最小版本小于8，读取手机号`READ_PHONE_NUMBERS`–>检查权限`READ_PHONE_STATE`
     7.   其他直接申请即可
4.   检查权限是否正常
     1.   检查是否声明。注意有可以删除权限的配置。(`<uses-permission android:name="xxx" tools:node="remove"/>`替换为`<uses-permission android:name="xxx" tools:node="replace"/>`
          1.   https://github.com/getActivity/XXPermissions/issues/98
     2.   检查权限的最大版本号如比xml获取的最大sdk版本号大，就抛出异常



### 优化申请权限列表

1.   小于安卓12，`ACCESS_COARSE_LOCATION`权限需要依赖`ACCESS_COARSE_LOCATION`权限
2.   安卓11只需要管理存储权限即可。 安卓11下需要读写权限。
     1.   管理存储权限`MANAGE_EXTERNAL_STORAGE`
     2.   读写存储卡权限(`READ_EXTERNAL_STORAGE`/`WRITE_EXTERNAL_STORAGE`)
3.   安卓8以下，读取手机号码权限`READ_PHONE_NUMBERS`和读取电话状态权限`READ_PHONE_STATE`合并为读取电话状态权限
     1.   高版本获取电话状态信息，需要`READ_PRIVILEGED_PHONE_STATE`,该库未实现
4.   安卓10以下，申请`ACTIVITY_RECOGNITION`且为申请`BODY_SENSORS`，合并为`BODY_SENSORS`
     1.   获取步数兼容，安卓10下使用`BODY_SENSORS`. 安卓10才剥离`ACTIVITY_RECOGNITION`权限



### 动态申请fragment附加到页面

1.   动态增加fragment到页面

     ```java
     activity.getFragmentManager().beginTransaction()
     	.add(this, this.toString())
     	.commitAllowingStateLoss();
     ```

     

2.   动态减少页面的fragment

     ```java
     activity.getFragmentManager()
         .beginTransaction().remove(this)
         .commitAllowingStateLoss();
     ```

     



### 申请权限接口方式

1.   `fragment.requestPermissions`方式申请权限