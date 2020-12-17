> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u012440406/article/details/54883220)

Android 支持多种设备的的输出。一台正常的机子，本身就自带话筒，扬声器，麦克风等多个声音输入输出设备，再加上五花八门的外置设备（通过耳机，蓝牙，wifi 等方式连接），使声音的输出更具多样性。Android 支持如此多的设备连接，那么 android 内部是怎样对设备的输出输出进行控制的呢？这一次我们主要来看看音频通路的切换。

音频流、设备、音频策略
===========

  

要想知道 Andorid 是怎样对设备的输出输出进行控制的，我们首先来了解一些音频相关的基本知识： stream_type、content_type、devices、routing_strategy。  
stream_type: 音频流的类型。在当前系统中，Android(6.0) 一共定义了 11 种 stream_type 以供开发者使用。Android 上层开发要想要发出声音，都必须先确定当前当前的音频类型。  
content_type: 具体输出类型。虽然当前一共有 11 种 stream_type, 但一旦进入到 Attribute，Android 就只将其整理成几种类型。这才是实际的类型。  
device: 音频输入输出设备。Android 定义了多种设备输入输出设备（具体物理设备可能还是那几个，但是输出场景不尽相同）。  
routing_strategy: 音频路由策略。默认情况下，Android 是根据路由策略去选择设备负责输出输入音频的。  

stream_type 在 android 中 java 层与 c++ 层均有定义。并且对应的值是保持一致的。

![](https://img-blog.csdn.net/20170205214137789?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

device 与 stream_type 一样，在 java 层和 C++ 层均有定义，并且会根据使用的情况不同蕴含多个定义：

![](https://img-blog.csdn.net/20170205214429386?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

相对 stream_type，routing_strategy 只是一个定义在 RountingStrategy.h 的一个简单的枚举结构体：

![](https://img-blog.csdn.net/20170205214527965?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

  

usecase 只是 qcom 内部定义的一个数据结构，位于 hal 层，用作处理处理内部声卡逻辑和输出方案。输出方案与声卡中的 mixer_path_xxx.xml 相联。而 mixer_path 等相关文件，才是具体的音频输出方案。

![](https://img-blog.csdn.net/20170205215213506?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

  

我们通过查看当前的声卡情况确定了当前具体的 mixer_path 文件——mixer_path_skue.xml。xml 文件内部就是我们预定义的 usecase 的具体情况：

![](https://img-blog.csdn.net/20170205215221393?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

在 mixer_path 类文件中，一个标准的 path 就如上面的红框那样。有名字，有一定的参数。另外，一个 patch 之中，还可以嵌套另外一个 patch。  

由于 usecase 只是目前高通 hal 层独有的定义，所以本文不会花太多时间和精力去探讨 usecase 的相关设置和内容。目前来说，对这个有一定的认知就可。

AudioPolicy 和 AudioPolicyService
--------------------------------

在了解完 Audio 一些基本的定义设定之后，我们来看一下 Android 的 Audio 整体架构。  
Audio 内部系统从上到下包含各方面的东西。对于声音输出的设备的选择与切换，我们主要需要关注 2 个地方。第一处，是外接设备如耳机，蓝牙设备等连接的通知。第二处就是 Audio 系统中核心的 AudioFinger 与 AudioPolicyService 的处理内容。  
AudioFinger 是 Audio 系统的工作引擎，管理者系统中输入输出音频流，并承担音频数据混音，以及读写 Audio 硬件等工作以实现数据的输入输出功能。AudioPolicyService 是 Audio 系统策略控制中心，具体负责掌管系统中声音设备的选择和切换，音量控制等功能。  

AudioFinger 与 AudioPolicyService 的类图关系：  

![](https://img-blog.csdn.net/20170205215825725?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

  

  

在 AudioFlinger 和 AudioPolicyService 的运作中其实包含着很多类，但同时，我们也可以发现，在 Audio 系统中， AudioFinger 与 AudioPolicyService 是紧密相连的。总得来说，AudioFinger 与 AudioPolicyService 是 Audio 系统的核心。所以下面我们很多内容的主角，都是他们 2 个。  
基本的声音输出调用  
发出声音是 Android 机器的一个最基本的功能。但是，Android 是怎么发出声音的呢？就算不连接外设，Android 最基本还有听筒和扬声器 2 个设备。那么，Android 内部，是怎么控制他们 2 个发出声音的呢？下面我们来具体看 一下 Android 一般情况下发出声音时选择设备的过程。  
我们要想分析 Android 中的声音输出，当然是先通过播放音频去一步一步了解 Android 是怎恶魔输出声音的。下面我们以一个最简单的 AudioTrack 播放音频为例，来看下 Android 的发生过程。  
一个简单的 AudioTrack 播放的例子如下：

  

```
AudioTrack audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC, 11025/2, 
                                        AudioFormat.CHANNEL_CONFIGURATION_MONO, 
                                        AudioFormat.ENCODING_PCM_16BIT,
                                        audioLength, AudioTrack.MODE_STREAM);
        audioTrack.play();
        audioTrack.write(audioData, 0, sizeInBytes);
```

  
AudioTrack 在接收参数创建的时候，就会将设置的 steamtype 保存在对应的 AudioAttributes 当中（AudioAttributes 是一个描述关于音频流的信息的属性集合的类）。  
   

我们知道，在 android 系统中，系统封装的对象是一层一层往下调用的。所以，在我们创建了一个 java 的 AudioTrack 对象的时候，其实在同时，在 C++ 当中，我们已经做了很多操作了。下面我们来看一下，AudioTrack 对象创建时，主要做了什么：

```
static jint
android_media_AudioTrack_setup(JNIEnv *env, jobject thiz, jobject weak_this,
        jobject jaa,
        jint sampleRateInHertz, jint channelPositionMask, jint channelIndexMask,
        jint audioFormat, jint buffSizeInBytes, jint memoryMode, jintArray jSession) {
 
    //……
    // create the native AudioTrack object
    sp<AudioTrack> lpTrack = new AudioTrack();
 
    //……
    // initialize the native AudioTrack object
    status_t status = NO_ERROR;
    switch (memoryMode) {
    case MODE_STREAM:
 
        status = lpTrack->set(
                AUDIO_STREAM_DEFAULT,
                sampleRateInHertz,
                format,
                nativeChannelMask,
                frameCount,
                AUDIO_OUTPUT_FLAG_NONE,
                audioCallback, &(lpJniStorage->mCallbackData),
                0,
                0,
                true,
                sessionId,
                AudioTrack::TRANSFER_SYNC,
                NULL,                         
                -1, -1,                       
                paa);
        break;
    //……
    if (status != NO_ERROR) {
        ALOGE("Error %d initializing AudioTrack", status);
        goto native_init_failure;
    }
    // save our newly created C++ AudioTrack in the "nativeTrackInJavaObj" field
    // of the Java object (in mNativeTrackInJavaObj)
    setAudioTrack(env, thiz, lpTrack);
   //……
 
}
```

从上面的代码可以看出，在创建 java 层的 AudioTrack 对象时，对应的 jni 也创建出一个 C++ 的 AudioTrack 对象，并且传入了部分参数和调用了其方法。  

接下来我们来看看 C++ 的 AudioTrack 对象的构造方法：

```
AudioTrack::AudioTrack()
    : mStatus(NO_INIT),
      mIsTimed(false),
      mPreviousPriority(ANDROID_PRIORITY_NORMAL),
      mPreviousSchedulingGroup(SP_DEFAULT),
      mPausedPosition(0),
      mSelectedDeviceId(AUDIO_PORT_HANDLE_NONE),
      mPlaybackRateSet(false)
{
    mAttributes.content_type = AUDIO_CONTENT_TYPE_UNKNOWN;
    mAttributes.usage = AUDIO_USAGE_UNKNOWN;
    mAttributes.flags = 0x0;
    strcpy(mAttributes.tags, "");
}
```

我们可以看到，AudioTrack 的无参构造方法只是进行了一些参数的初始化，那么，具体是 AudioTrack 初始化是进行在哪里呢？  

我们再回到上面，发现 jni 层在创建完 AudioTrack 对象后，根据 memoryMode 的不同而进行了不同的 AudioTrack->set() 操作，只是因为 AudioTrack 提供 2 种不同的输出方式（对内存的影响和要求不同）。我来看看看 set 中主要的操作：

```
status_t AudioTrack::set(…){
    //……
    status_t status = createTrack_l();
    if (status != NO_ERROR) {
        if (mAudioTrackThread != 0) {
            mAudioTrackThread->requestExit();   // see comment in AudioTrack.h
            mAudioTrackThread->requestExitAndWait();
            mAudioTrackThread.clear();
        }
        return status;
   //……
    }
```

在 AudioTrack 的 set() 中，除了部分的参数判断和设置之外，我们可以看到，他调用了自身的 createTrack_l() 进行了进一步的设置。

```
status_t AudioTrack::createTrack_l()
{
    const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
    if (audioFlinger == 0) {
        ALOGE("Could not get audioflinger");
        return NO_INIT;
    }
 
    audio_io_handle_t output;
    audio_stream_type_t streamType = mStreamType;
    audio_attributes_t *attr = (mStreamType == AUDIO_STREAM_DEFAULT) ? &mAttributes : NULL;
    //……
    audio_offload_info_t tOffloadInfo = AUDIO_INFO_INITIALIZER;
    if (mPlaybackRateSet == true && mOffloadInfo == NULL && mFormat == AUDIO_FORMAT_PCM_16_BIT) {
        mOffloadInfo = &tOffloadInfo;
    }
    status_t status = AudioSystem::getOutputForAttr(attr, &output,
                                           (audio_session_t)mSessionId, &streamType, mClientUid,
                                           mSampleRate, mFormat, mChannelMask,
                                           mFlags, mSelectedDeviceId, mOffloadInfo);
 
    //……
    IAudioFlinger::track_flags_t trackFlags = IAudioFlinger::TRACK_DEFAULT;
    //……
    sp<IAudioTrack> track = audioFlinger->createTrack(streamType,
                                                      mSampleRate,
                                                      mFormat,
                                                      mChannelMask,
                                                      &temp,
                                                      &trackFlags,
                                                      mSharedBuffer,
                                                      output,
                                                      tid,
                                                      &mSessionId,
                                                      mClientUid,
                                                      &status);
   //……
```

上面的代码可以看出，AudioTrack 从这里开始，与 AudioFlinger 等进行大量的交互：获取句柄，获取输出，创建 IAudioTrack 指针对象等等。所以接下来，就是 AudioFlinger 的相关内容了。在这里，我们先简单总结下 AudioTrack 的创建过程：  

![](https://img-blog.csdn.net/20170205220833247?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  

根据 AudioTrack 的性质，Java 层在创建完成 AudioTrack 对象直接调用 play() 和 write() 操作，那么其实从另一方面我们可以猜想，在 Java 层创建完成 AudioTrack 之后，系统已经设置好输出的设备等等操作，只等调用 play() 和 write 方法进行播放。所以为了验证我们的猜想，接下来针对 AudioFlinger&AudioSystem 的相关具体分析验证。

AudioFlinger&AudioPolicyService 的控制过程
-------------------------------------

回到上面的内容，我们可以看到，AudioTrack 在调用 createTrack_l() 的方法的时候，开始通过 AudioSystem 获取 output。所以下面我们来看看 AudioSystem 的 getOutputForAttr():

```
status_t AudioSystem::getOutputForAttr()
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return NO_INIT;
    return aps->getOutputForAttr(attr, output, session, stream, uid,
                                 samplingRate, format, channelMask,
                                 flags, selectedDeviceId, offloadInfo);
}
```

从上面我们可以看到，AudioSystem 只是作为一个过渡，然后通过获取 AudioPolicyService 的句柄去 getOutputForAttr()。我们继续跟踪 AudioPolicyService 的情况，会发现其实他只是在 AudioPolicyService 中也只是作为一个过渡，真正进行 getOutputForAttr() 的，在 AudioPolicyManager 之中。

```
status_t AudioPolicyManager::getOutputForAttr()
{
    //……
    *stream = streamTypefromAttributesInt(&attributes);
    sp<DeviceDescriptor> deviceDesc;
    for (size_t i = 0; i < mAvailableOutputDevices.size(); i++) {
        if (mAvailableOutputDevices[i]->getId() == selectedDeviceId) {
            deviceDesc = mAvailableOutputDevices[i];
            break;
        }
    }
    mOutputRoutes.addRoute(session, *stream, SessionRoute::SOURCE_TYPE_NA, deviceDesc, uid);
    
    //根据strategy获取device
    routing_strategy strategy = (routing_strategy) getStrategyForAttr(&attributes);
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
 
    if ((attributes.flags & AUDIO_FLAG_HW_AV_SYNC) != 0) {
        flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_HW_AV_SYNC);
    }
 
    *output = getOutputForDevice(device, session, *stream,
                                 samplingRate, format, channelMask,
                                 flags, offloadInfo);
    //……
```

在 AudioPolicyManager 的 getOutputForAttr() 中，我们可以发现关键点在 strategy 的获取与 device 的获取当中。而在这当中，关键的参数恰恰是在先前从 java 层一步一步封装的过来的 attributes。我们先来简单看一下 attributes 这个参数的数据结构：  

 ![](https://img-blog.csdn.net/20170205221127141?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

从 audio_attributes_t 的结构我们可以看出，audio_attributes_t 保存着需要输出音频的应用的相关配置信息。

然后，根据刚刚的代码，我们来了解一下 strategy 的获取：

  

```
uint32_t AudioPolicyManager::getStrategyForAttr(const audio_attributes_t *attr) {
    // flags to strategy mapping
    if ((attr->flags & AUDIO_FLAG_BEACON) == AUDIO_FLAG_BEACON) {
        return (uint32_t) STRATEGY_TRANSMITTED_THROUGH_SPEAKER;
    }
    if ((attr->flags & AUDIO_FLAG_AUDIBILITY_ENFORCED) == AUDIO_FLAG_AUDIBILITY_ENFORCED) {
        return (uint32_t) STRATEGY_ENFORCED_AUDIBLE;
    }
    // usage to strategy mapping
    return static_cast<uint32_t>(mEngine->getStrategyForUsage(attr->usage));
```

虽然在这里，会先对 flags 参数进行比较，但是，在实际上 flags 大部分时候都是 0。所以最后，都是根据 “mEngine->getStrategyForUsage(attr->usage)” 去选择 StrategyForUsage。当然，再到下一步就到了就是 switch 和 case 的过程，这里就不继续展开了。

在获取到 strategy 之后，我们来看看 Audio 接着是怎么来确定 device 的。

先继续看 AudioPolicyManager 的 getDeviceForStrategy():

```
audio_devices_t AudioPolicyManager::getDeviceForStrategy(routing_strategy strategy,
                                                         bool fromCache)
{
    // Routing
    // see if we have an explicit route
    // scan the whole RouteMap, for each entry, convert the stream type to a strategy
    // (getStrategy(stream)).
    // if the strategy from the stream type in the RouteMap is the same as the argument above,
    // and activity count is non-zero
    // the device = the device from the descriptor in the RouteMap, and exit.
    for (size_t routeIndex = 0; routeIndex < mOutputRoutes.size(); routeIndex++) {
        sp<SessionRoute> route = mOutputRoutes.valueAt(routeIndex);
        routing_strategy strat = getStrategy(route->mStreamType);
        bool strategyMatch = (strat == strategy) ||
                             ((strategy == STRATEGY_ACCESSIBILITY) &&
                              ((mEngine->getStrategyForUsage(
                                      AUDIO_USAGE_ASSISTANCE_ACCESSIBILITY) == strat) ||
                               (strat == STRATEGY_MEDIA)));
        if (strategyMatch && route->isActive()) {
            return route->mDeviceDescriptor->type();
        }
    }
    if (fromCache) {
        ALOGVV("getDeviceForStrategy() from cache strategy %d, device %x",
              strategy, mDeviceForStrategy[strategy]);
        return mDeviceForStrategy[strategy];
    }
    return mEngine->getDeviceForStrategy(strategy);
}
```

调用 AudioPolicyManager 的 getDeviceForStrategy() 的时候，一般会先查下一下当前的 RouteMap，看看有没有匹配的情况的。但由于我们新申请一个 output 的时候，传入的参数是 false，所以这个时候，是会直接通过 mEngine 去直接获取 device。

而在 mEngine 中，getDeviceForStrategy() 又是一堆的选择判断，然后返回设备：

```
audio_devices_t Engine::getDeviceForStrategy(routing_strategy strategy) const
{
    const DeviceVector &availableOutputDevices = mApmObserver->getAvailableOutputDevices();
    const DeviceVector &availableInputDevices = mApmObserver->getAvailableInputDevices();
 
    const SwAudioOutputCollection &outputs = mApmObserver->getOutputs();
 
    uint32_t device = AUDIO_DEVICE_NONE;
    uint32_t availableOutputDevicesType = availableOutputDevices.types();
 
    switch (strategy) {
    //……
    case STRATEGY_MEDIA: {
        uint32_t device2 = AUDIO_DEVICE_NONE;
 
        if (isInCall() && (device == AUDIO_DEVICE_NONE)) {
            // when in call, get the device for Phone strategy
            device = getDeviceForStrategy(STRATEGY_PHONE);
            break;
        }
 
        if (strategy != STRATEGY_SONIFICATION) {
            // no sonification on remote submix (e.g. WFD)
            if (availableOutputDevices.getDevice(AUDIO_DEVICE_OUT_REMOTE_SUBMIX, String8("0")) != 0) {
                device2 = availableOutputDevices.types() & AUDIO_DEVICE_OUT_REMOTE_SUBMIX;
            }
        }
        if (isInCall() && (strategy == STRATEGY_MEDIA)) {
            device = getDeviceForStrategy(STRATEGY_PHONE);
            break;
        }
        if ((device2 == AUDIO_DEVICE_NONE) &&
                (mForceUse[AUDIO_POLICY_FORCE_FOR_MEDIA] != AUDIO_POLICY_FORCE_NO_BT_A2DP) &&
                (outputs.getA2dpOutput() != 0)) {
            device2 = availableOutputDevicesType & AUDIO_DEVICE_OUT_BLUETOOTH_A2DP;
            if (device2 == AUDIO_DEVICE_NONE) {
                device2 = availableOutputDevicesType & AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES;
            }
           //……
    ALOGVV("getDeviceForStrategy() strategy %d, device %x", strategy, device);
    return device;
}
```

我们就其中一个 strategty(STRATEGY_MEDIA) 来具体看看 Audio 系统的选择输出设备:

1）  首先我们会获取当前存在的设备集合 availableOutputDevices

2）  然后根据传入的 strategty 类型进行匹配选择

3）  在选择之前会先检测是否处于特殊情况下（如通话中）

4）  最后按照优先级匹配设备。

然后就这样，选择设备的流程就此结束。简单来说，选择设备的流程，主要是几个参数一步一步去确定然后最后确定合适的设备。具体选择设备的简单流程如图：

![](https://img-blog.csdn.net/20170205221531405?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQ0MDQwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

针对音频通路的切换，我们就简单聊到这里。谢谢。