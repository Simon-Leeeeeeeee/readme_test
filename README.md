# :star2:&nbsp;XCodeScanner

A new frame for decode QR code and bar code on Android. It's faster, simpler and more accurate. It's based on ZBar, compatible with Android4.0 (API14) and above.

[中国人当然看中文的](https://github.com/Simon-Leeeeeeeee/XCodeScanner/blob/master/README_CN.md)

## Catalog

* [Demo](#demo)
* [Function](#function)
* [UML](#uml)
* [Gradle](#gradle)
* [Usage](#usage)
* [Update plan](#updateplan)
* [Changelog](#changelog)
* [Author](#author)

## Demo

|Download|Effect|
|:---:|:---:|
|[click to download](http://fir.im/XCodeScanner) Or scan the following QR code<br/>[![demo](/download.png)](http://fir.im/XCodeScanner  "Scan code to download")|[![gif](/demo.gif)](http://fir.im/XCodeScanner  "Effect")|

## Function
This project is based on the [ZBar](https://github.com/ZBar/ZBar) development. It has highly encapsulated the view, camera, and decoding, and reduces the coupling between the three and increases the flexibility of configuration.

* View
    * `AdjustTextureView`, extended from `TextureView`, can correct the image according to its size, frame width and rotation angle, and solve abnormal problems such as preview image distortion.
    * `ScannerFrameView`, extended from `View`, you can customize the size, color, and animation of the scan box, the four corners, and the scan line by using xml properties or interfaces. **The specific use of reference source code comments**.
    * `MaskRelativeLayout`&`MaskConstraintLayout`，extended from `RelativeLayout`&`ConstraintLayout`, as the parentView of ScannerFrameView, used to draw the outer shadow of the scan box.

* Camera
    * Compatible with `android.hardware.camera2` and `android.hardware.Camera` API。
    * Open the camera from the child thread to prevent blocking of the main thread.
    * Use the singleton to prevent multiple instances from simultaneously operating the camera device to cause an exception.
    * According to the preview size, the image frame size, and the preview direction, the actual position of the scan frame on the image frame is calculated, can decode the scan box area only.
    * Use `TextureReader` instead of `ImageReader`, using openGL to draw the image texture, mainly to solve the problem of serious frame loss in the preview, real-time output YUV format image.

* Decoding
    * Supports specified area decoding.
    * You can specify the type of barcode that needs to be decoded.
    * The callback result contains the barcode type and barcode precision. You can configure the dirty data filter rule.

## UML

[![uml](/uml.png)](#UML  "UML")

## Gradle

Add the following code in module's `build.gradle`
```
    dependencies {
        implementation 'cn.simonlee.xcodescanner:zbar:1.1.5'
    }
```

## Usage

* **STEP.1**<br/>
Get the CameraScanner instance in the Activity's onCreate, and set the monitor to CameraScanner and TextureView.
```java
public void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   setContentView(R.layout.activity_scan_constraint);
   mTextureView = findViewById(R.id.textureview);
   mTextureView.setSurfaceTextureListener(this);
   if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
      mCameraScanner = OldCameraScanner.getInstance();
   } else {
      mCameraScanner = NewCameraScanner.getInstance();
   }
   mCameraScanner.setCameraListener(this);
}
```
* **STEP.2**<br/>
Set the width and height of the SurfaceTexture and TextureView in onSurfaceTextureAvailable, then open the camera
```java
public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
   mCameraScanner.setSurfaceTexture(surface);
   mCameraScanner.setPreviewSize(width, height);
   mCameraScanner.openCamera(this.getApplicationContext());
}
```
* **STEP.3**<br/>
Set the width and height of the image frame in openCameraSuccess, and get the ZBarDecoder instance set to CameraScanner.
```java
public void openCameraSuccess(int frameWidth, int frameHeight, int frameDegree) {
   mTextureView.setImageFrameMatrix(frameWidth, frameHeight, frameDegree);
   if (mGraphicDecoder == null) {
      mGraphicDecoder = new ZBarDecoder();//Use the parameterized construction method to specify the format for barcode.
      mGraphicDecoder.setDecodeListener(this);
   }
   //Call setFrameRect will limit the decoding area.
   mCameraScanner.setFrameRect(mScannerFrameView.getLeft(), mScannerFrameView.getTop(), mScannerFrameView.getRight(), mScannerFrameView.getBottom());
   mCameraScanner.setGraphicDecoder(mZBarDecoder);
}
```
* **STEP.4**<br/>
Get the decoded result in decodeSuccess of ZBarDecoder. You can customize the dirty data filter rule according to the returned barcode type and precision.
```java
public void decodeSuccess(int type, int quality, String result) {
   ToastHelper.showToast("[类型" + type + "/精度" + quality + "]" + result, ToastHelper.LENGTH_SHORT);
}
```
* **STEP.5**<br/>
Close camera and stop decoding in Activity's onDestroy.
```java
public void onDestroy() {
   mCameraScanner.setGraphicDecoder(null);
   mCameraScanner.detach();
   if (mGraphicDecoder != null) {
      mGraphicDecoder.setDecodeListener(null);
      mGraphicDecoder.detach();
   }
   super.onDestroy();
}
```
* **Attention.1**<br/>
Close camera in Activity's onPause.
```java
public void onPause() {
   mCameraScanner.closeCamera();
   super.onPause();
}
```
* **Attention.2**<br/>
Open camera in Activity's onRestart.
```java
public void onRestart() {
   //Some devices will call onSurfaceTextureAvailable to open the camera when they go to the foreground in the background.
   if (mTextureView.isAvailable()) {
      mCameraScanner.setSurfaceTexture(mTextureView.getSurfaceTexture());
      mCameraScanner.setPreviewSize(mTextureView.getWidth(), mTextureView.getHeight());
      mCameraScanner.openCamera(this.getApplicationContext());
   }
   super.onRestart();
}
```

## Update plan

*  Support decode local pictures.
*  Solve the problems caused by changes in the TextureView size.
*  Support environmental brightness monitoring and support open the flash.
*  Supports Zxing.
*  Support generation QR code.

## Changelog

*  V1.1.5   `2018/05/01`
   1. 解决申请权限闪退的问题。
   2. 解决魅族MX5闪退的问题。
   3. 修改`ZBarDecoder`和`TextureReader`的实现方式，降低CPU占用。
   4. 新增`DebugZBarDecoder`，继承自`ZBarDecoder`,便于示例程序进行兼容性测试。
   5. 暂停/延时解码接口从`CameraScanner`迁移到`GraphicDecoder`，`CameraScanner`可能因为异步导致暂停后继续回调`decodeSuccess`接口。
   6. 发布开源库：`cn.simonlee.xcodescanner:zbar:1.1.5`。

*  V1.1.4   `2018/04/26`
   1. 解决Android4.2退出时闪退的问题。
   2. 解决某些低端机型可能预览严重丢帧的问题。
   3. 解决`OldCameraScanner`默认没有开启解码的问题。
   4. 发布开源库：`cn.simonlee.xcodescanner:zbar:1.1.4`。

*  V1.1.3   `2018/04/25`
   1. 解决部分x86设备闪退的问题。
   2. `CameraScanner`新增`stopDecode()`和`startDecode(int delay)`接口，可暂停/延时解码。
   3. ZBar包名由`com.simonlee.xcodescanner`变更为`com.simonlee.xcodescanner`。
   4. 发布开源库：`cn.simonlee.xcodescanner:zbar:1.1.3`，`codescanner`变更为`xcodescanner`，由此带来不便的敬请谅解。
   5. 有开发者反馈部分机型存在闪退、无法解析二维码的问题，将在近期解决。

*  V1.1.2   `2018/04/24`
   1. 解决`ZBarDecoder`中设置解码格式无效的问题。

*  V1.1.1   `2018/04/16`
   1. `ScannerFrameView`增加高占比属性，可设置相对父容器高的占比。
   2. 发布开源库：`cn.simonlee.codescanner:zbar:1.1.1`。

*  V1.1.0   `2018/04/16`
   1. 重写`ZBarDecoder`，解决单线程池可能引起的条码解析延迟问题。
   2. 解决`OldCameraScanner`扫描框区域识别异常的问题。

*  V1.0.9   `2018/04/14`
   1. 解决`NewCameraScanner`扫描框区域识别异常的问题。
   2. 解决连续快速旋转屏幕时`NewCameraScanner`出现异常的问题。

*  V1.0.8   `2018/04/13`
    1. `AutoFixTextureView`更名为`AdjustTextureView`，重写图像校正方式。
    2. `Camera2Scanner`更名为`NewCameraScanner`。
    3. 新增`OldCameraScanner`实现对`Android5.0`以下的支持。
    4. 下调minSdkVersion至14。
    5. 解决前后台切换，横竖屏切换可能产生的异常。
    6. `NewCameraScanner`中取消`ImageReader`的支持。

*  V1.0.7   `2018/04/10`
    1. 调整扫描框宽高计算方式，新增`MaskConstraintLayout`布局。
    2. 优化`Camera2Scanner`，解决后台切换导致的闪退问题。

*  V1.0.6   `2018/04/09`
    1. 调整代码结构，将扫码核心从app移植到zbar中。

*  V1.0.5   `2018/03/29`
    1. 增加帧数据的最大尺寸限制，避免因过高像素导致ZBar解析二维码失败。
    2. 屏蔽ZBar对DataBar(RSS-14)格式条码的支持，此格式实用性不高，且易产生误判。

*  V1.0.4   `2018/03/27`
    1. 修改`ZBarDecoder`，修复多线程可能的空指针异常。
    2. 修改`GraphicDecoder`，EGL14替换EGL10，解决部分机型不兼容的问题；解决多线程可能的空指针异常。

*  V1.0.3   `2018/03/23`
    1. 新增`TextureReader`，通过双缓冲纹理获取帧数据进行回调，代替`ImageReader`的使用。
    2. 修改`GraphicDecoder`，handler放到子类中去操作。

*  V1.0.2   `2018/03/14`
    1. 新增抽象类`GraphicDecoder`，将条码解析模块进行抽离。
    2. 新增`ZBarDecoder`，采用ZBar解析条码，并增加解析类型及解析精度设置。
    3. 修改`ScannerFrameView`，扫描线动画由属性动画实现。
    4. 修改`Camera2Scanner`，修复释放相机可能导致的异常，增加扫描框区域设置。
    5. 其他微调。

*  V1.0.1   `2018/02/09`
    1. 新增`ScannerFrameLayout`，为`RelativeLayout`的子类，可对扫描框的位置和大小进行设置。
    2. 修改`ScannerFrameView`，可对扫描框内部进行定制。

*  V1.0.0   `2018/02/03`
    初次提交代码。

## Author

I am a Chinese who is not good at English, so this document was written using a translation tool. There should be a lot of grammatical mistakes, and I would appreciate it if you were willing to correct them.

This is my personal first open source project and I have also devoted a lot of energy to it. In the process of open source encountered many doubts and difficulties, which draw on a lot of very good technical blog. Here to pay tribute to those who are silently dedicated to open source! thank you all!

If you encounter any unusual problems such as crash, black screen, unrecognizable, unable to focus, preview dropped frames, memory leak, etc. during use, please feel free to ask us! At the same time, please attach as much as possible the device model, android version number, BUG recurring steps, exception logs, unrecognized images, etc. I will arrange for a solution as soon as possible.

If you find it useful, please  give me a **Star** to encourage me!:blush:

|Author|E-mail|Blog|WeChat|
|:---:|:---:|:---:|:---:|
|Simon Lee|jmlixiaomeng@163.com|[简书](https://www.jianshu.com/p/65df16604646) · [掘金](https://juejin.im/post/5adf0f166fb9a07ac23a62d1)|[![wechat](/wechat.png)](#Author)|
