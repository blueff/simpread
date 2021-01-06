> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/sinat_18179367/article/details/106000100)

Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）
===============================================================================

随手笔记系列，内容较多较繁琐，用于结合目录快速查询

本文将通过 AudioManger 的 setMastermute Api 的通信传递实现逻辑，自上而下梳理一下调用顺序和各模块相关初始化，最终到的 audio_hw.c 来实现静音

### 文章目录

*   [Android Q Audio 自上而下的调用逻辑和各模块相关初始化（AudioFlinger 通信、Audio hal 加载、AudioDevice 选择）](#Android_Q_Audio_AudioFlingerAudio_hal_AudioDevice__0)
*   *   [二、native 层通信](#native_7)
    *   *   [2.1 AudioFlinger](#21_AudioFlinger_9)
        *   *   [2.1.1 AudioServce 启动](#211_AudioServce_19)
            *   [2.1.2 AudioFlinger 初始化](#212_AudioFlinger_76)
            *   [2.1.3 AudioFlinger 的获取](#213_AudioFlinger_161)
            *   [2.1.4 IAudioFlinger 介绍](#214_IAudioFlinger__213)
            *   [2.1.5 BpAudioFlinger](#215_BpAudioFlinger_230)
            *   [2.1.6 BnAudioFlinger](#216_BnAudioFlinger_254)
            *   [2.1.7 AudioFlinger setMasterMute 逻辑](#217_AudioFlinger_setMasterMute_272)
        *   [2.2 AudioHAL 加载](#22_AudioHAL_323)
        *   *   [2.2.1 DevicesFactoryHalInterface create()](#221_DevicesFactoryHalInterface_create_337)
            *   [2.2.2 DevicesFactoryHalHybrid device 工厂](#222_DevicesFactoryHalHybrid_device_381)
            *   [2.2.3 DevicesFactoryHalHidl 实例化](#223_DevicesFactoryHalHidl__409)

二、native 层通信
------------

### 2.1 AudioFlinger

AudioFlinger 是两个 audio nativeservice 中的一个，用于管理声音的播放和录制，volume 和硬件相关设置也在 audioFlinger 中进行

AudioSystem 到 AudioFlinger 大概流程为

AudioSystem -> BpAudioFlinger -> binder -> BnAudioFlinger -> AudioFlinger

#### 2.1.1 AudioServce 启动

解析 AudioServer.rc

Audioserver.rc (f:\q\frameworks\av\media\audioserver) 794 2020/2/22

```
service audioserver /system/bin/audioserver
    class core
    user audioserver
    # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
    group audio camera drmrpc inet media mediadrm net_bt net_bt_admin net_bw_acct wakelock
    capabilities BLOCK_SUSPEND
    ioprio rt 4
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
    onrestart restart vendor.audio-hal-2-0
    onrestart restart vendor.audio-hal-4-0-msd
    # Keep the original service name for backward compatibility when upgrading
    # O-MR1 devices with framework-only.
    onrestart restart audio-hal-2-0

on property:vts.native_server.on=1
    stop audioserver
on property:vts.native_server.on=0
    start audioserver

on init
    mkdir /dev/socket/audioserver 0775 audioserver audioserver
```

Android.mk

rameworks/av/media/audioserver/Android.mk

```
LOCAL_SRC_FILES := \
  main_audioserver.cpp \
LOCAL_C_INCLUDES := \
  frameworks/av/services/audioflinger \
  frameworks/av/services/audiopolicy \
LOCAL_MODULE := audioserver

LOCAL_INIT_RC := audioserver.rc
```

Main_audioserver.cpp (f:\q\frameworks\av\media\audioserver) 5892 2020/2/22

```
int main(int argc __unused, char **argv)
{
        AudioFlinger::instantiate();
        AudioPolicyService::instantiate();
}
```

#### 2.1.2 AudioFlinger 初始化

AudioFlinger::instantiate(); 实际上是 BinderService 的 instantiate

```
template<typename SERVICE>
class BinderService
{
    static status_t publish(bool allowIsolated = false,
                            int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
        sp<IServiceManager> sm(defaultServiceManager());
        //实例化 并add到 service_manager里
        return sm->addService(String16(SERVICE::getServiceName()), new SERVICE(), allowIsolated,
                              dumpFlags);
    }

	static void instantiate() { publish(); }
}
//AudioFlinger 继承BinderService SERVICE为AudioFlinger
class AudioFlinger :
    public BinderService<AudioFlinger>,
    public BnAudioFlinger
```

new SERVICE() 实际上是 AudioFlinger 的构造函数，就存在一个 AF 的类实例，可以通过 service_manager 的 getService 访问，再之后 onFirstRef 会被执行。onFirstRef（）属于其父类 RefBase，该函数在强引用 sp 新增引用计数时调用。就是当 有 sp 包装的类初始化的时候调用。

AudioFinger 继承 BnAudioFlinger

```
AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),
      mMediaLogNotifier(new AudioFlinger::MediaLogNotifier()),
      mPrimaryHardwareDev(NULL),
      mAudioHwDevs(NULL),
      mHardwareStatus(AUDIO_HW_IDLE),
      mMasterVolume(1.0f),
      mMasterMute(false),
      // mNextUniqueId(AUDIO_UNIQUE_ID_USE_MAX),
      mMode(AUDIO_MODE_INVALID),
      mBtNrecIsOff(false),
      mIsLowRamDevice(true),
      mIsDeviceTypeKnown(false),
      mTotalMemory(0),
      mClientSharedHeapSize(kMinimumClientSharedHeapSizeBytes),
      mGlobalEffectEnableTime(0),
      mPatchPanel(this),
      mSystemReady(false)
{

//device hal 加载
    mDevicesFactoryHal = DevicesFactoryHalInterface::create();
//effect hal 加载          
    mEffectsFactoryHal = EffectsFactoryHalInterface::create();

    mMediaLogNotifier->run("MediaLogNotifier");
}

void AudioFlinger::onFirstRef()
{
    //初始化audio mode
    mMode = AUDIO_MODE_NORMAL;
    gAudioFlinger = this;
}
```

BnAudioFlinger

BnAudioFlinger 作为 AudioSystem -> AF 的客户端接口

IAudioFlinger.h (f:\q\frameworks\av\media\libaudioclient\include\media) 22638 2020/2/22

```
class BnAudioFlinger : public BnInterface<IAudioFlinger>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);

    // Requests media.log to start merging log buffers
    virtual void requestLogMerge() = 0;
};
```

#### 2.1.3 AudioFlinger 的获取

AudioSystem 中有 get_audio_flinger() 方法获取 BpAudioFlinger，通过 binder 与 audioFlinger 进行通信的

```
const sp<IAudioFlinger> AudioSystem::get_audio_flinger()
{
    sp<IAudioFlinger> af;
    sp<AudioFlingerClient> afc;
    {
        Mutex::Autolock _l(gLock);
        if (gAudioFlinger == 0) {
            //通过handle 0 binder通信 获取service_manager
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;
            do {
                //get 之前add的service
                binder = sm->getService(String16("media.audio_flinger"));
                if (binder != 0)
                    break;
                ALOGW("AudioFlinger not published, waiting...");
                usleep(500000); // 0.5 s
            } while (true);
            if (gAudioFlingerClient == NULL) {
                //创建AudioFlingerClient bn端 用于 audioFlinger ->audioSystem的通信
                gAudioFlingerClient = new AudioFlingerClient();
            } else {
                if (gAudioErrorCallback) {
                    gAudioErrorCallback(NO_ERROR);
                }
            }
            binder->linkToDeath(gAudioFlingerClient);
            //强转
            gAudioFlinger = interface_cast<IAudioFlinger>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioFlinger == 0);
            afc = gAudioFlingerClient;
            // Make sure callbacks can be received by gAudioFlingerClient
            //启动binder线程池，用于af能找到这个AudioFlingerClient bn端 
            ProcessState::self()->startThreadPool();
        }
        af = gAudioFlinger;
    }
    if (afc != 0) {
        int64_t token = IPCThreadState::self()->clearCallingIdentity();
        af->registerClient(afc);
        IPCThreadState::self()->restoreCallingIdentity(token);
    }
    return af;
}
```

#### 2.1.4 IAudioFlinger 介绍

IAudioFlinger 是继承了 Interface，定义了 AudioSystem 和 AudioFlinger 之间通信的接口，类似 java 层的 aidl，其中参数数据使用 Parcel 进行传输

IAudioFlinger.h (f:\q\frameworks\av\media\libaudioclient\include\media) 22638 2020/2/22

```
class IAudioFlinger : public IInterface
{
    virtual     status_t    setMasterVolume(float value) = 0;
    virtual     status_t    setMasterMute(bool muted) = 0;

    virtual     float       masterVolume() const = 0;
    virtual     bool        masterMute() const = 0;
}
```

#### 2.1.5 BpAudioFlinger

通过 Parcel 写入数据 remote 可简单理解最后调用 Bn 端的 AudioFlinger，其实涉及到复杂的 binder 驱动

```
class BpAudioFlinger : public BpInterface<IAudioFlinger>
{
public:
    explicit BpAudioFlinger(const sp<IBinder>& impl)
        : BpInterface<IAudioFlinger>(impl)
    {
    }
    virtual status_t setMasterMute(bool muted)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IAudioFlinger::getInterfaceDescriptor());
        data.writeInt32(muted);
        //remote 
        remote()->transact(SET_MASTER_MUTE, data, &reply);
        return reply.readInt32();
    }
}
```

#### 2.1.6 BnAudioFlinger

通过 reply 写入返回值，调用 BnAudioFlinger 的 setMasterMute 方法

```
status_t BnAudioFlinger::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
	switch (code) {
        case SET_MASTER_MUTE: {
            CHECK_INTERFACE(IAudioFlinger, data, reply);
            reply->writeInt32( setMasterMute(data.readInt32()) );
            return NO_ERROR;
        } break;

}
```

#### 2.1.7 AudioFlinger setMasterMute 逻辑

如果 hal 支持 setMasterMute 用 hal 的 不支持通过设置 volume 0 来实现，接下来我们先分析 hal 的逻辑，本文主要是通过这个 setMaserMute API 来介绍 Audio 模块给层级间关系，volume 方式后续补充

AudioHwDevice *dev dev->hwDevice()->setMasterMute(muted); 通过 loadHwModule 拿到 DeviceFactory 生成

```
class AudioFlinger :
    public BinderService<AudioFlinger>,
    public BnAudioFlinger
{
	virtual     bool        streamMute(audio_stream_type_t stream) const;
}

status_t AudioFlinger::setMasterMute(bool muted)
{
    Mutex::Autolock _l(mLock);
    mMasterMute = muted;
//如果 hal 支持 setMasterMute 用hal的 不支持通过设置volume 0来实现	
    // Set master mute in the HALs which support it.
    //循环查找所有的hwdevice
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        AutoMutex lock(mHardwareLock);
        AudioHwDevice *dev = mAudioHwDevs.valueAt(i);

        mHardwareStatus = AUDIO_HW_SET_MASTER_MUTE;
        if (dev->canSetMasterMute()) {
            //调用到hal的加载
            dev->hwDevice()->setMasterMute(muted);
        }
        mHardwareStatus = AUDIO_HW_IDLE;
    }
//不支持通过设置volume 0来实现
    // Now set the master mute in each playback thread.  Playback threads
    // assigned to HALs which do not have master mute support will apply master
    // mute during the mix operation.  Threads with HALs which do support master
    // mute will simply ignore the setting.
    Vector<VolumeInterface *> volumeInterfaces = getAllVolumeInterfaces_l();
    for (size_t i = 0; i < volumeInterfaces.size(); i++) {
        volumeInterfaces[i]->setMasterMute(muted);
    }

    return NO_ERROR;
}
//getMasterMute 最后在这返回mute状态
bool AudioFlinger::masterMute_l() const
{
    return mMasterMute;
}
```

### 2.2 AudioHAL 加载

libAudioHAL 是用于初始化 Hal 加载的

之前 AudioFlinger 在构造函数里创建了 DeviceFactory，来看看这个 DeviceFactory 怎么和 hal 层挂钩的

```
AudioFlinger::AudioFlinger()
    : BnAudioFlinger(),
{
    mDevicesFactoryHal = DevicesFactoryHalInterface::create();
}
```

#### 2.2.1 DevicesFactoryHalInterface create()

```
//DevicesFactoryHalInterface.h (f:\q\frameworks\av\media\libaudiohal\include\media\audiohal)
class DevicesFactoryHalInterface : public RefBase
{
    virtual status_t openDevice(const char *name, sp<DeviceHalInterface> *device) = 0;

    static sp<DevicesFactoryHalInterface> create();
}
//DevicesFactoryHalInterface.cpp
//三个 hal文件 IDeviceFactory.hal
#include <android/hardware/audio/2.0/IDevicesFactory.h>
#include <android/hardware/audio/4.0/IDevicesFactory.h>
#include <android/hardware/audio/5.0/IDevicesFactory.h>
#include <libaudiohal/FactoryHalHidl.h>
//优先级 5.0》 4.0 》 2.0  通过getService（固定API） 获取hal层service 
sp<DevicesFactoryHalInterface> DevicesFactoryHalInterface::create() {
    if (hardware::audio::V5_0::IDevicesFactory::getService() != nullptr) {
        return V5_0::createDevicesFactoryHal();
    }
    if (hardware::audio::V4_0::IDevicesFactory::getService() != nullptr) {
        return V4_0::createDevicesFactoryHal();
    }
    if (hardware::audio::V2_0::IDevicesFactory::getService() != nullptr) {
        return V2_0::createDevicesFactoryHal();
    }
    return nullptr;
}

//FactoryHalHidl.h 定义了3个命名空间
namespace V2_0 {
sp<DevicesFactoryHalInterface> createDevicesFactoryHal();
} // namespace V2_0

namespace V4_0 {
sp<DevicesFactoryHalInterface> createDevicesFactoryHal();
} // namespace V4_0

namespace V5_0 {
sp<DevicesFactoryHalInterface> createDevicesFactoryHal();
} // namespace V5_0
```

#### 2.2.2 DevicesFactoryHalHybrid device 工厂

DevicesFactoryHalHybrid 包含 Local 和 Hidl，8.0 之后优先 Hidl，兼容 Local

在这 base 为 Q，就不分析 Local，只梳理 Hidl

```
class DevicesFactoryHalHybrid : public DevicesFactoryHalInterface
{
    sp<DevicesFactoryHalInterface> createDevicesFactoryHal() {
        return new DevicesFactoryHalHybrid();
    }
}
// DevicesFactoryHalHybrid.cpp  
DevicesFactoryHalHybrid::DevicesFactoryHalHybrid()
        : mLocalFactory(new DevicesFactoryHalLocal()),
          mHidlFactory(new DevicesFactoryHalHidl()) {
}
// openDevice流程
status_t DevicesFactoryHalHybrid::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mHidlFactory != 0 && strcmp(AUDIO_HARDWARE_MODULE_ID_A2DP, name) != 0 &&
        strcmp(AUDIO_HARDWARE_MODULE_ID_HEARING_AID, name) != 0) {
        return mHidlFactory->openDevice(name, device);
    }
    return mLocalFactory->openDevice(name, device);
}
```

#### 2.2.3 DevicesFactoryHalHidl 实例化

```
//DevicesFactoryHalHidl.cpp
DevicesFactoryHalHidl::DevicesFactoryHalHidl() {
    //获取hal层service
    sp<IDevicesFactory> defaultFactory{IDevicesFactory::getService()};
    if (!defaultFactory) {
        ALOGE("Failed to obtain IDevicesFactory/default service, terminating process.");
        exit(1);
    }
    // 将hal层 的IDeviceFactory 放入集合
    mDeviceFactories.push_back(defaultFactory);
    if (MAJOR_VERSION >= 4) {
        // The MSD factory is optional and only available starting at HAL 4.0
        sp<IDevicesFactory> msdFactory{IDevicesFactory::getService(AUDIO_HAL_SERVICE_NAME_MSD)};
        if (msdFactory) {
            mDeviceFactories.push_back(msdFactory);
        }
    }
    for (const auto& factory : mDeviceFactories) {
        // It is assumed that the DevicesFactoryHalInterface instance is owned
        // by AudioFlinger and thus have the same lifespan.
        factory->linkToDeath(HalDeathHandler::getInstance(), 0 /*cookie*/);
    }
}
```