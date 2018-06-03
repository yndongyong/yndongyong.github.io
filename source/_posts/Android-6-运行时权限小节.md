---
title: Android 6 运行时权限小节
date: 2017-08-15 21:31:00
tags:
---
## Android 6 运行时权限小结

用了很多的第三方权限框架，都不是很顺手。由于最近项目里使用的框架存在一些问题，于是自己阅读了官方的文档，梳理了运行时权限相关的知识点，并形成了一个工具类供复用。

### 权限的几种分类
在Android 6上系统将权限分为两种，一种是normal权限，另一种是dangerous权限。


对于normal权限 不会直接或者间接的涉及用户的隐私数据。申请这类权限直接在AndroidManifest.xml文件中声明。

dangerous权限，将会直接涉及到用户的隐私、敏感数据，在使用户的这类数据时，需要在程序运行时，向用户申请，只用当用户同意时才可以读取、访问用户的相关数据。

运行时申请的危险权限还需要在AndroidManifest.xml文件中，如果不声明的话，申请权限系统将会立刻拒绝。
如果使用运行时权限申请，而恰好AndroidManifest.xml文件中，一个权限都为声明，系统就会抛出错误，造成闪退，这应该是系统的一个强制性措施吧。

#### 权限分组
		//读写 日历
		CALENDAR = new String[]{
                Manifest.permission.READ_CALENDAR,
                Manifest.permission.WRITE_CALENDAR};
		//摄像头
        CAMERA = new String[]{
                Manifest.permission.CAMERA};
		 // 读取联系人 写入联系人
        CONTACTS = new String[]{
                Manifest.permission.READ_CONTACTS,
                Manifest.permission.WRITE_CONTACTS,
                Manifest.permission.GET_ACCOUNTS};
			//位置信息
        LOCATION = new String[]{
                Manifest.permission.ACCESS_FINE_LOCATION,
                Manifest.permission.ACCESS_COARSE_LOCATION};
			//麦克风
        MICROPHONE = new String[]{
                Manifest.permission.RECORD_AUDIO};
			//读取电话状态 打电话 读写通话记录 处理来电
        PHONE = new String[]{
                Manifest.permission.READ_PHONE_STATE,
                Manifest.permission.CALL_PHONE,
                Manifest.permission.READ_CALL_LOG,
                Manifest.permission.WRITE_CALL_LOG,
                Manifest.permission.USE_SIP,
                Manifest.permission.PROCESS_OUTGOING_CALLS};
		 // 传感器
        SENSORS = new String[]{
                Manifest.permission.BODY_SENSORS};
		 // 读写短信 收发短信
        SMS = new String[]{
                Manifest.permission.SEND_SMS,
                Manifest.permission.RECEIVE_SMS,
                Manifest.permission.READ_SMS,
                Manifest.permission.RECEIVE_WAP_PUSH,
                Manifest.permission.RECEIVE_MMS};
		//读写存储
        STORAGE = new String[]{
                Manifest.permission.READ_EXTERNAL_STORAGE,
                Manifest.permission.WRITE_EXTERNAL_STORAGE};



### 值得注意的地方
- 特别需要targetSDKVersion的版本指定。当targetSDKVersion<=22时，无论程序是跑在Android 6以及高版本的手机上，还是还是 5版本的手机上，运行时权限申请都将被通过。只用targetSDKVersion>=23版本时，运行时权限才起作用。

-  用户上一分钟授予了某一个权限，并不代表用户这一分钟依然具有这一个权限！！！


### Android 6 和 Android O上的差异

在Android 6 系统版本上，如果申请了某一个组内的其中给一个权限，该组内所有的权限都被授予了程序。例如，用户授予程序READ_EXTERNAL_STORAGE 的权限，组内的WRITE_EXTERNAL_STORAGE也也一同被授予，如果程序在得到了READ_EXTERNAL_STORAGE权限之后去写入SD数据，直接写入是可行的，直到Android o，这种机制被系统推翻了。在Android O中程序程序在得到了READ_EXTERNAL_STORAGE权限之后需要往SD卡写入数据时，必须主动去申请WRITE_EXTERNAL_STORAGE
权限，此时，系统不会弹出让用户授权的对话框，系统将立刻授予程序该程序，如果不申请权限直接就写入的话，系统会抛出异常。

### Util类的使用

#### step 0 

需要Activity 或者fragment实现util类中的回调PermissionCallback
	
	 interface PermissionCallback {
        //用户授予了所有的权限
        void onPermissionsGranted(int requestCode, String permissions[]);
        //用户拒绝了权限申请
        void onPermissionsDenied(int requestCode, String permissions[]);

    }


#### step 1 
检查是否需要向用户解释为什么需要申请该权限，只有当这个权限被用户拒绝授予过，而且未勾选“不再询问”的复选框时，该方法的返回值才会为true
	
	PermissionCompat.shouldShowRequestPermissionRationale(
		this，
		PermissionCompat.STORAGE）

#### step2

当上一个步骤的返回值为true时，表明用户已经拒绝授予某个权限，且未勾选不在询问的复选框，此时程序需要向用户解释说明为什么需要这个权限，例如，向用户弹个框解释，当用户点击了确定按钮时，去申请去权限。
如果上一个步骤的返回值为false，则可以直接去申请权限

	PermissionCompat.requestPermissions(this, requestCode,PermissionCompat.STORAGE);

#### step3

实现Activity 或者fragment的onRequestPermissionsResult方法，并使用Util类中的方法分发返回结果。

	@Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        PermissionCompat.onRequestPermissionsResult(this,requestCode,permissions,grantResults);
    }
    
    
#### step 4   


  - 当用户授予了权限时，在PermissionCallback的onPermissionsGranted回调方法中处理要做的工作。
  - 当用户拒绝了某一个权限授予时，会回调PermissionCallback的onPermissionsDenied方法，可以在这个方法中检测，用户是否勾选了“不再询问”的复选框，如果用户勾选了，需要向用户解释需要改权限的理由，同时引导用户跳转到《设置》下的应用程序权限设置，给用户一个设置再次开启权限的入口。例如弹个框，当用户点击确定按钮时，跳转到用程序权限设置界面。
  
		PermissionCompat.showAppSetting(MainActivity.this,requestCode);

#### step 5
	
在Activity 或者fragment的onActivityResult方法回调用处理应用权限的开启情况。 

需要注意的是，当用户在应用的权限设置界面打开某一个程序之后跳转到当前应用程序时（一直按返回键）会直接回调到onActivityResult，如果存在用户关闭某一个权限的情况下，再跳转回当前应用程序时（一直按返回键），会先导致当前应用的Activity重启，最后依然会回调到onActivityResult方法。
  
  
  demo地址
  
  
  
		

		
		
			
	
			
		
	
	
	
	
