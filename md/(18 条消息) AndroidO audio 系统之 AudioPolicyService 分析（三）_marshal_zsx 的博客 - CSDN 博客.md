> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/marshal_zsx/article/details/81060100)

1.AudioPolicyService 基础
-----------------------

AudioPolicy 在 Android 系统中主要负责 Audio” 策略” 相关的问题。它和 AudioFlinger 一起组成了 Android Audio 系统的两个服务。一个负责管理 audio 的 “路由”，一个负责管理 audio“设备”。在 Android O 版本的系统中，这两个服务都是在系统启动的过程中，通过 main_audioserver.cpp 来加载的。

AudioPolicyService 在 Android Audio 系统中主要完成以下几个任务：  
1. 管理输入输出设备，包括设备的连接、断开状态，设备的选择和切换等  
2. 管理系统的音频策略，比如通话时播放音乐、或者播放音乐时来电话的一系列处理  
3. 管理系统的音量  
4. 上层的一些音频参数也可以通过 AudioPolicyService 设置到底层去

2.AudioPolicyService 启动流程
-------------------------

AudioPolicyService 服务运行在 mediaserver 进程中, 随着 audioserver 进程启动而启动。

```
//frameworks/av/media/audioserver/main_audioserver.cpp
int main(int argc __unused, char **argv)
{
        ......
        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        AudioPolicyService::instantiate();
        ......
}
```

AudioPolicyService::instantiate() 并不属于 AudioPolicyService 的内部类，而是 BinderService 类的一个实现包括 AudioFlinger,AudioPolicy 等在内的几个服务都继承自这个统一的 Binder 的服务类，具体实现在 BinderService.h 中

```
// frameworks/native/include/binder/BinderService.h
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        //SERVICE是文件中定义的一个模板，AudioPolicyService调用了instantiate()函数，
        //所以当前的SERVICE为AudioPolicyService
        return sm->addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }

    static void instantiate() { publish(); }
```

可以看出 publish() 函数所做的事获取到 ServiceManager 的代理，然后 new 一个调用 instantiate 的那个 service 的对象并把它添加到 ServiceManager 中。  
所以下一步就是去分析 AudioPolicyService 的构造函数了

```
// frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService(), mpAudioPolicyDev(NULL), mpAudioPolicy(NULL),
      mAudioPolicyManager(NULL), mAudioPolicyClient(NULL), mPhoneState(AUDIO_MODE_INVALID)
{
}
```

然后发现它的构造函数里面除了初始化了一些变量之外似乎并没有做其他的事情，既然这样，那么 AudioPolicyService 所作的初始化的事情到底是在什么地方进行的呢，继续分析上面的构造函数，AudioPolicyService 是继承自 BnAudioPolicyService 的，一步步往上推，最终发现它的祖先是 RefBase，根据强指针的特性，目标对象在第一次被引用时会调用 onFirstRef() 的，现在目标明确了，我们就去看一下 AudioPolicyService::onFirstRef() 里面有没有我们比较关心的信息。

```
// frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
void AudioPolicyService::onFirstRef()
{
    {
        Mutex::Autolock _l(mLock);

        // start tone playback thread 用于播放tone音
        mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
        // start audio commands thread  用于执行audio命令
        mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
        // start output activity command thread  用于执行输出命令
        mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);

        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
    }
    // load audio processing modules
    sp<AudioPolicyEffects>audioPolicyEffects = new AudioPolicyEffects();
    {
        Mutex::Autolock _l(mLock);
        mAudioPolicyEffects = audioPolicyEffects;
    }
}
```

可以看出在第一次被强引用时 AudioPolicyService 创建了 3 个 AudioCommandThread 和 AudioPolicyManager。

看一下创建了 3 个 AudioCommandThread 之后做了哪些事。首先它直接 new 了一个 AudioPolicyClient,AudioPolicyClient 类定义在 AudioPolicyService.h 中

```
//frameworks/av/services/audiopolicy/service/AudioPolicyService.h
class AudioPolicyClient : public AudioPolicyClientInterface
{
public:
    explicit AudioPolicyClient(AudioPolicyService *service) : mAudioPolicyService(service) {}
    ......
}
```

它的实现在 frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp 中。创建完了 AudioPolicyClient 之后通过调用 createAudioPolicyManager 方法创建了一个 AudioPolicyManager 对象下面看一下 createAudioPolicyManager 方法中是怎样创建 AudioPolicyManager 的。

```
//frameworks/av/services/audiopolicy/manager/AudioPolicyFactory.cpp
extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}
```

可见他是直接 new 了一个 AudioPolicyManager 然后把刚才创建的 AudioPolicyClient 传了进去。

```
//  frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    ......
    mEngine->setObserver(this);
    ......
}
```

    mEngine->setObserver(this) 把 mApmObserver 绑定为 AudioPolicyManager 对象，所以在 frameworks/ av/services/audiopolicy/enginedefault/Engine.cpp 中调用 mApmObserver->xxx() 都是调用的 AudioPolicyManager 类中的成员函数。 Engine 类里要用到 AudioPOlicyManager 类里的成员函数就用 mApmObserver->xxx(),AudioPolicyManager 类里要使用 Engine 类的成员函数就用 mEngine->xxx()

3.AudioPolicyManager 分析
-----------------------

```
//frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface) :
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false),
    mMusicEffectOutput(AUDIO_IO_HANDLE_NONE)
{
    //1.加载audio_policy.conf配置文件 
    //配置文件路径：/vendor/etc/audio_policy.conf 或者 /system/etc/audio_policy.conf
    ConfigParsingUtils::loadConfig(....);
    //2.初始化各种音频流对应的音量调节点
    mVolumeCurves->initializeVolumeCurves(speakerDrcEnabled);
    //3.加载audio policy硬件抽象库
    mHwModules[i]->mHandle =  mpClientInterface->loadHwModule(mHwModules[i]->mName);
    //4.打开输出设备
    mpClientInterface->openOutput(....);
    //5.保存输出设备描述符对象
    addOutput(output, outputDesc);
    //6.设置输出设备
    setOutputDevice(....);
    //7.打开输入设备
    mpClientInterface->openInput(....);
    //8.更新输出设备
    updateDevicesAndOutputs();
}
```

### 3.1 加载 audio_policy.conf 配置文件

在 AudioPolicyManager 创建过程中会通过加载 audio_policy.conf 配置文件来加载音频设备, Android 为每种音频接口定义了对应的硬件抽象层。每种音频接口定义了不同的输入输出，一个接口可以具有多个输入或者输出，每个输入输出可以支持不同的设备，通过读取 audio_policy.conf 文件可以获取系统支持的音频接口参数，在 AudioPolicyManager 中会优先加载 / vendor/etc/audio_policy.conf 配置文件, 如果该配置文件不存在, 则加载 / system/etc/audio_policy.conf 配置文件。AudioPolicyManager 加载完所有音频接口后, 就知道了系统支持的所有音频接口参数, 可以为音频输出提供决策。

audio_policy.conf 同时定义了多个 audio 接口, 每一个 audio 接口包含若干 output 和 input, 而每个 output 和 input 又同时支持多种输入输出模式, 每种输入输出模式又支持若干种设备.  
![](https://images2015.cnblogs.com/blog/708332/201607/708332-20160725230446622-1465025873.jpg)

```
ConfigParsingUtils::loadAudioPolicyConfig(....);
```

分成两部分，第一部分是解析全局标签，第二部分是解析 audio_hw_modules 标签，其子标签都表示 hardware module，有 primary 和 r_submix 两种 hardware module 都被解析到 mHwModules，hardware module 的子标签有 outputs 和 inputs,outputs 的各个子标签被解析到 mHwModules 的 mOutputProfiles，inputs 的各个子标签被解析到 mHwModules 的 mInputProfiles。

### 3.2 初始化各种音频流对应的音量调节点

```
mVolumeCurves->initializeVolumeCurves(mSpeakerDrcEnabled);

//frameworks/av/services/audiopolicy/common/managerdefinitions/src/StreamDescriptor.cpp
void StreamDescriptorCollection::initializeVolumeCurves(bool isSpeakerDrcEnabled)
{
    for (int i = 0; i < AUDIO_STREAM_CNT; i++) {
        for (int j = 0; j < DEVICE_CATEGORY_CNT; j++) {
            //设置音量
            setVolumeCurvePoint(static_cast<audio_stream_type_t>(i),
                                static_cast<device_category>(j),
                                Gains::sVolumeProfiles[i][j]);
        }
    }
    ......
}
```

### 3.3 加载 audio policy 硬件抽象库

```
mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->getName());

//frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp
audio_module_handle_t AudioPolicyService::AudioPolicyClient::loadHwModule(const char *name)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    return af->loadHwModule(name);
}
```

这里直接调用了 AudioFlinger::loadHwModule()。下面进入到 AudioFlinger.cpp 中看一下它做了那些事

```
//frameworks/av/services/audioflinger/AudioFlinger.cpp
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (name == NULL) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    if (!settingsAllowed()) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}

audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    //1.是否已经加载过这个interface
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }
    //2.加载audio interface
    int rc = mDevicesFactoryHal->openDevice(name, &dev);
    //3.初始化
    rc = dev->initCheck();
    //4.添加到全局变量中
    audio_module_handle_t handle = (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));
}
```

loadHwModule_l 是通过调用 openDevice 方法来打开加载 audio 设备的，该方法的实现类是 DevicesFactoryHalHybrid

```
//frameworks/av/media/libaudiohal/DevicesFactoryHalHybrid.cpp
status_t DevicesFactoryHalHybrid::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mHidlFactory != 0 && strcmp(AUDIO_HARDWARE_MODULE_ID_A2DP, name) != 0) {
        return mHidlFactory->openDevice(name, device); //Hidl方式加载
    }
    return mLocalFactory->openDevice(name, device);  //旧版本本地加载
}
```

DevicesFactoryHalHybrid 中提供了两种方式加载，本文将介绍 hidl 的方式加载

```
//frameworks/av/media/libaudiohal/DevicesFactoryHalHidl.cpp
status_t DevicesFactoryHalHidl::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mDevicesFactory == 0) return NO_INIT;
    IDevicesFactory::Device hidlDevice;
    status_t status = nameFromHal(name, &hidlDevice);
    if (status != OK) return status;
    Result retval = Result::NOT_INITIALIZED;
    Return<void> ret = mDevicesFactory->openDevice(
            hidlDevice,
            [&](Result r, const sp<IDevice>& result) {
                retval = r;
                if (retval == Result::OK) {
                    *device = new DeviceHalHidl(result);
                }
            });
    if (ret.isOk()) {
        if (retval == Result::OK) return OK;
        else if (retval == Result::INVALID_ARGUMENTS) return BAD_VALUE;
        else return NO_INIT;
    }
    return FAILED_TRANSACTION;
}
```

```
//hardware/interfaces/audio/2.0/default/DevicesFactory.cpp
Return<void> DevicesFactory::openDevice(IDevicesFactory::Device device, openDevice_cb _hidl_cb)  {
    audio_hw_device_t *halDevice;
    Result retval(Result::INVALID_ARGUMENTS);
    sp<IDevice> result;
    const char* moduleName = deviceToString(device);
    if (moduleName != nullptr) {
        int halStatus = loadAudioInterface(moduleName, &halDevice);//加载audio interface
        if (halStatus == OK) {
            if (device == IDevicesFactory::Device::PRIMARY) {
                result = new PrimaryDevice(halDevice);
            } else {
                result = new ::android::hardware::audio::V2_0::implementation::
                    Device(halDevice);
            }
            retval = Result::OK;
        } else if (halStatus == -EINVAL) {
            retval = Result::NOT_INITIALIZED;
        }
    }
    _hidl_cb(retval, result);//将加载的设备通过回调匿名方法传递回去
    return Void();
}

int DevicesFactory::loadAudioInterface(const char *if_name, audio_hw_device_t **dev)
{
    const hw_module_t *mod;
    int rc;
    //在system/lib/hw/等目录下查找对应的动态库并加载
    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    //打开对应的device，并获取hw_device_t指针类型的设备对象
    rc = audio_hw_device_open(mod, dev);
    ......
}
```

### 3.4 打开输出设备

```
//frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp
status_t AudioPolicyService::AudioPolicyClient::openOutput(audio_module_handle_t module,
                                                           audio_io_handle_t *output,
                                                           audio_config_t *config,
                                                           audio_devices_t *devices,
                                                           const String8& address,
                                                           uint32_t *latencyMs,
                                                           audio_output_flags_t flags)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return PERMISSION_DENIED;
    }
    return af->openOutput(module, output, config, devices, address, latencyMs, flags);
}
```

这里直接调用了 AudioFlinger::openOutput()。下面进入到 AudioFlinger.cpp 中看一下它做了那些事

```
//frameworks/av/services/audioflinger/AudioFlinger.cpp
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    ......
    sp<ThreadBase> thread = openOutput_l(module, output, config, *devices, address, flags);
    ......
}

sp<AudioFlinger::ThreadBase> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    //查找合适的设备并打开
    AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
    ......
    AudioStreamOut *outputStream = NULL;
    //在硬件设备上打开一个输出流
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;

    if (status == NO_ERROR) {
        if (flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) {
            sp<MmapPlaybackThread> thread =
                    new MmapPlaybackThread(this, *output, outHwDev, outputStream,
                                          devices, AUDIO_DEVICE_NONE, mSystemReady);
            mMmapThreads.add(*output, thread);
            ALOGV("openOutput_l() created mmap playback thread: ID %d thread %p",
                  *output, thread.get());
            return thread;
        } else {
            sp<PlaybackThread> thread;
            if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
                //AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD音频流，创建OffloadThread实例
                thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created offload output: ID %d thread %p",
                      *output, thread.get());
            } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                    || !isValidPcmSinkFormat(config->format)
                    || !isValidPcmSinkChannelMask(config->channel_mask)) {
                //若是AUDIO_OUTPUT_FLAG_DIRECT音频，则创建DirectOutputThread实例    
                thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created direct output: ID %d thread %p",
                      *output, thread.get());
            } else {
                //其他音频流，则创建MixerThread实例
                thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created mixer output: ID %d thread %p",
                      *output, thread.get());
            }
            mPlaybackThreads.add(*output, thread);
            return thread;
        }
    }

    return 0;
}
```

打开输入设备的流程与打开输出设备的流程基本类似，此处就不在累赘分析了，有兴趣的可以尝试分析。

4. 框架时序图
--------

![](https://img-blog.csdn.net/20180716143844331?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21hcnNoYWxfenN4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

5. 小结
-----

```
到此相信大家对audiopolicyservice启动流程有了大致的熟悉，接下来将分析一下audioflinger的启动流程，
未完待续。。。
```