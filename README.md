# PLDroidCameraStreaming
PLDroidCameraStreaming 是一个适用于 Android 推送直播流的 SDK，可高度定制化和二次开发，支持 RTMP 推流以及 H.264 / AAC 硬编码。

# 内容摘要
- [功能特性](#功能特性)
- [使用方法](#使用方法)
  - [配置工程](#项目配置)
  - [权限](#权限)
  - [示例代码](#示例代码)
- [依赖库](#依赖库)
- [版本历史](#版本历史)

## 功能特性
1. 支持 MediaCodec 硬编码 
2. 支持 AAC 音频编码 
3. 支持 H264 视频编码 
4. 内置生成安全的 RTMP 推流地址
5. 支持 RTMP 协议推流
6. 支持 ARMv7a 
7. Android Min API 18 
8. 支持前后置摄像头 
9. 支持自动对焦

## 使用方法
### 项目配置
从 `releases/` 目录获取：

- pldroid-camera-streaming-xxx.jar
- armeabi-v7a/libpldroid_ffmpegbridge.so
并在项目中加入对应的 jar / so 文件的依赖关系，可参考 [PLDroidCameraStreamingDemo][1] 中的做法。

### 权限
在项目开始前，您需要在 `AndroidManifest.xml` 中添加如下权限：

```XML
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.WAKE_LOCK" />

<uses-feature android:name="android.hardware.camera.autofocus" />
<uses-feature android:glEsVersion="0x00020000" android:required="true" />
```
可参考 [PLDroidCameraStreamingDemo][2] 中的做法。

### 示例代码
1) 编写布局文件
```XML
<com.pili.pldroid.streaming.widget.AspectFrameLayout
    android:id="@+id/cameraPreview_afl"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_centerInParent="true" >

    <android.opengl.GLSurfaceView
        android:id="@+id/cameraPreview_surfaceView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="center" />
</com.pili.pldroid.streaming.widget.AspectFrameLayout>
```

2) 初始化 `Layout` 和 `View`
```JAVA
AspectFrameLayout afl = (AspectFrameLayout) findViewById(R.id.cameraPreview_afl);
GLSurfaceView glSurfaceView = (GLSurfaceView) findViewById(R.id.cameraPreview_surfaceView);
```

3) 实例化并初始化 `StreamingProfile`、`Stream`、`CameraStreamingSetting`

`streamJsonStrFromServer` 是由服务端返回的一段 JSON String，该 JSON String 描述了 `Stream` 的结构。通常，您可以使用 Pili 服务端 SDK 的 `getStream(streamId)` 方法来获取一个 `Stream` 对象，在服务端并将该对象以 JSON String 格式输出，该输出即是 `streamJsonStrFromServer` 变量的内容。例如，一段 JSON 格式的 `Stream` 内容如下：

```
{
    "id": "xxx",
    "hub": "live",
    "title": "xxx",
    "publishKey": "xxx",
    "publishSecurity": "xxx",
    // ...
}
```
然后根据 `streamJsonStrFromServer` 构造 `JSONObject` 类型的对象 `streamJson`。

```JAVA
// Get publish host from your server
String publishHost = "publish host from server";  // such as "f9zdwh.pub.z1.pili.qiniup.com"
/*
*
* You should get the streamJson from your server, maybe like this:
*
* Step 1: Get streamJsonStrFromServer from server
* URL url = new URL(yourURL);
* URLConnection conn = url.openConnection();
*
* HttpURLConnection httpConn = (HttpURLConnection) conn;
* httpConn.setAllowUserInteraction(false);
* httpConn.setInstanceFollowRedirects(true);
* httpConn.setRequestMethod("GET");
* httpConn.connect();
*
* InputStream is = httpConn.getInputStream();
* streamJsonStrFromServer = convertInputStreamToString(is);
*
* Step 2: Instantiate streamJson object
* JSONObject streamJson = new JSONObject(streamJsonStrFromServer);
*
*
* Then you can use streamJson to instantiate stream object
* Stream stream = new Stream(streamJson);
*
* */
String streamJsonStrFromServer = "{"
        + "\"id\"             :\"z1.live.55910c19fb16df0cbf00af8e\", "
        + "\"hub\"            :\"live\", "
        + "\"title\"          :\"55910c19fb16df0cbf00af8e\", "
        + "\"publishKey\"     :\"b06c7427b454762c\", "
        + "\"publishSecurity\":\"dynamic\", "
        + "..."
        + "}";

JSONObject streamJson = null;
try {
    streamJson = new JSONObject(streamJsonStrFromServer);
} catch (JSONException e) {
    e.printStackTrace();
}
        
Stream stream = new Stream(streamJson);

StreamingProfile profile = new StreamingProfile();
profile.setQuality(StreamingProfile.QUALITY_MEDIUM1)
       .setStreamAndPublishhost(stream, publishHost);

CameraStreamingSetting setting = new CameraStreamingSetting();
setting.setCameraId(Camera.CameraInfo.CAMERA_FACING_BACK)
       .setStreamingProfile(profile)
       .setCameraPreviewSize(1280, 720);
```

目前 SDK 预定义的 Quality 列表：
```JAVA
public static final int QUALITY_LOW1;
public static final int QUALITY_LOW2;

public static final int QUALITY_MEDIUM1;
public static final int QUALITY_MEDIUM2;
public static final int QUALITY_MEDIUM3;

public static final int QUALITY_HIGH1;
public static final int QUALITY_HIGH2;
public static final int QUALITY_HIGH3;
```

>若缺少上述步骤，SDK 默认使用如下设置：
```
Camera Id   : Camera.CameraInfo.CAMERA_FACING_BACK
Publish Url : Environment.getExternalStorageDirectory().getAbsolutePath() + "/pldroid-recording.mp4"
Quality     : StreamingProfile.QUALITY_MEDIUM1
Preview Size: 640X480
```

4) 实例化并初始化核心类 `CameraStreamingManager`
```JAVA
mCameraStreamingManager = new CameraStreamingManager(this, afl, glSurfaceView);
mCameraStreamingManager.onPrepare(setting);
mCameraStreamingManager.setStreamingStateListener(this);
```

您需要实现 `StreamingStateListener`，以便通过回调函数 `onStateChanged` 接收如下消息：
- STATE.PREPARING
- STATE.READY
- STATE.CONNECTING
- STATE.STREAMING
- STATE.SHUTDOWN
- STATE.IOERROR
- STATE.NETBLOCKING

>您需要注意的是，`onStateChanged` 回调函数可能被非 UI 线程调用，可参考 [CameraStreamingActivity][3] 

5) 开始推流
```JAVA
// should be invoked after getting STATE.READY message
mCameraStreamingManager.startStreaming();
```

6) 停止推流
```JAVA
mCameraStreamingManager.stopStreaming();
```

7) `CameraStreamingManager` 另外几个重要的状态周期函数 `onResume()`、`onPause()`、`onDestory()`

如果 `CameraStreamingManager` 存在于某一个 Activity 中，建议在 Activity 的 `onResume()`、`onPause()`、`onDestory()` 中分别进行调用 `CameraStreamingManager` 的周期函数，即：

```JAVA
@Override
protected void onResume() {
  super.onResume();
  mCameraStreamingManager.onResume();
}

@Override
protected void onPause() {
  super.onPause();
  mCameraStreamingManager.onPause();
  getWindow().clearFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
}
    
@Override
protected void onDestroy() {
  super.onDestroy();
  mCameraStreamingManager.onDestroy();
}
```

### 依赖库
- FFMPEG

### 版本历史
* 1.0.2 ([Release Notes][6])
  - 发布 pldroid-camera-streaming-1.0.2.jar
  - 修复无 `StreamingStateListener` 情况下的 Crash 问题
  - 修复正常启动后无 READY 消息返回问题
  - 更新 `Stream` 定义，并与服务端保持一致
  - 增加相机正常启动后即开始推流功能
  
* 1.0.1 ([Release Notes][5])
  - 发布 pldroid-camera-streaming-1.0.1.jar
  - 更新 `Stream` 类结构
  - 更新 `Stream` 的构造方式

* 1.0.0 ([Release Notes][4])
  - 发布 PLDroidCameraStreaming v1.0.0


[1]: /PLDroidCameraStreamingDemo
[2]: /PLDroidCameraStreamingDemo/app/src/main/AndroidManifest.xml
[3]: /PLDroidCameraStreamingDemo/app/src/main/java/com/pili/pldroid/streaming/camera/demo/CameraStreamingActivity.java
[4]: /ReleaseNotes/release-notes-1.0.0.md
[5]: /ReleaseNotes/release-notes-1.0.1.md
[6]: /ReleaseNotes/release-notes-1.0.2.md