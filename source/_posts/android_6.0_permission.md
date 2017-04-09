---
title: 6.0以下国产手机权限处理
date: 2017-04-09 17:24:34
categories: Android
tags: []
---
### 前言
Android是在6.0加入的权限机制，但是不少国产手机比如华为小米等，在6.0之前的设备已经在设置里面有权限开关。调皮的某个用户会关掉这个权限，然后去拍照什么的，直接崩掉了。之前大牛郭霖在csdn做了一期权限机制的教学直播，结束后我问了他这个问题，他的回答是简单try catch即可无需深究。那我想结合实际项目研究下这个问题。
### Android权限机制简介
Android是在6.0之前只需要在menifest注册，用户安装app会有一个权限列表，类似一份协议表示用户对此已经知晓。但是实际中哪个用户会认真去看呢？导致很多app滥用权限给用户造成风险，于是在6.0后Android推出9组危险权限，要求开发者不仅要在menifest注册还要动态申请权限，比如调用拍照会弹出权限提示，只有用户自己点了确定才能继续拍照。
### 测试
了解了Android6.0权限机制后，我们用系统的权限API对国产手机测试一下。

测试设备：华为，系统版本4.4
测试内容：拍照，通话，录音
测试API：
1. ContextCompat.checkSelfPermission：检查是否有某个权限
2. requestPermission：主动申请某个权限，看看回调结果
测试代码：
```java
String[] permissions = new String[]{Manifest.permission.CALL_PHONE,Manifest.permission.CAMERA,Manifest.permission.RECORD_AUDIO};
                for(int i=0;i<permissions.length;i++){
                    String permission = permissions[i];
                    int check = ContextCompat.checkSelfPermission(MyActivity.this, permission);
                    if(check== PackageManager.PERMISSION_GRANTED){
                        Log.d("dml","权限通过");
                    }else{
                        Log.d("dml","无权限");
                    }
                    requestPermission(permissions, new OnPermissionCallback() {
                        @Override
                        public void onGranted() {
                            Log.d("dml","权限申请成功");
                        }
                        @Override
                        public void onDenied(List<String> deniedPermissions) {
                            Log.d("dml","权限被拒绝");
                        }
                    });
                }
```

先进入设置关闭这个app的电话，照相和录音权限

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-67e92d1524ed9189.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行app，发现并没有弹出让用户开关权限的Dialiog，并且检查权限返回权限通过。看看日志：

D/dml: android.permission.CALL_PHONE权限通过
D/dml: android.permission.CALL_PHONE权限申请成功
D/dml: android.permission.CAMERA权限通过
D/dml: android.permission.CAMERA权限申请成功
D/dml: android.permission.RECORD_AUDIO权限通过
D/dml: android.permission.RECORD_AUDIO权限申请成功

### 结论
检查权限和申请权限的api在华为4.4手机 完全失效，看下源码也不难得到印证：


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-53550ebf0c3d994f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当运行设备在23以下也就是6.0以下的设备时，权限申请其实是通过PackageManager.checkPermission()来进行，看下这个方法：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-c52be3332025ae70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


是个抽象方法，不过注释看出：只是判断你的apk也就是你的Manifest.xml有没有注册这个权限，有那么就返回true。

### 实际开发的问题
1. 如果不去做检测，直接调用会如何呢？

经过测试以华为Android4.4的手机为例，关闭上述三项权限进行操作，发现系统都弹出了没有权限的Toast，不一样的是：
打电话：不能进入拨号界面，也没有闪退，没有异常输出
打开摄像头：直接闪退，有异常输出
录音：没有闪退，没有异常，录音开启了但是没有数据（如果我们直接用了这些空数据，极大可能崩溃）

厂商ROM只是给出了无权限提示，但我们要做的就是保证app不能崩溃！


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-395f74d3d24fc4fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###  解决方案
1. 在代码中我们针对Android6.0的权限检测（ContextCompat.checkSelfPermission和requestPermission）按照正常的写，保证在Android6.0以上的设备正常运行。
2. 然后在具体的操作比如拨号，拍照或者录音，加一层tyr catch，能捕获到异常最好，不能捕获到的话继续第三步。
3. 对具体机型我们加入if判断，对操作数据做合法性判断，比如录音生成的数据，下面以华为为例：

还是上面的代码，我们对三种操作都加上try catch：
```java

public class MyActivity extends BaseActivity {
    private Button btn1,btn2,btn3;
    private Camera camera;
    private AudioRecord mRecorder;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity);
        btn1 = (Button) findViewById(R.id.button);
        btn2 = (Button) findViewById(R.id.button2);
        btn3 = (Button) findViewById(R.id.button3);
        btn1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try{
                    call();
                }catch (Exception e){
                    Log.e("dml","exception = " + e.getMessage());
                }
            }
        });
        btn2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try{
                    openCamera();
                }catch (Exception e){
                    Log.e("dml","exception = " + e.getMessage());
                }
            }
        });
        btn3.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try{
                    startRecord();
                }catch (Exception e){
                    Log.e("dml","exception = " + e.getMessage());
                }
            }
        });
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        if(mRecorder!=null){
            mRecorder.stop();
        }
    }
    private void call(){
        Intent intent = new Intent();
        intent.setData(Uri.parse("tel://1212121212"));
        intent.setAction(Intent.ACTION_CALL);
        startActivity(intent);
    }
    private void openCamera(){
        camera = Camera.open(0);
    }
    private void startRecord(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                int bufferSize = AudioRecord.getMinBufferSize(8000, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT);
                mRecorder = new AudioRecord(MediaRecorder.AudioSource.MIC, 8000, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, bufferSize * 2);
                mRecorder.startRecording();
                byte[] tempBuffer = new byte[bufferSize];
                while(true){
                    int recordCountSize = mRecorder.read(tempBuffer, 0, bufferSize);
                }
            }
        }).start();
    }
}
```
#### 拍照
这次拍照没有闪退了，并且捕获到了异常,那直接在catch里面弹出Dialog就行了

E/dml: exception = Fail to connect to camera service

#### 拨号
但是拨号失败却没有任何异常，如何处理呢？我们可以在设置一个布尔值标志位(isNewCal)，监听到拨打电话的状态设为true，在拨出电话延迟500ms检测这个标志位，如果还是false说明拨打失败，然后排除移动网络异常，其他情况全部算作没有权限：

```java

TelephonyManager tm = (TelephonyManager)getSystemService(Service.TELEPHONY_SERVICE);
        tm.listen(new PhoneStateListener(){
            @Override
            public void onCallStateChanged(int state, String incomingNumber) {
                // TODO Auto-generated method stub
                super.onCallStateChanged(state, incomingNumber);
                if(state==TelephonyManager.CALL_STATE_OFFHOOK){
                    isNewCall = true;
                }
            }
        }, PhoneStateListener.LISTEN_CALL_STATE);
```
第二种方法处理拨号权限问题，就是换一种Intent，不要直接拨号而是跳到系统拨号界面，让用户自己点击拨号。
```java

private void jumpToDial(){
        Intent intent = new Intent(Intent.ACTION_DIAL,Uri.parse("tel:1234567"));
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
```

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-25a95d0bb121150f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 录音
录音代码加了try catch，但是关闭权限没有捕获任何异常，但是可以看出录出来的数据缓冲区全部为0:

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-ea64bb3b7aadccfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们打开权限再看下：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/4943911-17298c20bd46f872.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因此我们可以取录音缓冲区的部分数据，如果全部为零，那么当作没权限立即通知录音并给出Dialog提示，这里我取缓冲区的前20个byte：
```java
    private void startRecord(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                int bufferSize = AudioRecord.getMinBufferSize(8000, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT);
                mRecorder = new AudioRecord(MediaRecorder.AudioSource.MIC, 8000, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, bufferSize * 2);
                mRecorder.startRecording();
                byte[] tempBuffer = new byte[bufferSize];
                byte[] checkBuffer = null;
                int valuePlus = 0;
                while(true){
                    int recordCountSize = mRecorder.read(tempBuffer, 0, bufferSize);
                    if(checkBuffer==null){
                        checkBuffer = new byte[20];
                        System.arraycopy(tempBuffer,0,checkBuffer,0,20);
                        for(int value:checkBuffer){
                            valuePlus+=value;
                            Log.d("dml","value = " + value);
                        }
                        if(valuePlus==0){
                            if(mRecorder!=null){
                                mRecorder.stop();
                                Log.d("dml","stop record!");
                            }
                            break;
                        }
                    }
                }
            }
        }).start();
    }
```
### 总结
**try catch ＋ 数据异常判断的思路**，上面我主要针对华为手机做了部分权限的适配，不过在公司测试来看也兼容魅族三星，如果除了大部分主流机型还有个别不能兼容怎么办？我的建议是让这些调皮的用户自己玩去吧。