> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/fellow1988/p/10122034.html)

Audio policy basic:https://www.cnblogs.com/CoderTian/p/5705742.html

Set volume flow:https://blog.csdn.net/axlecho/article/details/78510496

https://blog.csdn.net/u014702999/article/details/52488803

audio policy config xml:https://blog.csdn.net/u012188065/article/details/84104275

AudioPolicy 在 Android 系统中主要负责 Audio"策略" 相关的问题。它和 AudioFlinger 一起组成了 Android Audio 系统的两个服务。一个负责管理 audio 的 “路由”，一个负责管理 audio“设备”。这两个服务都是在系统启动的过程中，通过 MediaServer 来加载的。

AudioPolicyService 在 Android Audio 系统中主要完成以下几个任务：

①管理输入输出设备，包括设备的连接、断开状态，设备的选择和切换等

②管理系统的音频策略，比如通话时播放音乐、或者播放音乐时来电话的一系列处理

③管理系统的音量

④上层的一些音频参数也可以通过 AudioPolicyService 设置到底层去

 Audio Policy 初始化：

AudioPolicyService 是继承自 BnAudioPolicyService 的，一步步往上推，最终发现它的祖先是 RefBase，根据强指针的特性，目标对象在第一次被引用时会调用 onFirstRef()。

在第一次被强引用时 AudioPolicyService 创建了 3 个 AudioCommandThread 和 AudioPolicyManager。

三个 AudioCommandThread 线程分别是 ApmTone、ApmAudio、ApmOutput

 ApmTone 用于播放 tone 音；

 ApmAudio 用于执行 audio 命令；

ApmOutput 用于执行输出命令；

![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181228195820691-1196400128.png)

1.loadconfig

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
void AudioPolicyManager::loadConfig() {
#ifdef USE_XML_AUDIO_POLICY_CONF
    if (deserializeAudioPolicyXmlConfig(getConfig()) != NO_ERROR) {
#else
    if ((ConfigParsingUtils::loadConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE, getConfig()) != NO_ERROR)
           && (ConfigParsingUtils::loadConfig(AUDIO_POLICY_CONFIG_FILE, getConfig()) != NO_ERROR)) {
#endif
        ALOGE("could not load audio policy configuration file, setting defaults");
        getConfig().setDefault();
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

1）如果没有 define USE_XML_AUDIO_POLICY_CONF, 加载 audio_policy.conf 配置文件 /system/etc/audio_policy.conf（对于模拟器而言走的是 defaultAudioPolicyConfig()）

  在 AudioPolicyManager 创建过程中会通过加载 audio_policy.conf 配置文件来加载音频设备, Android 为每种音频接口定义了对应的硬件抽象层。硬件抽象层代码参考写法

hardware/libhardware/modules/audio

| 

external/bluetooth/bluedroid/audio_a2dp_hw/

 | 

audio.a2dp.default.so

 |
| 

hardware/libhardware/modules/audio/

 | 

audio.primary.default.so

 |
| 

hardware/libhardware/modules/usbaudio/

 | 

audio.usb.default.so

 |

  每种音频接口定义了不同的输入输出，一个接口可以具有多个输入或者输出，每个输入输出可以支持不同的设备，通过读取 audio_policy.conf 文件可以获取系统支持的音频接口参数，在 AudioPolicyManager 中会优先加载 / vendor/etc/audio_policy.conf 配置文件, 如果该配置文件不存在, 则加载 / system/etc/audio_policy.conf 配置文件。AudioPolicyManager 加载完所有音频接口后, 就知道了系统支持的所有音频接口参数, 可以为音频输出提供决策。

  audio_policy.conf 同时定义了多个 audio 接口, 每一个 audio 接口包含若干 output 和 input, 而每个 output 和 input 又同时支持多种输入输出模式, 每种输入输出模式又支持若干种设备.

![](https://images2015.cnblogs.com/blog/708332/201607/708332-20160725230446622-1465025873.jpg)

```
ConfigParsingUtils::loadAudioPolicyConfig(....);
```

分成两部分，第一部分是解析全局标签，第二部分是解析 audio_hw_modules 标签，其子标签都表示 hardware module，有 primary 和 r_submix 两种 hardware module 都被解析到 mHwModules，hardware module 的子标签有 outputs 和 inputs,outputs 的各个子标签被解析到 mHwModules 的 mOutputProfiles，inputs 的各个子标签被解析到 mHwModules 的 mInputProfiles。

2) 如果有 define USE_XML_AUDIO_POLICY_CONF, 则解析 audio_policy_configure.xml

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
<!-- Modules section:
        There is one section per audio HW module present on the platform.
        Each module section will contains two mandatory tags for audio HAL “halVersion” and “name”.
        The module names are the same as in current .conf file:
                “primary”, “A2DP”, “remote_submix”, “USB”
        Each module will contain the following sections:
        “devicePorts”: a list of device descriptors for all input and output devices accessible via this
        module.
        This contains both permanently attached devices and removable devices.
        “mixPorts”: listing all output and input streams exposed by the audio HAL
        “routes”: list of possible connections between input and output devices or between stream and
        devices.
            "route": is defined by an attribute:
                -"type": <mux|mix> means all sources are mutual exclusive (mux) or can be mixed (mix)
                -"sink": the sink involved in this route
                -"sources": all the sources than can be connected to the sink via vis route
        “attachedDevices”: permanently attached devices.
        The attachedDevices section is a list of devices names. The names correspond to device names
        defined in <devicePorts> section.
        “defaultOutputDevice”: device to be used by default when no policy rule applies
    -->
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
<modules>
        <!-- Primary Audio HAL -->
        <module >
            <attachedDevices>
                <item>Speaker</item>
                <item>Built-In Mic</item>
                <item>Built-In Back Mic</item>
            </attachedDevices>
            <defaultOutputDevice>Speaker</defaultOutputDevice>
            <mixPorts>
                <mixPort >
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort 
                        flags="AUDIO_OUTPUT_FLAG_DEEP_BUFFER">
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort 
                         flags="AUDIO_OUTPUT_FLAG_DIRECT|AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD|AUDIO_OUTPUT_FLAG_NON_BLOCKING">
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO,AUDIO_CHANNEL_OUT_MONO"/>
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO,AUDIO_CHANNEL_OUT_MONO"/>
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_OUT_STEREO,AUDIO_CHANNEL_OUT_MONO"/>
                </mixPort>
                <mixPort >
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
                </mixPort>
                <mixPort >
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </mixPort>
                <mixPort >
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <!-- Output devices declaration, i.e. Sink DEVICE PORT -->
                <devicePort tag>
                   <profile 
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain 
                              minValueMB="-8400"
                              maxValueMB="4000"
                              defaultValueMB="0"
                              stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_OUT_MONO"/>
                </devicePort>

                <devicePort tag>
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO,AUDIO_CHANNEL_IN_FRONT_BACK"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000" channelMasks="AUDIO_CHANNEL_IN_MONO"/>
                </devicePort>
            </devicePorts>
            <!-- route declaration, i.e. list all available sources for a given sink -->
            <routes>
                <route type="mix" sink="Earpiece"
                       sources="primary output,deep_buffer,BT SCO Headset Mic"/>
                <route type="mix" sink="Speaker"
                       sources="primary output,deep_buffer,compressed_offload,BT SCO Headset Mic,Telephony Rx"/>
                <route type="mix" sink="Wired Headset"
                       sources="primary output,deep_buffer,compressed_offload,BT SCO Headset Mic,Telephony Rx"/>
                <route type="mix" sink="Wired Headphones"
                       sources="primary output,deep_buffer,compressed_offload,BT SCO Headset Mic,Telephony Rx"/>
                <route type="mix" sink="primary input"
                       sources="Built-In Mic,Built-In Back Mic,Wired Headset Mic,BT SCO Headset Mic"/>
                <route type="mix" sink="Telephony Tx"
                       sources="Built-In Mic,Built-In Back Mic,Wired Headset Mic,BT SCO Headset Mic, voice_tx"/>
                <route type="mix" sink="voice_rx"
                       sources="Telephony Rx"/>
            </routes>

        </module>

        <!-- A2dp Audio HAL -->
        <xi:include href="a2dp_audio_policy_configuration.xml"/>

        <!-- Usb Audio HAL -->
        <xi:include href="usb_audio_policy_configuration.xml"/>

        <!-- Remote Submix Audio HAL -->
        <xi:include href="r_submix_audio_policy_configuration.xml"/>

        <!-- Hearing aid Audio HAL -->
        <xi:include href="hearing_aid_audio_policy_configuration.xml"/>

    </modules>
    <!-- End of Modules section -->

    <!-- Volume section -->

    <xi:include href="audio_policy_volumes.xml"/>
    <xi:include href="default_volume_tables.xml"/>

    <!-- End of Volume section -->

</audioPolicyConfiguration>
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

每个 <module> 都是一个 audio hal(对应代码中的 HwModule 类)。

<mixPort>(对应代码中 AudioPort 的子类 IOProfile)， 共有两大类 role， 一类是 "source"， 一类是 “sink”。  
"source" 类的 <mixPort> 共同组成了 HwModule 中的 mOutputProfiles， "sink" 类的 < mixPort> 共同组成了 HwModule 中的 mInputProfiles， mOutputProfiles + mInputProfiles 共同组成了 HwModule 中的 mPorts。  
<devicePort>(对应代码中 AudioPort 和 AudioPortConfig 的子类 DeviceDescriptor)，共同组成了 HwModule 中的 mDeclaredDevices， 里面的各个 device 成员也会加载到先前 HwModule 类中的 mPort。

<route>(对应代码中的 AudioRoute 类)，顾名思义，是通路的意思，其连接 port 和 device 的作用. 每个 < route > 实质上是一个 MIX 或者 MUX, 以 MIX 为例，一个 MIX 包含一个 sink(即 AudioRoute 的 mSink) 和多个 source(分别 add 到 AudioRoute 的 mSurce)，此时已经将 sink 和 source 建立了简单的关系。  
![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181228205848170-1523901873.png)

2)initialize

![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181228202920612-1460714709.png)

load_audio_interface() 用来 audio hal so，然后打开设备并创建一个 audio_hw_device_t 实例，音频设备接口所对应的 so 名称是有一定格式的，如 a2dp 的模块名可能是 audio.a2dp.so 或 audio.a2dp.default.so 等，查找路径主要有两个 / system/lib/hw 和 / vendor/lib/hw

load（）dlopen hal so,dlsym 返回 audio_module:

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .module_api_version = AUDIO_MODULE_API_VERSION_0_1,
        .hal_api_version = HARDWARE_HAL_API_VERSION,
        .id = AUDIO_HARDWARE_MODULE_ID,
        .name = "QCOM Audio HAL",
        .author = "Code Aurora Forum",
        .methods = &hal_module_methods,
    },
};
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

open 返回 audio_hw_device_t。这样 audio policy 完成了对 Hal module 的加载

![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181228211708594-204424545.png)

 对于每个 outputProfile，创建 1 个 outputDescription, 每个 outputDescription 调用 open 来 openOutput, 返回 audio_io_handle_t 类型的 output handle. 并把 output handle add 到 mOutput. 然后调用 setOutputDevice(在里面 createAudioPatch)。

![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181228224721659-1332609179.png)

 ![](https://img2018.cnblogs.com/blog/709240/201812/709240-20181229200727227-1813761011.png)

setDeviceConnectionState: 当有设备（比如耳机）接入或者断开时 ，会由 java 层通知 audio system 来更改输出设备。

 ![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101171038267-2043532025.png)

设置音量：

![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101135106185-1481073296.png)

getOutput flow:

![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101153723642-1333999954.png)

 ![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101150616248-2110775976.png)

setForceUse & create patch flow:

![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101182838476-667483280.png)

Audio policy 与 Audio Flinger control block:

![](https://img2018.cnblogs.com/blog/709240/201901/709240-20190101220941519-750915924.png)