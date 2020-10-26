\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[blog.csdn.net\](https://blog.csdn.net/sinat\_18179367/article/details/106000254)

Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）
===============================================================================

随手笔记系列，内容较多较繁琐，用于结合目录快速查询

本文将通过 AudioManger 的 setMastermute Api 的通信传递实现逻辑，自上而下梳理一下调用顺序和各模块相关初始化，最终到的 audio\_hw.c 来实现静音

### 文章目录

*   [Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）](#Android_Q_Audio_AudioFlingerAudio_hal_AudioDevice__0)
*   *   [三、openHwDevice 打开硬件设备](#openHwDevice_7)
    *   *   [3.1 audioserver loadHwMoudle](#31_audioserver_loadHwMoudle_9)
        *   *   [3.1.1 AudioPolicyServicec 初始化](#311__AudioPolicyServicec_11)
            *   [3.1.2 AudioPolicyManager 初始化](#312__AudioPolicyManager_62)
            *   [3.1.3 mpClientInterface->loadHwModule](#313__mpClientInterfaceloadHwModule_152)
            *   [3.1.4 AudioFlinger loadHwModule](#314_AudioFlinger__loadHwModule_177)
            *   [3.1.5 对 libAudioHal 返回的 device add 进 mAudioHwDevs 的维护](#315_libAudioHaldevice_addmAudioHwDevs_258)
        *   [3.2 libAudioHal openDevice](#32_libAudioHal_openDevice_333)
        *   *   [3.2.1 libAudioHAL openDevice](#321_libAudioHAL_openDevice_335)
            *   [3.2.2 对 Hal 层返回的 device 进行封装返回 DeviceHALHidl](#322_HaldeviceDeviceHALHidl_381)
        *   [3.3 hal 层 openDevice](#33_hal_openDevice_436)
        *   *   [3.3.1 hal 层 openDevice](#331_hal_openDevice_438)
            *   [3.3.2 对返回的硬件 device 进行封装 PrimaryDevice 返回](#332devicePrimaryDevice_531)
            *   [3.3.3 hal loadAudioInterface](#333_hal_loadAudioInterface_639)
            *   [3.3.4 audio\_hw\_device\_open 各结构体转换和初始化 audio\_hw\_device 到 hw\_device\_t](#334_audio_hw_device_open_audio_hw_devicehw_device_t_675)
        *   [3.4 Hardware device\_open](#34_Hardware_device_open_872)
        *   *   [3.4.1 hw\_get\_module\_by\_class](#341_hw_get_module_by_class_874)
            *   [3.4.2 load 加载](#342_load__935)
            *   [3.4.3 hw\_module\_t 赋值返回](#343__hw_module_t_1020)
        *   [3.5 audio\_hw.c adev\_open](#35_audio_hwc__adev_open_1098)
        *   *   [3.5.1 open 方法引用初始化](#351_open_1100)
            *   [3.5.2 各家 adev\_open 实现](#352_adev_open_1111)

三、openHwDevice 打开硬件设备
---------------------

### 3.1 audioserver loadHwMoudle

#### 3.1.1 AudioPolicyServicec 初始化

首先在 AudioServer 启动时，AudioPolicyService 进行初始化

```
//Main\_audioserver
int main(int argc \_\_unused, char \*\*argv)
{
	AudioPolicyService::instantiate();
}
```

instantiate() 逻辑与 AudioFlinger 一样，详见 2.1.2。所以现在直接看 AudioPolicyService 的构造函数，和 onFirstRef();

```
AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService(), mpAudioPolicyDev(NULL), mpAudioPolicy(NULL),
      mAudioPolicyManager(NULL), mAudioPolicyClient(NULL), mPhoneState(AUDIO\_MODE\_INVALID)
{
}
void AudioPolicyService::onFirstRef()
{
    {
        Mutex::Autolock \_l(mLock);
		//启动 audioCommand线程
        // start audio commands thread
        mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
        //启动outputCommand线程
        // start output activity command thread
        mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);
//初始化AudioPolicyClient 用于和af通信
        mAudioPolicyClient = new AudioPolicyClient(this);
//创建AudioPolicyManager 把 mAudioPolicyClient传入
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
    }
    // load audio processing modules
    sp<AudioPolicyEffects>audioPolicyEffects = new AudioPolicyEffects();
    {
        Mutex::Autolock \_l(mLock);
        mAudioPolicyEffects = audioPolicyEffects;
    }

    mUidPolicy = new UidPolicy(this);
    mUidPolicy->registerSelf();

    mSensorPrivacyPolicy = new SensorPrivacyPolicy(this);
    mSensorPrivacyPolicy->registerSelf();
}
```

#### 3.1.2 AudioPolicyManager 初始化

AudioPolicyManger 有多个 分为 vendor 目录下的 managerdefault 目录下的

vendor 为厂商自己定义的，managerdefault 目录为原生默认，接下来以原生为主

先看 mk

```
LOCAL\_SRC\_FILES:= \\
     AudioPolicyFactory.cpp

LOCAL\_SHARED\_LIBRARIES := \\
	libaudiopolicymanagerdefault

LOCAL\_MODULE:= libaudiopolicymanager
```

AudioPolicyFactory.h

```
extern "C" AudioPolicyInterface\* createAudioPolicyManager(
        AudioPolicyClientInterface \*clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}
```

AudioPolicyManager

```
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface \*clientInterface)
        : AudioPolicyManager(clientInterface, false /\*forTesting\*/)
{
    //解析audio\_policy\_configuration.xml        
    loadConfig();
    //hwModule 加载  output input device初始化 等       
    initialize();
}
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface \*clientInterface,
                                       bool /\*forTesting\*/)
    :
    mUidCached(AID\_AUDIOSERVER), // no need to call getuid(), there's only one of us running.
    mpClientInterface(clientInterface),
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
//初始化 mHwModulesAll   
    mConfig(mHwModulesAll, mAvailableOutputDevices, mAvailableInputDevices, mDefaultOutputDevice),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false),
    mMusicEffectOutput(AUDIO\_IO\_HANDLE\_NONE)
{
}
```

这次主要看 hwModule 加载

loadConfig(); 通过解析 XML, 得到不同的 Moudle 放入 mHwModulesAll 集合里

解析过程代码太多可以看 Serializer.cpp (f:\\q\\frameworks\\av\\services\\audiopolicy\\common\\managerdefinitions\\src)

```
#define AUDIO\_HARDWARE\_MODULE\_ID\_PRIMARY "primary"
#define AUDIO\_HARDWARE\_MODULE\_ID\_A2DP "a2dp"
#define AUDIO\_HARDWARE\_MODULE\_ID\_USB "usb"
#define AUDIO\_HARDWARE\_MODULE\_ID\_REMOTE\_SUBMIX "r\_submix"
#define AUDIO\_HARDWARE\_MODULE\_ID\_CODEC\_OFFLOAD "codec\_offload"
#define AUDIO\_HARDWARE\_MODULE\_ID\_STUB "stub"
#define AUDIO\_HARDWARE\_MODULE\_ID\_HEARING\_AID "hearing\_aid"
#define AUDIO\_HARDWARE\_MODULE\_ID\_MSD "msd"
```

在 initialize 里初始化 loadHwMoudle 循环打开 loadHwModule

```
status\_t AudioPolicyManager::initialize() {
    for (const auto& hwModule : mHwModulesAll) {
        hwModule->setHandle(mpClientInterface->loadHwModule(hwModule->getName()));
        if (hwModule->getHandle() == AUDIO\_MODULE\_HANDLE\_NONE) {
            ALOGW("could not open HW module %s", hwModule->getName());
            continue;
        }
        ······
}
```

#### 3.1.3 mpClientInterface->loadHwModule

mpClientInterface 为之前传入的 AudioPolicyClient

```
//AudioPolicyClientImpl.cpp (f:\\q\\frameworks\\av\\services\\audiopolicy\\service)

audio\_module\_handle\_t AudioPolicyService::AudioPolicyClient::loadHwModule(const char \*name)
{	
    //再次看到 AudioSystem::get\_audio\_flinger() 
    sp<IAudioFlinger> af = AudioSystem::get\_audio\_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", \_\_func\_\_);
        return AUDIO\_MODULE\_HANDLE\_NONE;
    }

    return af->loadHwModule(name);
}
```

:get\_audio\_flinger() 逻辑详见 2.1.3 AudioFlinger 的获取

最后为 AudioFlinger 的 loadHwModule 方法

#### 3.1.4 AudioFlinger loadHwModule

```
audio\_module\_handle\_t AudioFlinger::loadHwModule(const char \*name){
	return loadHwModule\_l(name);
}
audio\_module\_handle\_t AudioFlinger::loadHwModule\_l(const char \*name)
{
//防止重复load    
    for (size\_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    sp<DeviceHalInterface> dev;
//openDevice
    int rc = mDevicesFactoryHal->openDevice(name, &dev);
    if (rc) {
        ALOGE("loadHwModule() error %d loading module %s", rc, name);
        return AUDIO\_MODULE\_HANDLE\_NONE;
    }

    mHardwareStatus = AUDIO\_HW\_INIT;
    rc = dev->initCheck();
    mHardwareStatus = AUDIO\_HW\_IDLE;
    if (rc) {
        ALOGE("loadHwModule() init check error %d for module %s", rc, name);
        return AUDIO\_MODULE\_HANDLE\_NONE;
    }
//初始化flags    
    AudioHwDevice::Flags flags = static\_cast<AudioHwDevice::Flags>(0);
    {  // scope for auto-lock pattern
        AutoMutex lock(mHardwareLock);
		//获取device的 volume和mute状态
        if (0 == mAudioHwDevs.size()) {
            mHardwareStatus = AUDIO\_HW\_GET\_MASTER\_VOLUME;
            float mv;
            if (OK == dev->getMasterVolume(&mv)) {
                mMasterVolume = mv;
            }

            mHardwareStatus = AUDIO\_HW\_GET\_MASTER\_MUTE;
            bool mm;
            if (OK == dev->getMasterMute(&mm)) {
                mMasterMute = mm;
            }
        }

        mHardwareStatus = AUDIO\_HW\_SET\_MASTER\_VOLUME;
        if (OK == dev->setMasterVolume(mMasterVolume)) {
            flags = static\_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD\_CAN\_SET\_MASTER\_VOLUME);
        }

        mHardwareStatus = AUDIO\_HW\_SET\_MASTER\_MUTE;
        if (OK == dev->setMasterMute(mMasterMute)) {
            flags = static\_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD\_CAN\_SET\_MASTER\_MUTE);
        }

        mHardwareStatus = AUDIO\_HW\_IDLE;
    }
    if (strcmp(name, AUDIO\_HARDWARE\_MODULE\_ID\_MSD) == 0) {
        // An MSD module is inserted before hardware modules in order to mix encoded streams.
        flags = static\_cast<AudioHwDevice::Flags>(flags | AudioHwDevice::AHWD\_IS\_INSERT);
    }
    // flag 是 音量和静音状态
    //其中handle的值是由nextUniqueId生成的，这样做保证了这个audiointerface拥有全局唯一的id号。
//添加到mAudioHwDevs里    
    audio\_module\_handle\_t handle = (audio\_module\_handle\_t) nextUniqueId(AUDIO\_UNIQUE\_ID\_USE\_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));

    ALOGI("loadHwModule() Loaded %s audio interface, handle %d", name, handle);

    return handle;

}
```

#### 3.1.5 对 libAudioHal 返回的 device add 进 mAudioHwDevs 的维护

在 loadHwModule 最后会 将拿到的 dev handle name flag 作为参数，创建 AudioHwDevice，最后加入到 mAudioHwDevs 里

```
audio\_module\_handle\_t handle = (audio\_module\_handle\_t) nextUniqueId(AUDIO\_UNIQUE\_ID\_USE\_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));
```

dev 是 hal 层 DeviceHalInterface 类型的指针

name 是 moudle 的名字

handle 是通过 nextUniqueId 生成的唯一 id 用作 key 值来查询

flags 是当前 device 的 volume mute 等信息

创建 AudioHwDevice

```
class AudioHwDevice {
public:
    enum Flags {
        AHWD\_CAN\_SET\_MASTER\_VOLUME  = 0x1,
        AHWD\_CAN\_SET\_MASTER\_MUTE    = 0x2,
        // Means that this isn't a terminal module, and software patches
        // are used to transport audio data further.
        AHWD\_IS\_INSERT              = 0x4,
    };

    AudioHwDevice(audio\_module\_handle\_t handle,
                  const char \*moduleName,
                  sp<DeviceHalInterface> hwDevice,
                  Flags flags)
        : mHandle(handle)
        , mModuleName(strdup(moduleName))
        , mHwDevice(hwDevice)
        , mFlags(flags) { }
```

AudioHwDevice 提供的方法

```
bool canSetMasterVolume() const {
        return (0 != (mFlags & AHWD\_CAN\_SET\_MASTER\_VOLUME));
    }

    bool canSetMasterMute() const {
        return (0 != (mFlags & AHWD\_CAN\_SET\_MASTER\_MUTE));
    }

    bool isInsert() const {
        return (0 != (mFlags & AHWD\_IS\_INSERT));
    }

    audio\_module\_handle\_t handle() const { return mHandle; }
    const char \*moduleName() const { return mModuleName; }
//获取 DeviceHalInterface 指针
    sp<DeviceHalInterface> hwDevice() const { return mHwDevice; }

    status\_t openOutputStream(
            AudioStreamOut \*\*ppStreamOut,
            audio\_io\_handle\_t handle,
            audio\_devices\_t devices,
            audio\_output\_flags\_t flags,
            struct audio\_config \*config,
            const char \*address);

    bool supportsAudioPatches() const;
```

### 3.2 libAudioHal openDevice

#### 3.2.1 libAudioHAL openDevice

mDevicesFactoryHal 为之前 DevicesFactoryHalInterface create

```
//DevicesFactoryHalHybrid.cpp
status\_t DevicesFactoryHalHybrid::openDevice(const char \*name, sp<DeviceHalInterface> \*device) {
    if (mHidlFactory != 0 && strcmp(AUDIO\_HARDWARE\_MODULE\_ID\_A2DP, name) != 0 &&
        strcmp(AUDIO\_HARDWARE\_MODULE\_ID\_HEARING\_AID, name) != 0) {
        return mHidlFactory->openDevice(name, device);
    }
    return mLocalFactory->openDevice(name, device);
}

//DevicesFactoryHalHidl.cpp
status\_t DevicesFactoryHalHidl::openDevice(const char \*name, sp<DeviceHalInterface> \*device) {
    if (mDeviceFactories.empty()) return NO\_INIT;
    status\_t status;
    auto hidlId = idFromHal(name, &status);
    if (status != OK) return status;
    Result retval = Result::NOT\_INITIALIZED;
    for (const auto& factory : mDeviceFactories) {
// 与Hal层通信  factory 为 之前create流程中的IDeviceFactory.hal        
        Return<void> ret = factory->openDevice(
                hidlId,
                \[&\](Result r, const sp<IDevice>& result) {
                    retval = r;
                    if (retval == Result::OK) {
                        \*device = new DeviceHalHidl(result);
                    }
                });
        if (!ret.isOk()) return FAILED\_TRANSACTION;
        switch (retval) {
            // Device was found and was initialized successfully.
            case Result::OK: return OK;
            // Device was found but failed to initalize.
            case Result::NOT\_INITIALIZED: return NO\_INIT;
            // Otherwise continue iterating.
            default: ;
        }
    }
    ALOGW("The specified device name is not recognized: \\"%s\\"", name);
    return BAD\_VALUE;
}
```

#### 3.2.2 对 Hal 层返回的 device 进行封装返回 DeviceHALHidl

```
// 与Hal层通信  factory 为 之前create流程中的IDeviceFactory.hal        
        Return<void> ret = factory->openDevice(
                hidlId,
                \[&\](Result r, const sp<IDevice>& result) {
                    retval = r;
                    if (retval == Result::OK) {
                        \*device = new DeviceHalHidl(result);
                    }
                });
```

从 Hal 拿到 IDevice 后，当做参数构建 DeviceHalHidl

mDevice 通过 IDevice 初始化

mPrimaryDevice 通过 IPrimaryDevice.hal 强转 并初始化

```
class DeviceHalHidl : public DeviceHalInterface, public ConversionHelperHidl
{
	sp<IDevice> mDevice;
    sp<IPrimaryDevice> mPrimaryDevice;  // Null if it's not a primary device.

DeviceHalHidl::DeviceHalHidl(const sp<IDevice>& device)
        : ConversionHelperHidl("Device"), mDevice(device),
          mPrimaryDevice(IPrimaryDevice::castFrom(device)) {
}

    virtual status\_t setMasterMute(bool state);
    virtual status\_t getMasterMute(bool \*state);

status\_t DeviceHalHidl::setMasterMute(bool state) {
    if (mDevice == 0) return NO\_INIT;
    return processReturn("setMasterMute", mDevice->setMasterMute(state));
}

status\_t DeviceHalHidl::getMasterMute(bool \*state) {
    if (mDevice == 0) return NO\_INIT;
    Result retval;
    Return<void> ret = mDevice->getMasterMute(
            \[&\](Result r, bool mute) {
                retval = r;
                if (retval == Result::OK) {
                    \*state = mute;
                }
            });
    return processReturn("getMasterMute", ret, retval);
}
```

所以最后 opendevice 最后放入 device 地址里的是 DeviceHalHidl，mDevice 里放的是 Hal 层的 DeviceShim

### 3.3 hal 层 openDevice

#### 3.3.1 hal 层 openDevice

IDeviceFactory.hal (4.0)

```
interface IDevicesFactory {


    openDevice(string device) generates (Result retval, IDevice result);

    openPrimaryDevice() generates (Result retval, IPrimaryDevice result);
};
```

DeviceFactory.h

```
//实现了 IDeviceFactory hal 接口
struct DevicesFactory : public IDevicesFactory {
#if MAJOR\_VERSION == 2
    Return<void> openDevice(IDevicesFactory::Device device, openDevice\_cb \_hidl\_cb) override;
#elif MAJOR\_VERSION >= 4
    Return<void> openDevice(const hidl\_string& device, openDevice\_cb \_hidl\_cb) override;
    Return<void> openPrimaryDevice(openPrimaryDevice\_cb \_hidl\_cb) override;
#endif

   private:
    template <class DeviceShim, class Callback>
    Return<void> openDevice(const char\* moduleName, Callback \_hidl\_cb);
    Return<void> openDevice(const char\* moduleName, openDevice\_cb \_hidl\_cb);

    static int loadAudioInterface(const char\* if\_name, audio\_hw\_device\_t\*\* dev);
};
extern "C" IDevicesFactory\* HIDL\_FETCH\_IDevicesFactory(const char\* name);
```

DevicesFactory openDevice

```
// /external/swiftshader/src/Common/Version.h
 #define MAJOR\_VERSION 4
```

```
#if MAJOR\_VERSION == 2
Return<void> DevicesFactory::openDevice(IDevicesFactory::Device device, openDevice\_cb \_hidl\_cb) {
    switch (device) {
        case IDevicesFactory::Device::PRIMARY:
            return openDevice<PrimaryDevice>(AUDIO\_HARDWARE\_MODULE\_ID\_PRIMARY, \_hidl\_cb);
        case IDevicesFactory::Device::A2DP:
            return openDevice(AUDIO\_HARDWARE\_MODULE\_ID\_A2DP, \_hidl\_cb);
        case IDevicesFactory::Device::USB:
            return openDevice(AUDIO\_HARDWARE\_MODULE\_ID\_USB, \_hidl\_cb);
        case IDevicesFactory::Device::R\_SUBMIX:
            return openDevice(AUDIO\_HARDWARE\_MODULE\_ID\_REMOTE\_SUBMIX, \_hidl\_cb);
        case IDevicesFactory::Device::STUB:
            return openDevice(AUDIO\_HARDWARE\_MODULE\_ID\_STUB, \_hidl\_cb);
    }
    \_hidl\_cb(Result::INVALID\_ARGUMENTS, nullptr);
    return Void();
}
#elif MAJOR\_VERSION >= 4
Return<void> DevicesFactory::openDevice(const hidl\_string& moduleName, openDevice\_cb \_hidl\_cb) {
    if (moduleName == AUDIO\_HARDWARE\_MODULE\_ID\_PRIMARY) {
        return openDevice<PrimaryDevice>(moduleName.c\_str(), \_hidl\_cb);
    }
    return openDevice(moduleName.c\_str(), \_hidl\_cb);
}

//模板 函数  
template <class DeviceShim, class Callback>
Return<void> DevicesFactory::openDevice(const char\* moduleName, Callback \_hidl\_cb) {
    audio\_hw\_device\_t\* halDevice;
    Result retval(Result::INVALID\_ARGUMENTS);
    sp<DeviceShim> result;
    int halStatus = loadAudioInterface(moduleName, &halDevice);
    if (halStatus == OK) {
   //返回IDevice 
        result = new DeviceShim(halDevice);
        retval = Result::OK;
    } else if (halStatus == -EINVAL) {
        retval = Result::NOT\_INITIALIZED;
    }
    \_hidl\_cb(retval, result);
    return Void();
}
```

#### 3.3.2 对返回的硬件 device 进行封装 PrimaryDevice 返回

```
Return<void> ret = factory->openDevice(
                hidlId,
                \[&\](Result r, const sp<IDevice>& result) {
                    retval = r;
                    if (retval == Result::OK) {
                        \*device = new DeviceHalHidl(result);
                    }
                });

    int halStatus = loadAudioInterface(moduleName, &halDevice);
    if (halStatus == OK) {
        result = new DeviceShim(halDevice);
        retval = Result::OK;
    } else if (halStatus == -EINVAL) {
        retval = Result::NOT\_INITIALIZED;
    }
    /\*
    lamda表达式 入参为 HidlId  返回值为  Result r和 sp<IDevice>& result
    
    通过\_hidl\_cb(retval, result); 进行回调返回
    
    
    
    \*/
```

DeviceShim 封装 halDevice（audio\_hw\_device\_t） DeviceShim 是模板类 openDevice(moduleName.c\_str(), \_hidl\_cb);

所以实际就是 PrimaryDevice

PrimaryDevice 的初始化

PrimaryDevice 对 Device 进行封装 Device 对 audio\_hw\_device\_t 进行了封装

PrimaryDevice 的 setMastMute 就是调用的 audio\_hw\_device\_t 的 setMastMute

```
struct PrimaryDevice : public IPrimaryDevice {

PrimaryDevice::PrimaryDevice(audio\_hw\_device\_t\* device) : mDevice(new Device(device)) {}
    
Return<Result> PrimaryDevice::initCheck() {
    return mDevice->initCheck();
}

Return<Result> PrimaryDevice::setMasterVolume(float volume) {
    return mDevice->setMasterVolume(volume);
}

Return<void> PrimaryDevice::getMasterVolume(getMasterVolume\_cb \_hidl\_cb) {
    return mDevice->getMasterVolume(\_hidl\_cb);
}

Return<Result> PrimaryDevice::setMicMute(bool mute) {
    return mDevice->setMicMute(mute);
}

Return<void> PrimaryDevice::getMicMute(getMicMute\_cb \_hidl\_cb) {
    return mDevice->getMicMute(\_hidl\_cb);
}

Return<Result> PrimaryDevice::setMasterMute(bool mute) {
    return mDevice->setMasterMute(mute); 
}
```

Device 初始化

```
struct Device : public IDevice, public ParametersUtil {
    explicit Device(audio\_hw\_device\_t\* device);

    // Methods from ::android::hardware::audio::CPP\_VERSION::IDevice follow.
    Return<Result> initCheck() override;
    Return<Result> setMasterVolume(float volume) override;
    Return<void> getMasterVolume(getMasterVolume\_cb \_hidl\_cb) override;
    Return<Result> setMicMute(bool mute) override;
    Return<void> getMicMute(getMicMute\_cb \_hidl\_cb) override;
    Return<Result> setMasterMute(bool mute) override;
    Return<void> getMasterMute(getMasterMute\_cb \_hidl\_cb) override;
    Return<void> getInputBufferSize(const AudioConfig& config,
                                    getInputBufferSize\_cb \_hidl\_cb) override;
    
    
    Return<Result> Device::setMasterMute(bool mute) {
    Result retval(Result::NOT\_SUPPORTED);
    if (mDevice->set\_master\_mute != NULL) {
        retval = analyzeStatus("set\_master\_mute", mDevice->set\_master\_mute(mDevice, mute));
    }
    return retval;
}

Return<void> Device::getMasterMute(getMasterMute\_cb \_hidl\_cb) {
    Result retval(Result::NOT\_SUPPORTED);
    bool mute = false;
    if (mDevice->get\_master\_mute != NULL) {
        retval = analyzeStatus("get\_master\_mute", mDevice->get\_master\_mute(mDevice, &mute));
    }
    \_hidl\_cb(retval, mute);
    return Void();
}
```

#### 3.3.3 hal loadAudioInterface

```
// static
int DevicesFactory::loadAudioInterface(const char\* if\_name, audio\_hw\_device\_t\*\* dev) {
    const hw\_module\_t\* mod;
    int rc;
//通用的hal模块加载 hw\_get\_module\_by\_class 是Hardware.c的公共方法 拿到audio.primary.msm8996的地址
    rc = hw\_get\_module\_by\_class(AUDIO\_HARDWARE\_MODULE\_ID, if\_name, &mod);
    if (rc) {
        ALOGE("%s couldn't load audio hw module %s.%s (%s)", \_\_func\_\_, AUDIO\_HARDWARE\_MODULE\_ID,
              if\_name, strerror(-rc));
        goto out;
    }
// 打开 hw\_debice     audio\_hw\_device\_open 为Audio.h里的方法
    rc = audio\_hw\_device\_open(mod, dev);
    if (rc) {
        ALOGE("%s couldn't open audio hw device in %s.%s (%s)", \_\_func\_\_, AUDIO\_HARDWARE\_MODULE\_ID,
              if\_name, strerror(-rc));
        goto out;
    }
    if ((\*dev)->common.version < AUDIO\_DEVICE\_API\_VERSION\_MIN) {
        ALOGE("%s wrong audio hw device version %04x", \_\_func\_\_, (\*dev)->common.version);
        rc = -EINVAL;
        audio\_hw\_device\_close(\*dev);
        goto out;
    }
    return OK;

out:
    \*dev = NULL;
    return rc;
}
```

#### 3.3.4 audio\_hw\_device\_open 各结构体转换和初始化 audio\_hw\_device 到 hw\_device\_t

1.  audio\_hw\_device\_t 通过 TO\_HW\_DEVICE\_T\_OPEN 宏定义强转成 hw\_module\_t
2.  module 为 通过 hw\_get\_module\_by\_class 初始化拿到的 hw\_module\_t 也就是 audio\_module HAL\_MODULE\_INFO\_SYM
3.  module->methods 是 hw\_module\_methods\_t 存放 module 方法，也就是 audio\_module HAL\_MODULE\_INFO\_SYM 里的. methods = &hal\_module\_methods
4.  hw\_module\_methods\_t 里面定义了一个 open 方法的引用
5.  hal\_module\_methods 里有一个. open = adev\_open,
6.  所以 module->methods->open(module, AUDIO\_HARDWARE\_INTERFACE, TO\_HW\_DEVICE\_T\_OPEN(device)); 就是调用的 adev\_open 方法

```
typedef struct hw\_module\_t {
    struct hw\_module\_methods\_t\* methods;

} hw\_module\_t;

typedef struct hw\_module\_methods\_t {
    /\*\* Open a specific device \*/
    int (\*open)(const struct hw\_module\_t\* module, const char\* id,
            struct hw\_device\_t\*\* device);

} hw\_module\_methods\_t;


#ifdef \_\_cplusplus
#define TO\_HW\_DEVICE\_T\_OPEN(x) reinterpret\_cast<struct hw\_device\_t\*\*>(x)
#else
#define TO\_HW\_DEVICE\_T\_OPEN(x) (struct hw\_device\_t\*\*)(x)
#endif

#define AUDIO\_HARDWARE\_INTERFACE "audio\_hw\_if"
 
static inline int audio\_hw\_device\_open(const struct hw\_module\_t\* module,
                                       struct audio\_hw\_device\*\* device)
{
    //TO\_HW\_DEVICE\_T\_OPEN是宏定义 将audio\_hw\_device 转换成hw\_device\_t
    return module->methods->open(module, AUDIO\_HARDWARE\_INTERFACE,
                                 TO\_HW\_DEVICE\_T\_OPEN(device));
}

static inline int audio\_hw\_device\_close(struct audio\_hw\_device\* device)
{
    return device->common.close(&device->common);
}
//audio\_hw.c
static struct hw\_module\_methods\_t hal\_module\_methods = {
    .open = adev\_open,
};

struct audio\_module HAL\_MODULE\_INFO\_SYM = {
    .common = {
        .tag = HARDWARE\_MODULE\_TAG,
        .module\_api\_version = AUDIO\_MODULE\_API\_VERSION\_0\_1,
        .hal\_api\_version = HARDWARE\_HAL\_API\_VERSION,
        .id = AUDIO\_HARDWARE\_MODULE\_ID,
        .name = "Generic car audio HW HAL",
        .author = "The Android Open Source Project",
        .methods = &hal\_module\_methods,
    },
};


static int adev\_open(const hw\_module\_t \*module,
        const char \*name, hw\_device\_t \*\*device) {
    static struct generic\_audio\_device \*adev;
    
    adev->device.common.tag = HARDWARE\_DEVICE\_TAG;
    adev->device.common.version = AUDIO\_DEVICE\_API\_VERSION\_3\_0;
    adev->device.common.module = (struct hw\_module\_t \*) module;
    adev->device.common.close = adev\_close;
    
    adev->device.set\_master\_mute = adev\_set\_master\_mute;
    adev->device.get\_master\_mute = adev\_get\_master\_mute;
    \*device = &adev->device.common;
}
struct generic\_audio\_device {
  struct audio\_hw\_device device;  // Constant after init
  pthread\_mutex\_t lock;
  unsigned int last\_patch\_id;   // Protected by this->lock
  bool master\_mute;             // Protected by this->lock
  bool mic\_mute;                // Protected by this->lock
  struct mixer \*mixer;          // Protected by this->lock
  Hashmap \*out\_bus\_stream\_map;  // Extended field. Constant after init
};
struct audio\_hw\_device {

    struct hw\_device\_t common;
    
    int (\*set\_master\_mute)(struct audio\_hw\_device \*dev, bool mute);

    int (\*get\_master\_mute)(struct audio\_hw\_device \*dev, bool \*mute);
}

typedef struct hw\_device\_t {
    /\*\* tag must be initialized to HARDWARE\_DEVICE\_TAG \*/
    uint32\_t tag;
    uint32\_t version;

    /\*\* reference to the module this device belongs to \*/
    struct hw\_module\_t\* module;
    int (\*close)(struct hw\_device\_t\* device);

} hw\_device\_t;
```

最后为 module->methods->open(module, AUDIO\_HARDWARE\_INTERFACE,  
TO\_HW\_DEVICE\_T\_OPEN(device));

完整 audio\_hw\_device

Audio.h 还定义了一个 audio\_hw\_device，通过 openDevice 最后的 adev\_open 进行初始化，见 3.5

```
struct audio\_hw\_device {

    struct hw\_device\_t common;

    uint32\_t (\*get\_supported\_devices)(const struct audio\_hw\_device \*dev);

    int (\*init\_check)(const struct audio\_hw\_device \*dev);

    int (\*set\_voice\_volume)(struct audio\_hw\_device \*dev, float volume);

    int (\*set\_master\_volume)(struct audio\_hw\_device \*dev, float volume);

    int (\*get\_master\_volume)(struct audio\_hw\_device \*dev, float \*volume);

    int (\*set\_mode)(struct audio\_hw\_device \*dev, audio\_mode\_t mode);

    int (\*set\_mic\_mute)(struct audio\_hw\_device \*dev, bool state);
    int (\*get\_mic\_mute)(const struct audio\_hw\_device \*dev, bool \*state);

    int (\*set\_parameters)(struct audio\_hw\_device \*dev, const char \*kv\_pairs);

    char \* (\*get\_parameters)(const struct audio\_hw\_device \*dev,
                             const char \*keys);

    size\_t (\*get\_input\_buffer\_size)(const struct audio\_hw\_device \*dev,
                                    const struct audio\_config \*config);

    int (\*open\_output\_stream)(struct audio\_hw\_device \*dev,
                              audio\_io\_handle\_t handle,
                              audio\_devices\_t devices,
                              audio\_output\_flags\_t flags,
                              struct audio\_config \*config,
                              struct audio\_stream\_out \*\*stream\_out,
                              const char \*address);

    void (\*close\_output\_stream)(struct audio\_hw\_device \*dev,
                                struct audio\_stream\_out\* stream\_out);

    int (\*open\_input\_stream)(struct audio\_hw\_device \*dev,
                             audio\_io\_handle\_t handle,
                             audio\_devices\_t devices,
                             struct audio\_config \*config,
                             struct audio\_stream\_in \*\*stream\_in,
                             audio\_input\_flags\_t flags,
                             const char \*address,
                             audio\_source\_t source);

    void (\*close\_input\_stream)(struct audio\_hw\_device \*dev,
                               struct audio\_stream\_in \*stream\_in);

    int (\*get\_microphones)(const struct audio\_hw\_device \*dev,
                           struct audio\_microphone\_characteristic\_t \*mic\_array,
                           size\_t \*mic\_count);

    int (\*dump)(const struct audio\_hw\_device \*dev, int fd);

    int (\*set\_master\_mute)(struct audio\_hw\_device \*dev, bool mute);

    int (\*get\_master\_mute)(struct audio\_hw\_device \*dev, bool \*mute);


    int (\*create\_audio\_patch)(struct audio\_hw\_device \*dev,
                               unsigned int num\_sources,
                               const struct audio\_port\_config \*sources,
                               unsigned int num\_sinks,
                               const struct audio\_port\_config \*sinks,
                               audio\_patch\_handle\_t \*handle);

    int (\*release\_audio\_patch)(struct audio\_hw\_device \*dev,
                               audio\_patch\_handle\_t handle);

    int (\*get\_audio\_port)(struct audio\_hw\_device \*dev,
                          struct audio\_port \*port);

    int (\*set\_audio\_port\_config)(struct audio\_hw\_device \*dev,
                         const struct audio\_port\_config \*config);

};
typedef struct audio\_hw\_device audio\_hw\_device\_t;
```

### 3.4 Hardware device\_open

#### 3.4.1 hw\_get\_module\_by\_class

这是在 Hardware.c (f:\\q\\hardware\\libhardware) 7169 2020/2/22

共用方法

```
int hw\_get\_module\_by\_class(const char \*class\_id, const char \*inst,
                           const struct hw\_module\_t \*\*module)
{
    int i = 0;
    char prop\[PATH\_MAX\] = {0};
    char path\[PATH\_MAX\] = {0};
    char name\[PATH\_MAX\] = {0};
    char prop\_name\[PATH\_MAX\] = {0};

// 处理得到path  class id转 path
    if (inst)
        snprintf(name, PATH\_MAX, "%s.%s", class\_id, inst);
    else
        strlcpy(name, class\_id, PATH\_MAX);

    /\*
     \* Here we rely on the fact that calling dlopen multiple times on
     \* the same .so will simply increment a refcount (and not load
     \* a new copy of the library).
     \* We also assume that dlopen() is thread-safe.
     \*/

    /\* First try a property specific to the class and possibly instance \*/
    snprintf(prop\_name, sizeof(prop\_name), "ro.hardware.%s", name);
    if (property\_get(prop\_name, prop, NULL) > 0) {
        if (hw\_module\_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /\* Loop through the configuration variants looking for a module \*/
    for (i=0 ; i<HAL\_VARIANT\_KEYS\_COUNT; i++) {
        if (property\_get(variant\_keys\[i\], prop, NULL) == 0) {
            continue;
        }
        if (hw\_module\_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /\* Nothing found, try the default \*/
    if (hw\_module\_exists(path, sizeof(path), name, "default") == 0) {
        goto found;
    }

    return -ENOENT;

found:
    /\* load the module, if this fails, we're doomed, and we should not try
     \* to load a different variant. \*/
    return load(class\_id, path, module);
}
```

#### 3.4.2 load 加载

```
static int load(const char \*id,
        const char \*path,
        const struct hw\_module\_t \*\*pHmi)
{
    int status = -EINVAL;
    void \*handle = NULL;
    struct hw\_module\_t \*hmi = NULL;
#ifdef \_\_ANDROID\_VNDK\_\_
    const bool try\_system = false;
#else
    const bool try\_system = true;
#endif

    /\*
     \* load the symbols resolving undefined symbols before
     \* dlopen returns. Since RTLD\_GLOBAL is not or'd in with
     \* RTLD\_NOW the external symbols will not be global
     \*/
    if (try\_system &&
        strncmp(path, HAL\_LIBRARY\_PATH1, strlen(HAL\_LIBRARY\_PATH1)) == 0) {
        /\* If the library is in system partition, no need to check
         \* sphal namespace. Open it with dlopen.
         \*/
  //打开动态链接库
  // path = audio.primary=caremu 模拟器
  // path = audio.primary.default
  // path = audio.primary.msm8996      
        handle = dlopen(path, RTLD\_NOW);
    } else {
#if defined(\_\_ANDROID\_RECOVERY\_\_)
        handle = dlopen(path, RTLD\_NOW);
#else
        handle = android\_load\_sphal\_library(path, RTLD\_NOW);
#endif
    }
    if (handle == NULL) {
        char const \*err\_str = dlerror();
        ALOGE("load: module=%s\\n%s", path, err\_str?err\_str:"unknown");
        status = -EINVAL;
        goto done;
    }
// 根据动态链接库handle 和 符号，返回对应的地址
    /\* Get the address of the struct hal\_module\_info. \*/
    const char \*sym = HAL\_MODULE\_INFO\_SYM\_AS\_STR;
    hmi = (struct hw\_module\_t \*)dlsym(handle, sym);
    if (hmi == NULL) {
        ALOGE("load: couldn't find symbol %s", sym);
        status = -EINVAL;
        goto done;
    }

    /\* Check that the id matches \*/
    if (strcmp(id, hmi->id) != 0) {
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;

    /\* success \*/
    status = 0;

    done:
    if (status != 0) {
        hmi = NULL;
        if (handle != NULL) {
            dlclose(handle);
            handle = NULL;
        }
    } else {
        ALOGV("loaded HAL id=%s path=%s hmi=%p handle=%p",
                id, path, hmi, handle);
    }
// 给 pHmi 复制 Hal层拿到 mod
    \*pHmi = hmi;

    return status;
}
```

#### 3.4.3 hw\_module\_t 赋值返回

audio\_module hw\_module\_t 两个结构体 存的信息一一对应 通过 dlsym 函数 拿到 HAL\_MODULE\_INFO\_SYM 强转成 hw\_module\_t 类型 所以 hw\_module\_t 就是 audio\_hw.c audio\_module HAL\_MODULE\_INFO\_SYM

```
/\*\*
 \* Name of the hal\_module\_info
 \*/
#define HAL\_MODULE\_INFO\_SYM         HMI

/\*\*
 \* Name of the hal\_module\_info as a string
 \*/
#define HAL\_MODULE\_INFO\_SYM\_AS\_STR  "HMI"

static int load(const char \*id,
        const char \*path,
        const struct hw\_module\_t \*\*pHmi)
{
    void \*handle = NULL;
    struct hw\_module\_t \*hmi = NULL;
    handle = dlopen(path, RTLD\_NOW);
   // 根据动态链接库handle 和 符号，返回对应的地址
    /\* Get the address of the struct hal\_module\_info. \*/
    const char \*sym = HAL\_MODULE\_INFO\_SYM\_AS\_STR;
  //dlsym是一个计算机函数，功能是根据动态链接库操作句柄与符号，返回符号对应的地址，不但可以获取函数地址，也可以获取变量地址。  
    //拿到audio\_module  HAL\_MODULE\_INFO\_SYM地址 相当于赋值了
    hmi = (struct hw\_module\_t \*)dlsym(handle, sym); 
    // 给 pHmi 复制 Hal层拿到 mod
    \*pHmi = hmi;
    return status;
}
//3.3.3
int DevicesFactory::loadAudioInterface(const char\* if\_name, audio\_hw\_device\_t\*\* dev) {
	rc = hw\_get\_module\_by\_class(AUDIO\_HARDWARE\_MODULE\_ID, if\_name, &mod);
    // 打开 hw\_debice     audio\_hw\_device\_open 为Audio.h里的方法
    rc = audio\_hw\_device\_open(mod, dev);
}

audio\_module hw\_module\_t两个结构体 存的信息一一对应  通过dlsym函数 拿到HAL\_MODULE\_INFO\_SYM 强转成hw\_module\_t类型
//Audio\_hw.c
struct audio\_module {
    struct hw\_module\_t common;
};

struct audio\_module HAL\_MODULE\_INFO\_SYM = {
    .common = {
        .tag = HARDWARE\_MODULE\_TAG,
        .module\_api\_version = AUDIO\_MODULE\_API\_VERSION\_0\_1,
        .hal\_api\_version = HARDWARE\_HAL\_API\_VERSION,
        .id = AUDIO\_HARDWARE\_MODULE\_ID,
        .name = "Generic car audio HW HAL",
        .author = "The Android Open Source Project",
        .methods = &hal\_module\_methods,
    },
};
//Hardware.h
typedef struct hw\_module\_t {
    uint32\_t tag; 
    uint16\_t module\_api\_version;
#define version\_major module\_api\_version
    uint16\_t hal\_api\_version;
#define version\_minor hal\_api\_version
    const char \*id;
    const char \*name;
    const char \*author;
    struct hw\_module\_methods\_t\* methods;
    void\* dso;
#ifdef \_\_LP64\_\_
    uint64\_t reserved\[32-7\];
#else
    uint32\_t reserved\[32-7\];
#endif

} hw\_module\_t;
```

### 3.5 audio\_hw.c adev\_open

#### 3.5.1 open 方法引用初始化

即 module->methods->open(module, AUDIO\_HARDWARE\_INTERFACE,  
TO\_HW\_DEVICE\_T\_OPEN(device)); 最终是调用的 audio\_hw 里的 adev\_open

```
static struct hw\_module\_methods\_t hal\_module\_methods = {
    .open = adev\_open,
};
```

#### 3.5.2 各家 adev\_open 实现

vendor 是厂商自己定义

```
//使用原生 模拟器代码说明 

struct generic\_audio\_device {
  struct audio\_hw\_device device;  // Constant after init
  pthread\_mutex\_t lock;
  unsigned int last\_patch\_id;   // Protected by this->lock
  bool master\_mute;             // Protected by this->lock
  bool mic\_mute;                // Protected by this->lock
  struct mixer \*mixer;          // Protected by this->lock
  Hashmap \*out\_bus\_stream\_map;  // Extended field. Constant after init
};

static int adev\_open(const hw\_module\_t \*module,
        const char \*name, hw\_device\_t \*\*device) {
    static struct generic\_audio\_device \*adev;

    if (strcmp(name, AUDIO\_HARDWARE\_INTERFACE) != 0)
        return -EINVAL;

    pthread\_mutex\_lock(&adev\_init\_lock);
    if (audio\_device\_ref\_count != 0) {
        \*device = &adev->device.common;
        audio\_device\_ref\_count++;
        ALOGV("%s: returning existing instance of adev", \_\_func\_\_);
        ALOGV("%s: exit", \_\_func\_\_);
        goto unlock;
    }
    adev = calloc(1, sizeof(struct generic\_audio\_device));

    pthread\_mutex\_init(&adev->lock, (const pthread\_mutexattr\_t \*) NULL);
//设置通用引用
    adev->device.common.tag = HARDWARE\_DEVICE\_TAG;
    adev->device.common.version = AUDIO\_DEVICE\_API\_VERSION\_3\_0;
    adev->device.common.module = (struct hw\_module\_t \*) module;
    adev->device.common.close = adev\_close;
//设置方法引用
    adev->device.init\_check = adev\_init\_check;               // no op
    adev->device.set\_voice\_volume = adev\_set\_voice\_volume;   // no op
    adev->device.set\_master\_volume = adev\_set\_master\_volume; // no op
    adev->device.get\_master\_volume = adev\_get\_master\_volume; // no op
//  设置静音   
    adev->device.set\_master\_mute = adev\_set\_master\_mute;
    adev->device.get\_master\_mute = adev\_get\_master\_mute;
    adev->device.set\_mode = adev\_set\_mode;                   // no op
    adev->device.set\_mic\_mute = adev\_set\_mic\_mute;
    adev->device.get\_mic\_mute = adev\_get\_mic\_mute;
    adev->device.set\_parameters = adev\_set\_parameters;       // no op
    adev->device.get\_parameters = adev\_get\_parameters;       // no op
    adev->device.get\_input\_buffer\_size = adev\_get\_input\_buffer\_size;
    adev->device.open\_output\_stream = adev\_open\_output\_stream;
    adev->device.close\_output\_stream = adev\_close\_output\_stream;
    adev->device.open\_input\_stream = adev\_open\_input\_stream;
    adev->device.close\_input\_stream = adev\_close\_input\_stream;
    adev->device.dump = adev\_dump;

    // New in AUDIO\_DEVICE\_API\_VERSION\_3\_0
    adev->device.set\_audio\_port\_config = adev\_set\_audio\_port\_config;
    adev->device.create\_audio\_patch = adev\_create\_audio\_patch;
    adev->device.release\_audio\_patch = adev\_release\_audio\_patch;

    \*device = &adev->device.common;

    adev->mixer = mixer\_open(PCM\_CARD);
    struct mixer\_ctl \*ctl;

    // Set default mixer ctls
    // Enable channels and set volume
    for (int i = 0; i < (int)mixer\_get\_num\_ctls(adev->mixer); i++) {
        ctl = mixer\_get\_ctl(adev->mixer, i);
        ALOGD("mixer %d name %s", i, mixer\_ctl\_get\_name(ctl));
        if (!strcmp(mixer\_ctl\_get\_name(ctl), "Master Playback Volume") ||
            !strcmp(mixer\_ctl\_get\_name(ctl), "Capture Volume")) {
            for (int z = 0; z < (int)mixer\_ctl\_get\_num\_values(ctl); z++) {
                ALOGD("set ctl %d to %d", z, 100);
                mixer\_ctl\_set\_percent(ctl, z, 100);
            }
            continue;
        }
        if (!strcmp(mixer\_ctl\_get\_name(ctl), "Master Playback Switch") ||
            !strcmp(mixer\_ctl\_get\_name(ctl), "Capture Switch")) {
            for (int z = 0; z < (int)mixer\_ctl\_get\_num\_values(ctl); z++) {
                ALOGD("set ctl %d to %d", z, 1);
                mixer\_ctl\_set\_value(ctl, z, 1);
            }
            continue;
        }
    }

    // Initialize the bus address to output stream map
    adev->out\_bus\_stream\_map = hashmapCreate(5, str\_hash\_fn, str\_eq);

    audio\_device\_ref\_count++;

unlock:
    pthread\_mutex\_unlock(&adev\_init\_lock);
    return 0;
}
```