> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/sinat_18179367/article/details/105999807)

Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）
===============================================================================

随手笔记系列，内容较多较繁琐，用于结合目录快速查询

本文将通过 AudioManger 的 setMastermute Api 的通信传递实现逻辑，自上而下梳理一下调用顺序和各模块相关初始化，最终到的 audio_hw.c 来实现静音

### 文章目录

*   [Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）](#Android_Q_Audio_AudioFlingerAudio_hal_AudioDevice__0)
*   *   [一、Framework 层通信](#Framework_9)
    *   *   [1.1 AudioManeger](#11_AudioManeger_11)
        *   *   [1.1.1 AudioManager setMasterMute 逻辑](#111_AudioManager_setMasterMute_20)
            *   [1.1.2 AudioManger 的初始化](#112_AudioManger_35)
            *   [1.1.3 AudioManager 的获取](#113_AudioManager_74)
        *   [1.2 AudioService](#12_AudioService_96)
        *   *   [1.2.1 SystemServer 启动 AudioService](#121_SystemServerAudioService_98)
            *   [1.2.2 AudioService 的初始化和 onStart](#122_AudioServiceonStart_154)
            *   [1.2.3 AudioService 的获取](#123_AudioService_179)
            *   [1.2.3 AudioService setMasterMute 逻辑](#123_AudioService_setMasterMute_197)
        *   [1.3 AudioSystem](#13_AudioSystem_223)
        *   *   [1.3.1 AudioSystem setMasterMute](#131_AudioSystem_setMasterMute_227)
            *   [1.3.2 Zygote.rc 解析](#132_Zygoterc_236)
            *   [1.3.3 AndroidRuntime.cpp 初始化](#133_AndroidRuntimecpp__325)
            *   [1.3.4 android_media_AudioSystem 注册](#134_android_media_AudioSystem__377)
            *   [1.3.5 AudioSystem::setMasterMute](#135_AudioSystemsetMasterMute_405)

一、Framework 层通信
---------------

### 1.1 AudioManeger

客户端通过 AudioManager 调用 setMasterMute

```
mAudioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
mAudioManager.setMasterMute(mute, flags);
```

#### 1.1.1 AudioManager setMasterMute 逻辑

```
@UnsupportedAppUsage
    public void setMasterMute(boolean mute, int flags) {
        final IAudioService service = getService();
        try {
            service.setMasterMute(mute, flags, getContext().getOpPackageName(),
                    UserHandle.getCallingUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

#### 1.1.2 AudioManger 的初始化

SystemServiceRegistry 里维护了 SYSTEM_SERVICE_NAMES 和 SYSTEM_SERVICE_FETCHERS 两个 static Map

```
//Context.java (f:\q\frameworks\base\core\java\android\content)	236699	2020/4/26
    public static final String AUDIO_SERVICE = "audio";


//SystemServiceRegistry.java (f:\q\frameworks\base\core\java\android\app)	73684	2020/2/22
static {
        registerService(Context.AUDIO_SERVICE, AudioManager.class,
                new CachedServiceFetcher<AudioManager>() {
            @Override
            public AudioManager createService(ContextImpl ctx) {
                return new AudioManager(ctx);
            }});
}
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
```
static {
        registerService(Context.AUDIO_SERVICE, AudioManager.class,
                new CachedServiceFetcher<AudioManager>() {
            @Override
            public AudioManager createService(ContextImpl ctx) {
                return new AudioManager(ctx);
            }});
}
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
```

#### 1.1.3 AudioManager 的获取

SYSTEM_SERVICE_FETCHERS 直接获取 AudioManager

```
//Context.java (f:\q\frameworks\base\core\java\android\content)	236699	2020/4/26

public abstract @Nullable Object getSystemService(@ServiceName @NonNull String name);

//ContextImpl.java (f:\q\frameworks\base\core\java\android\app)	104383	2020/2/22

    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
//SystemServiceRegistry.java (f:\q\frameworks\base\core\java\android\app)	73684	2020/2/22
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```

### 1.2 AudioService

#### 1.2.1 SystemServer 启动 AudioService

在 SystemServer 的启动流程中 startOtherServices

```
private void startOtherServices() {
		   traceBeginAndSlog("StartAudioService");
            if (!isArc) {
                mSystemServiceManager.startService(AudioService.Lifecycle.class);
            } else {
                String className = context.getResources()
                        .getString(R.string.config_deviceSpecificAudioService);
                try {
                    mSystemServiceManager.startService(className + "$Lifecycle");
                } catch (Throwable e) {
                    reportWtf("starting " + className, e);
                }
            }
            traceEnd();
  }

    @SuppressWarnings("unchecked")
    public SystemService startService(String className) {
        final Class<SystemService> serviceClass;
        try {
            serviceClass = (Class<SystemService>)Class.forName(className);
        } catch (ClassNotFoundException ex) {
        }
        return startService(serviceClass);
    }

@SuppressWarnings("unchecked")
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        //反射调用Lifecycle extends SystemService构造方法
        Constructor<T> constructor = serviceClass.getConstructor(Context.class);
        service = constructor.newInstance(mContext);
    	startService(service);
        return service;
    }
    public void startService(@NonNull final SystemService service) {
        // Register it.
        //放入数组用于 接受SystemService启动过程 中flag回调 如onBootPhase
        mServices.add(service);
        // Start it.
        long time = SystemClock.elapsedRealtime();
        try {
            //AudioService onStart
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
    }
```

#### 1.2.2 AudioService 的初始化和 onStart

构造方法创建 AudioService 实例，onStart 注册到 ServiceManager 里

```
public static final class Lifecycle extends SystemService {
        private AudioService mService;    
    public Lifecycle(Context context) {
        super(context);
        mService = new AudioService(context);
    }

    @Override
    public void onStart() {
        publishBinderService(Context.AUDIO_SERVICE, mService);
    }
}
//SystemService里将AudioService add到ServiceManager
    protected final void publishBinderService(String name, IBinder service,
            boolean allowIsolated, int dumpPriority) {
        //addService通过jin binder将service添加到c层的Service_manager(do_add_service)中 
        ServiceManager.addService(name, service, allowIsolated, dumpPriority);
    }
```

#### 1.2.3 AudioService 的获取

在 AudioManger 中通过 ServiceManger getService 获取器 AudioService,binder 通信比较复杂，在这就不细说了

```
@UnsupportedAppUsage
    private static IAudioService getService()
    {
        if (sService != null) {
            return sService;
        }
        //ServiceManager getService 是从C层的service_manager.c（do_find_service）中获取之前addService
        IBinder b = ServiceManager.getService(Context.AUDIO_SERVICE);
        sService = IAudioService.Stub.asInterface(b);
        return sService;
    }
```

#### 1.2.3 AudioService setMasterMute 逻辑

```
public void setMasterMute(boolean mute, int flags, String callingPackage, int userId) {
    setMasterMuteInternal(mute, flags, callingPackage, Binder.getCallingUid(),
            userId);
}

        if ((isPlatformAutomotive() && userId == UserHandle.USER_SYSTEM)
                || (getCurrentUserId() == userId)) {
            if (mute != AudioSystem.getMasterMute()) {
                //HDMI相关mute
                setSystemAudioMute(mute);
                //调用底层mute逻辑 接下来分析这条路
                AudioSystem.setMasterMute(mute);
                //UI update and Broadcast Intent
                sendMasterMuteUpdate(mute, flags);
				
                //发送广播
                Intent intent = new Intent(AudioManager.MASTER_MUTE_CHANGED_ACTION);
                intent.putExtra(AudioManager.EXTRA_MASTER_VOLUME_MUTED, mute);
                sendBroadcastToAll(intent);
            }
        }
```

### 1.3 AudioSystem

AudioSystem 里有大量的 native static 方法 用于上层 AudioService 等和 native Service AudioFlinger AudioPolicy 通信

#### 1.3.1 AudioSystem setMasterMute

很简单调用 native 方法，复杂点在于这个 jni 在什么时候加载注册的

```
@UnsupportedAppUsage
    public static native boolean getMasterMute();
```

#### 1.3.2 Zygote.rc 解析

init 进程解析 Zygote.rc 文件在这就不细说了，Android runtime (android_media_AudioSystem 属于 Runtime jni 在 Zygote 里动态加载注册)JNI 是在 Zygote 初始化时注册的

init.rc 位于 / system/core/rootdir 下。在这个路径下还包括四个关于 zygote 的 rc 文件。

分别是 init.zygote32.rc，init.zygote32_64.rc，init.zygote64.rc，init.zygote64_32.rc，由硬件决定调用哪个文件。

这里拿 64 位处理器为例，init.zygote64.rc 的代码如下所示： 入口为 app_process64 的 main

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main		# class是一个option，指定zygote服务的类型为main
        priority -20
        user root
   	    group root readproc reserved_disk
        socket zygote stream 660 root system  # socket关键字表示一个option，创建一个名为dev/socket/zygote，类型为stream，权限为660的socket
   	    socket usap_pool_primary stream 660 root system
        onrestart write /sys/android_power/request_state wake # onrestart是一个option，说明在zygote重启时需要执行的command
	    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

Android.mk

```
app_process_src_files := \
     app_main.cpp \
 
 LOCAL_SRC_FILES:= $(app_process_src_files)
 LOCAL_MODULE:= app_process
 LOCAL_MULTILIB := both
 LOCAL_MODULE_STEM_32 := app_process32
 LOCAL_MODULE_STEM_64 := app_process64
```

App_main.cpp

App_main 是 Zygote 启动的入口，

```
int main(int argc, char* const argv[])
{
	AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    
    //读取响应的入参
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }
    
    if (startSystemServer) {
        args.add(String8("start-system-server"));
    }
    
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

#### 1.3.3 AndroidRuntime.cpp 初始化

AndroidRuntime 是 Android/Java 的运行环境 包括启动 systemServer 启动虚拟机 注册 JNI android_media_AudioSystem 中的 native 放法在这开始注册

start 就进入了 JAVA 世界了

```
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    static const String8 startSystemServer("start-system-server");
    JNIEnv* env;
    //启动虚拟机
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);
    //注册JNI
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }
}

int AndroidRuntime::startReg(JNIEnv* env)
{
	if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) {
        env->PopLocalFrame(NULL);
        return -1;
    }
}

static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
{
    for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
//列举部分audio runtime jni
static const RegJNIRec gRegJNI[] = {
    REG_JNI(register_android_media_AudioEffectDescriptor),
    REG_JNI(register_android_media_AudioSystem),
    REG_JNI(register_android_media_AudioRecord),
    REG_JNI(register_android_media_AudioTrack),
}
```

#### 1.3.4 android_media_AudioSystem 注册

```
int register_android_media_AudioSystem(JNIEnv *env)
{
    RegisterMethodsOrDie(env, kClassPathName, gMethods, NELEM(gMethods));
    return RegisterMethodsOrDie(env, kEventHandlerClassPathName, gEventHandlerMethods,
                                NELEM(gEventHandlerMethods));
}

static const JNINativeMethod gMethods[] = {
    {"setMasterVolume",     "(F)I",     (void *)android_media_AudioSystem_setMasterVolume},
    {"getMasterVolume",     "()F",      (void *)android_media_AudioSystem_getMasterVolume},
    {"setMasterMute",       "(Z)I",     (void *)android_media_AudioSystem_setMasterMute},
    {"getMasterMute",       "()Z",      (void *)android_media_AudioSystem_getMasterMute},
}
```

android_media_AudioSystem android_media_AudioSystem_setMasterMute

```
static jint
android_media_AudioSystem_setMasterMute(JNIEnv *env, jobject thiz, jboolean mute)
{
    return (jint) check_AudioSystem_Command(AudioSystem::setMasterMute(mute));
}
```

#### 1.3.5 AudioSystem::setMasterMute

进入 C 层

```
status_t AudioSystem::setMasterMute(bool mute)
{
    const sp<IAudioFlinger>& af = AudioSystem::get_audio_flinger();
    if (af == 0) return PERMISSION_DENIED;
    af->setMasterMute(mute);
    return NO_ERROR;
}
```