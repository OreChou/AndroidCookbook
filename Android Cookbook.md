### #1 如何实现 App 定时关闭

一共可以有三种实现方式：

1. 使用常驻服务轮询，然后发送广播，app 接受广播。
2. 使用 Timer 定时。
3. 使用 AlarmManager 实现。(具体可搭配 PendingIntent 和 IntentFilter)

以上方式第三种最佳。前两种存在系统进入深度睡眠，无法唤醒线程的问题。[官方文档](https://developer.android.com/reference/android/app/AlarmManager)说明如下：

> These allow you to schedule your application to be run at some point in the future. When an alarm goes off, the `Intent` that had been registered for it is broadcast by the system, automatically starting the target application if it is not already running. Registered alarms are retained while the device is asleep (and can optionally wake the device up if they go off during that time), but will be cleared if it is turned off and rebooted.



### #2 实现简单拍照功能

#### Step 1 获取 CameraManager

一个 Android 系统服务用于管理检测、描述、连接相机设备（CameraDevice）。

> A system service manager for detecting, characterizing, and connecting to `CameraDevice`.

```java
CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
```

##### Step 1.1 获取操作的 CameraDevice

现在的 Android 手机通常有前置和后置摄像头，而且往往现在的后置摄像头不止一个，在拍照的时候需要具体到操作哪一个摄像头。Android Camera2 中 CameraDevice 表示一个单独连接到 Android 设备的相机，能够进行对拍照进行细粒度的控制和帧率的处理。

> The CameraDevice class is a representation of a single camera connected to an Android device, allowing for fine-grain control of image capture and post-processing at high frame rates.

但实际上，不管手机的实际摄像头的数量有多少个。通过 CameraManager#getCameraList 获取到的相机 id 的数量为 2。其原因大概在于 Camera2 将前置摄像头和后置摄像头进行了封装，返回了两个虚拟摄像头分别控制前置和后置。（Todo：那么如何实际操控单个摄像头？）

选定特定的相机 id，可以获取相机描述的静态信息，并且打开相机。

```java
CameraCharacteristics characteristics = manager.getCameraCharacteristics(cameraId);
manager.openCamera(cameraId, CameraDevice.StateCallback, Handler);
```

调用 manager#openCamera 函数不会返回成功打开的 CameraDevice ，而是会有一个回调 CameraDevice.StateCallback。在这个回调中需要去实现三种钩子方法，分别是：

1. onOpend: The method called when a camera device has finished opening.
2. onDisconnected: The method called when a camera device is no longer available for use.
3. onError: The method called when a camera device encountered a serious error.  

在 onOpend 这个钩子方法中，我们可以获取到打开的 CameraDevice。

##### Step 1.2 获取合适的图像大小

Android Camera 开发中，尺寸和方向的问题要搞清楚。

尺寸是指：

1. 相机预览帧的尺寸。
2. 相机拍摄（保存）帧的尺寸。
3. Android 显示的控件尺寸。

方向是指：

1. 相机预览帧的方向。
2. 相机拍摄（保存）帧的方向。
3. Android 自身的方向。

相机这种硬件设备可以提供拍摄帧和预览帧的尺寸。对于一种照片编码格式，首先获取设备能够支持的所有尺寸大小。

```java
// The available stream configurations that this camera device supports
StreamConfigurationMap configMap = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
Size[] sizeList = configMap.getOutputSizes(ImageFormat.JPEG);
```

因为相机应用通常全屏显示，所以 Android 显示的最大尺寸通常是显示尺寸。

```java
Point displaySize = new Point();
activity.getWindowManager().getDefaultDisplay().getSize(displaySize);
```

如果应用需要所拍即所得，即预览尺寸与拍摄的图片尺寸一致，那就需要计算统一预览帧、拍摄帧和 Android 显示尺寸。这个大致的思路是确定以哪个尺寸为基准，计算宽高比，然后确定另外两个尺寸的大小。

下面讨论方向。

Android 屏幕的坐标原点在左上角，水平向右是 x 轴正向，竖直向下是 y 轴正向。而 Android 设备中后置 Camera 通常是在屏幕右上角（视线正对屏幕），并且是水平安装的。对比如下：

![back_camera_coordiante](.\images\back_camera_coordiante.png)

根据后置 Camera 的坐标参考系和方向，可以推断 Android 设备中前置 Camera 。前置有一次影像的反向，这里有 180 度，再加上一样的水平安装，有一个 90 度。所以前置 Camera 的方向是 270 度。坐标原点水平向右是 y 轴，坐标原点竖直向上是 x 轴。

应用可以获取到屏幕旋转的方向和利用 Camera2 获取相机传感器的方向。

```java
int displayOrientation = activity.getWindowManager().getDefaultDisplay().getRotaion();
// Clockwise angle through which the output image needs to be rotated to be upright on the device screen in its native orientation.
int sensorOrientation = characteristics.get(CameraCharacteristics.SENSOR_ORIENTATION);
```

由此后置摄像头，当 Android 处于横屏拍摄时，预览图像不需要旋转。但当 Android 处于竖屏拍摄时，预览图像需要旋转 90 度。

应用可以通过简单的公式计算，拍摄的图像在预览时需要旋转的角度。

```java
int degree = 0;
if (facing == CameraCharacteristics.LENS_FACING_BACK) {
    degree = (displayOrientation - 90 + 360) % 360;
} else if (facing == CameraCharacteristics.LENS_FACING_FRONT) {
    degree = (displayOrientation - 270 + 360) % 360；
}
```

#### Step 2 开启预览

首先创建拍照请求的 Builder，即 CaptureRequest.Builder 。获取该类的实例需要调用 CameraDevice#createCaptureRequest 方法。请求有不同的类型，一共有 6 种不同的类型：

1. TEMPLATE_MANUAL: A basic template. All automatic control is disabled (AE/AF/AWB). 
2. TEMPLATE_PREVIEW
3. TEMPLATE_RECORD
4. TEMPLATE_STILL_CAPTURE
5. TEMPLATE_VIDEO_SNAPSHOT
6. TEMPLATE_ZERO_SHUTTER_LAG

上诉 6 种请求类型都有不同的适用场景。

然后创建管理拍照的 Session，即 CameraCaptureSession。Session 的创建方法是 CameraDevice#createCaptureSession() ，该方法具有 3 个参数，分别是：

1. List<Surface>: 应用需要输入到的 Surface 列表。
2. CameraCaptureSession.StateCallback: Session 状态变化的回调。
3. Handler: 处理回调的线程。

CameraCaptureSession.StateCallback 有 7 种状态的回调，分别是：

1. onConfigured: Finished configuring itself, session can start processing capture requests.
2. onConfigureFailed: Session cannot be configured.
3. onReady: Session has no more capture request to process.
4. onSurfacePrepared: The buffer pre-allocation for an output Surface is complete.
5. onCaptureQueueEmpty: Camera input capture queue becomes empty, it ready to accept the next request.
6. onActive: Session starts actively processing capture requests.
7. onClosed: Session is closed.

另一个需要实现的是拍照状态回调，即 CameraCaptureSession.CaptureCallback。

然后就通过 CameraCaptureSession#setRepeatingRequest 方法开启预览。

#### Step 3 处理拍照

应用可以通过 CaptureRequest.Builder 去设置拍照相关的参数，通过 CameraCaptureSession.CaptureCallback 去处理拍照的结果 CaptureResult。在编写相关的代码的时候，需要理清楚拍照过程发生了什么，然后将动作进行拆分。数码相机在拍照的过程涉及到 3A 算法，即：

1. AF：Auto focus，自动对焦
2. AE：Auto exposure，自动曝光
3. AWB：Auto white balance，自动白平衡

这里，文章暂不讨论 AWB，而 AF 和 AE 是需要去讨论的。一般来说，拍照的动作有对焦、调整曝光、拍摄。首先定义一些状态，指示拆分动作的状态，分别如下：

1. STATE_PREVIEW: 拍照按钮并未按下，摄像头正在预览。
2. STATE_WAITING_LOCK: 拍照按钮按下，正在等待对焦完成。
3. STATE_WAITING_PRECAPTURE: 正在等待曝光完成。
4. STATE_WAITING_NON_PRECAPTURE: 正在等待非曝光的其他步骤。
5. STATE_PICTURE_TAKEN: 等待和处理拍照。

##### Step 3.1 对焦

按钮按下之后，开始对焦的代码如下：

```java
mCaptureRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER, CameraMetadata.CONTROL_AF_TRIGGER_START);
// 
mState = STATE_WAITING_LOCK;
mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, mBackgroundHandler);
```

处理对焦结果的代码如下：

```java
private void process(CaptureResult result) {
    case STATE_PREVIEW: {
        break;
    }
    case STATE_WAITING_LOCK: {
        Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
        // 未对焦
        if (afState == null) {
            captureStillPicture();
        } 
        // 对焦完成：有两种对焦成功或对焦失败
        else if (CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState || CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED) {
            Integer aeState = result.get(CaptureResult.CONTRIL_AE_STATE);
            // CONTROL_AE_STATE can be null on some devices
            // 曝光已经设置好了
            if (aeState == null || aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                mState = STATE_PICTURE_TAKEN;
                captureStillPicture();
            }
            // 需要做一些拍摄前自动曝光外其它的设置
            else {
                runPrecaptureSequence();
            }
        }
        break;
    }
    case STATE_WAITING_PRECAPTURE: {
        Integer aeState = result.get(CaptureResult.CONTRIL_AE_STATE);
        /**
       	 * 第一个状态的意思:
         * Once PRECAPTURE completes, AE will transition to CONVERGED
         * or FLASH_REQUIRED as appropriate. This is a transient
         * state, the camera device may skip reporting this state in
         * capture result.
         */
        if (aeState == null || aeState == CaptureResult.CONTROL_AE_STATE_PRECAPTURE || aeState == CaptureResult.CONTROL_AE_STATE_FLASH_REQUIRED) {
            mState = STATE_WAITING_NON_PRECAPTURE;
        }
        break;
    }
    case STATE_WAITING_NON_PRECAPTURE: {
        Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
        // 从上面那个状态改变过来，会成为 CONVERGED 或者 FLASH_REQUIRED
        if (aeState == null || aeState != CaptureResult.CONTROL_AE_STATE_PRECAPTURE) {
        	mState = STATE_PICTURE_TAKEN;
            captureStillPicture();
        }
        break;
    }
}
```

##### Step 3.2 拍照

上面的处理回调中，实际拍照的函数为 captureStillPicture，其实现如下：

```java
// 区别 preview 的 builder，新生成一个用以实际拍照的 builder
final CaptureRequest.Builder captureBuilder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE);
captureBuilder.addTarget(mImageReader.getSurface());
captureBuilder.set(CaptureRequest.CONTROL_AF_MODE,CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
// set property orientation
...
// set property orientation finish
// 设置拍照完成后的回调
CameraCaptureSession.CaptureCallback CaptureCallback = new CameraCaptureSession.CaptureCallback() {
    ...
}
// 拍照的时候，停止连续取景 （应该是要减少 RequestQueue 的入队数量？）
mCaptureSession.stopRepeating();
mCaptureSession.abortCaptures();
// 捕获照片
mCaptureSession.capture(captureBuilder.build(), CaptureCallback, null);
```

##### Step 3.3 取消对焦，继续预览

这部分就是上面的 “设置拍照完成后的回调” ，具体如下：

```java
@Override
public void onCaptureCompleted(@NonNull CameraCaptureSession session, @NonNull CaptureRequest request, @NonNull TotalCaptureResult result) {
    // 取消自动对焦
    mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER, CameraMetadata.CONTROL_AF_TRIGGER_CANCEL);
    setAutoFlash(mPreviewRequestBuilder);
    mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback, mBackgroundHandler);
    // After this, the camera will go back to the normal state of preview.
    mState = STATE_PREVIEW;
    // 预览又开始持续取景
    mCaptureSession.setRepeatingRequest(mPreviewRequest, mCaptureCallback, mBackgroundHandler);
}
```



### #3 Android 存储系统

Android 存储系统大致有 2 种，分类如下：

#### 3.1 内部存储

##### 3.1.1 私有文件

用以保存应用的私有文件，其他应用无法访问，不需要额外权限。

```java
// write
fos = openFileOutput(filename, Context.MODE_PRIVATE);
// read
fis = openFileInput(filename);
```

##### 3.1.2 静态文件

文件路径：res/raw

使用方法：Resources#openRawResource

##### 3.1.3 缓存数据

使用方法：Context#getCacheDir

#### 3.2 外部存储

指一切可移除的存储介质（SD 卡），和不可移除的存储介质（手机内置的存储器）。需要额外权限：

```java
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

##### 3.2.1 公共文件

对于在应用中产生的多媒体类型的文件，如音乐、图片、铃声等，一般应该保存在外置存储中对应的公共目录下，如/Music、/Pictures、/Ringtones，这样方便和其他的应用共享这些文件。同时，系统的媒体扫描器也能正确地对这些文件进行归类。通过 Environment#getExternalStoragePublicDirectory 调用。Android 定义好的公共文件类型如下：

1. DIRECTORY_MUSIC
2. DIRECTORY_PICTURES：图片
3. DIRECTORY_MOVIES
4. DIRECTORY_DCIM：照片（Digital Camera IMages）
5. DIRECTORY_DOWNLOADS
6. DIRECTORY_DOCUMENTS
7. DIRECTORY_RINGTONES：铃声
8. DIRECTORY_ALARMS：闹钟
9. DIRECTORY_NOTIFICATIONS：通知提示音
10. DIRECTORY_PODCASTS

可能会不存在，所以在使用前最好先检查。

##### 3.2.2 私有文件

外部存储中应用私有的存储文件，使用 Context#getExternalFilesDir 和 Context#getExternalFilesDirs 调用。对于即有 SD 卡又又内置存储器的设备，前一个方法只会返回内置存储器中的文件，而后一个方法能够返回两者。

根目录的路径：Android/data/< package >/files/

##### 3.2.3 缓存文件

使用方法：Context#getExternalCacheDir 和 Context#getExternalCacheDirs

根目录的路径：Android/data/< package >/cache/



### #4 让拍摄的照片出现在照片浏览器

Android 知道公共文件中存在哪些多媒体文件靠 MediaStore。官方文档的解释如下：

> The contract between the media provider and applications. Contains definitions for the supported URIs and columns.
>
> The media provider provides an indexed collection of common media types, such as `Audio`, `Video`, and `Images`, from any attached storage devices. Each collection is organized based on the primary MIME type of the underlying content; for example, `image/*` content is indexed under `Images`. The `Files` collection provides a broad view across all collections, and does not filter by MIME type.

即 MediaStore 提供了公共文件夹中所有媒体文件的索引，能够方便快速查找到相应的文件。

当应用程序将一些多媒体文件存储到公共文件夹中时，Meida Scanner Service 不会立即去扫描这些文件，所以在多媒体浏览器中无法显示刚刚存储的文件。通常刷新扫描的方法有如下三种：

1. 操作 MediaStore 类。
2. 发送广播更新 MediaStore。
3. 使用 MediaScannerConnection

这里，使用 MediaScannerConnection，它的介绍如下：

> MediaScannerConnection provides a way for applications to pass a newly created or downloaded media file to the media scanner service. The media scanner service will read metadata from the file and add the file to the media content provider. The MediaScannerConnectionClient provides an interface for the media scanner service to return the Uri for a newly scanned file to the client of the MediaScannerConnection class.

使用 MediaScannerConnection 的原因只有一个，简单。它提供了一个静态方法可以自动帮你建立与关闭与 Media Scanner Service。在使用的时候只需要传入当前环境的上下文，文件所在的位置，还有文件的类型就可以了。再扫描完成之后，如果需要进行一些操作的话，可以设置一个回调。方法的定义如下：

```java
public static void scanFile (Context context, 
                String[] paths, 
                String[] mimeTypes, 
                MediaScannerConnection.OnScanCompletedListener callback)
```

### #5 Android 沉浸式显示

沉浸模式（Immersive mode）

[参考博文地址](https://blog.csdn.net/chazihong/article/details/70228933)