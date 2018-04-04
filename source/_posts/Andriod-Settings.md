---
title: Andriod Settings
date: 2018-04-04 23:24:24
tags: Android 系统属性
---

### 前言

> 公司开发 Android Launcher 项目，需要设置一些系统的属性，比如说修改系统语言设置，修改输入法，修改时间格式，修改时区等等，因为是一些系统属性，配置的时候都一些注意的要点。

<!-- more -->

#### 修改系统语言设置

> 修改系统语言比较特殊，需要用到一个叫作 `ActivityManagerNative` 类，这个类在 `layoutlib.jar` 中，**它与 android.jar 是同一级别的，不参与编译进 apk 中，所以需要自己导入**。

- android studo 的导入方法如下：

```groovy
android {
  ...需写在 android 之后
}

dependencies {
  ...
    provided files(getLayoutLibPath())
  ...
}

tasks.withType(JavaCompile) {
  options.encoding = "UTF-8"
}

/**
 * get layoutlib.jar path
 * android.os.SystemProperties need it
 * must called after "android" definition
 */
def getLayoutLibPath() {
	return "${android.getSdkDirectory().getAbsolutePath()}" + "/platforms/" + android.compileSdkVersion + "/data/layoutlib.jar"
}
```

接下来就是修改语言的代码

```java
IActivityManager iManager;
try {
      iManager = ActivityManagerNative.getDefault();
      Configuration localConfiguration = iManager.getConfiguration();
      localConfiguration.locale = mInfos.get(position).getLocale();
      iManager.updateConfiguration(localConfiguration);
      BackupManager.dataChanged("com.android.providers.settings");
      finish();
      } catch (RemoteException e) {
      e.printStackTrace();
}
```

- 这里的locale对象，代表了一个特定的地理、政治和文化地区。在操作 Date、Calendar等表示日期/时间的对象时，经常会用到。

- 这里用到了 `Configuration`，对其进行了修改，就需要声明权限

  ```xml
  <uses-permission android:name="android.permission.CHANGE_CONFIGURATION"/>
  ```

- 同时因为这是系统级别的权限，普通的第三方应用没有权限使用，即使声明了该权限也无法获取该权限，需要声明该应用为系统应用：需要在 AndroidManifest.xml 添加：

  ```xml
  android:sharedUserId="android.uid.system"
  ```

#### Android Settings

> 其它许多系统属性的修改都在 Settings 应用当中进行设置，比如 wifi，蓝牙状态，等一些相关的系统属性。这些属性的数据主要保存在数据库 settings 中，而 settings 保存的位置在：/data/data/com.android.providers.settings/databases/settings.db。

我从自己机器中拉了一个出来，上传了坚果云，可以下载了看看。

[settings数据库坚果云地址](https://www.jianguoyun.com/p/DWIxMcYQ2uTiBhjBiUs)

主要的操作就是针对以下三张表：

|        | 表名            | URI                       | 备注                                                         |
| ------ | --------------- | ------------------------- | ------------------------------------------------------------ |
| secure | Settings.Secure | content://settings/secure | 安全性的用户偏好系统设置，第三方APP有读没有写的权限          |
| global | Settings.Global | content://settings/global | 所有的编号设置，对系统的所有用户公开，第三方APP有读没有写的权限 |
| system | Settings.system | content://settings/system | 包含各种各样的用户偏好系统设置                               |

因为这是一个本应用外的数据库，不能跨进程和包直接访问，就需要用到了内容提供者`ContentProvider `和内容解析者`ContentResolver`。<br>[跨进程通信的ContentProvider](www.baidu.com)<br>

简短的概括一下就是，如果跨进程访问数据，需要将这个数据通过`ContentProvider`共享出来，然后通过`ContentResolver` 访问到，执行增删改查的操作。`ContentProvider`是一个抽象类，也就是继承实现它，而 Android 中对 `settings.db` 实现的就是 `SettingsProvider`，当中封装了对 `settings.db` 的操作，而`framework`有一个类对使用`SettingsProvider`进行了封装。

#### 以Secure表为例

三个表的使用都差不多，就以`secure`为例。

##### 获取当前输入法

```java
String defaultMethodId = Settings.Secure.getString(getContentResolver(), Settings.Secure.DEFAULT_INPUT_METHOD);
```

##### 修改当前输入法

> 先在系统的配置信息中添加该输入法，通过 Settings.Secure.ENABLE_INPUT_METHODS 添加，如果有多个输入法，这些字符串间以分号分隔，再将输入法设置成默认输入法

```java
Settings.Secure.putString(getContentResolver(), Settings.Secure.ENABLED_INPUT_METHODS, newInputMethodId);
Settings.Secure.putString(getContentResolver(), Settings.Secure.DEFAULT_INPUT_METHOD, newInputMethodId);
```

##### 获取日期格式

```java
Settings.System.getString(getContentResolver(), Settings.System.DATE_FORMAT)
```

##### 添加权限

```xml
<uses-permission android:name="android.permission.SET_TIME"/> <!--允许程序设置系统时间-->
<uses-permission android:name="android.permission.SET_TIME_ZONE" /> <!--允许程序设置系统时区-->
<uses-permission android:name="android.permission.WRITE_SECURE_SETTINGS"/> <!--允许应用程序读取或写入安全系统设置-->
<uses-permission android:name="android.permission.WRITE_SETTINGS" /> <!--允许应用程序读取或写入安全系统设置-->
```

##### Condition

- 在获取时区的时候，Calendar对象需要重新获取，不然获得时区值不会变
- 在 Android6.0 之后，申请 WRITE_SETTINGS 的权限，会出现闪退的情况，异常为：java.lang.IllegalArgumentException : You can not keep your settings in the secure settings.
  - 原因：在 Android6.0 以后，WRITE_SETTINGS 权限的保护等级已经由原来的 dangerous 升级为signatrue, 意味着APP需要用系统签名或者成为系统预装软件才能够申请该权限，并且还需要提示用户跳转到修改系统的设置界面去授予此权限。
  - 解决方案：要想申请该权限，apk必须要打包，签名打包，debug模式是不能申请该权限的