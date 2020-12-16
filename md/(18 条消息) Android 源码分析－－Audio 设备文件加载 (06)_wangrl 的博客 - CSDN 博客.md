> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/WAN_EXE/article/details/79164135)

        手机上有多少设备都是通过配置 xml 文件来实现的，然后在构造 AudioPolicyManager 实例时进行文件解析，初始化相关的变量。比如有些手机有两个麦克风，通常叫做内置麦克风，和副麦克风，但是有些低端手机只会配置内置的麦克风。

        通过全局搜索可以看到

/device/google/dragon/device.mk:USE_XML_AUDIO_POLICY_CONF := 1

在源码的 device.mk 中有 USE_XML_AUDIO_POLICY_CONF 的定义，device.mk 应该是手机上所有硬件的配置目录，芯片开发商可以根据自己产品的需要另外配置相关的产品。

以下 / device/google/dragon / 项目为基础进行分析，这样不会涉及到公司的代码，这个目录应该是标准的配置目录，其它芯片厂商可以修改配置文件进行自己项目的配置。需要掌握 Android 的编译原理，后面会有专门的章节进行介绍。

        目前只重点关注这两句话  

```
PRODUCT_COPY_FILES += \
    frameworks/av/services/audiopolicy/config/default_volume_tables.xml:system/etc/default_volume_tables.xml \
    $(LOCAL_PATH)/audio_policy_configuration.xml:system/etc/audio_policy_configuration.xml
```

```
#define AUDIO_POLICY_XML_CONFIG_FILE_PATH_MAX_LENGTH 128
#define AUDIO_POLICY_XML_CONFIG_FILE_NAME "audio_policy_configuration.xml"
```

上面这个 Android.mk 编译语句表示返回当前被调用的文件路径。PRODUCT_COPY_FILES 的意思是将前面的文件复制到后面的文件中，最后编译打包 / system/etc/default_volume_tables.xml 和 / system/etc/audio_policy_configuration.xml 会放到手机的 / system/etc / 目录中，可以通过 adb shell 进行查看。

这两个文件在 AudioPolicyManager 中已经指定文件目录

```
mVolumeCurves = new VolumeCurvesCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled,
                             static_cast<VolumeCurvesCollection *>(mVolumeCurves));
    if (deserializeAudioPolicyXmlConfig(config) != NO_ERROR) {
        ALOGE("could not load audio policy configuration file, setting defaults");
        config.setDefault();
    }
```

接下来通过这两个文件来分析解析过程，配置不同可能解析的文件的路径不相同，文件的内容也会不相同，但是原理是相同的。

查看 if 分支

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!-- Copyright (C) 2016 The Android Open Source Project

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
-->
 
<audioPolicyConfiguration version="1.0" xmlns:xi="http://www.w3.org/2001/XInclude">
    <globalConfiguration speaker_drc_enabled="true"/>
 
    <modules>
        <module >
            <attachedDevices>
                <item>Speaker</item>
                <item>Built-In Mic</item>
            </attachedDevices>
            <defaultOutputDevice>Speaker</defaultOutputDevice>
            <mixPorts>
                <mixPort >
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort >
                    <profile 
                             samplingRates="8000,11025,12000,16000,22050,24000,32000,44100,48000"
                             channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
 
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
                <devicePort tag>
                </devicePort>
            </devicePorts>
            <routes>
                <route type="mix" sink="Speaker"
                       sources="primary output"/>
                <route type="mix" sink="Wired Headset"
                       sources="primary output"/>
                <route type="mix" sink="Wired Headphones"
                       sources="primary output"/>
                <route type="mix" sink="BT SCO"
                       sources="primary output"/>
                <route type="mix" sink="BT SCO Headset"
                       sources="primary output"/>
                <route type="mix" sink="BT SCO Car Kit"
                       sources="primary output"/>
                <route type="mix" sink="HDMI Out"
                       sources="primary output"/>
 
                <route type="mix" sink="primary input"
                       sources="Built-In Mic,Wired Headset Mic,BT SCO Headset Mic"/>
            </routes>
        </module>
 
        <!-- A2dp Audio HAL -->
        <xi:include href="a2dp_audio_policy_configuration.xml"/>
 
        <!-- Usb Audio HAL -->
        <xi:include href="usb_audio_policy_configuration.xml"/>
 
        <!-- Remote Submix Audio HAL -->
        <xi:include href="r_submix_audio_policy_configuration.xml"/>
 
    </modules>
 
    <!-- Volume section -->
    <xi:include href="audio_policy_volumes_drc.xml"/>
    <xi:include href="default_volume_tables.xml"/>
 
</audioPolicyConfiguration>
```

```
void setDefault(void)
    {
        mDefaultOutputDevices = new DeviceDescriptor(AUDIO_DEVICE_OUT_SPEAKER);
        sp<HwModule> module;
        sp<DeviceDescriptor> defaultInputDevice = new DeviceDescriptor(AUDIO_DEVICE_IN_BUILTIN_MIC);
        mAvailableOutputDevices.add(mDefaultOutputDevices);
        mAvailableInputDevices.add(defaultInputDevice);
 
        module = new HwModule("primary");
 
        sp<OutputProfile> outProfile;
        outProfile = new OutputProfile(String8("primary"));
        outProfile->attach(module);
        outProfile->addAudioProfile(
                new AudioProfile(AUDIO_FORMAT_PCM_16_BIT, AUDIO_CHANNEL_OUT_STEREO, 44100));
        outProfile->addSupportedDevice(mDefaultOutputDevices);
        outProfile->setFlags(AUDIO_OUTPUT_FLAG_PRIMARY);
        module->mOutputProfiles.add(outProfile);
 
        sp<InputProfile> inProfile;
        inProfile = new InputProfile(String8("primary"));
        inProfile->attach(module);
        inProfile->addAudioProfile(
                new AudioProfile(AUDIO_FORMAT_PCM_16_BIT, AUDIO_CHANNEL_IN_MONO, 8000));
        inProfile->addSupportedDevice(defaultInputDevice);
        module->mInputProfiles.add(inProfile);
 
        mHwModules.add(module);
    }
```

有对 HwModules 进行配置，有输入输出以及默认设备的配置，和音量曲线分布的配置。定义一个 config 变量，然后调用 deserializeAudioPolicyXmlConfig 函数进行初始化。

/[device](http://androidxref.com/8.0.0_r4/xref/device/)/[google](http://androidxref.com/8.0.0_r4/xref/device/google/)/[dragon](http://androidxref.com/8.0.0_r4/xref/device/google/dragon/)/[audio_policy_configuration.xml](http://androidxref.com/8.0.0_r4/xref/device/google/dragon/audio_policy_configuration.xml)

```
// Treblized audio policy xml config will be located in /odm/etc or /vendor/etc.
static const char *kConfigLocationList[] =
        {"/odm/etc", "/vendor/etc", "/system/etc"};
static const int kConfigLocationListSize =
        (sizeof(kConfigLocationList) / sizeof(kConfigLocationList[0]));
```

        解析过程开始，通过解析这些参数来理解变量的含义，先通过默认参数设置认识它们。

```
#include <stdio.h>
 
       int printf(const char *format, ...);
       int fprintf(FILE *stream, const char *format, ...);
       int sprintf(char *str, const char *format, ...);
       int snprintf(char *str, size_t size, const char *format, ...);
```

可以看到默认的输出设备是外放，默认的输入设备是内置麦克风。模块名为 primary, 输出定义 16 位 pcm 数据，双声道，采样率 44100Hz，输入定义为 16 位 pcm 数据，单声道，采样率为 8000Hz。

不得不佩服 Android 代码写的好

```
// Treblized audio policy xml config will be located in /odm/etc or /vendor/etc.
static const char *kConfigLocationList[] =
        {"/odm/etc", "/vendor/etc", "/system/etc"};
static const int kConfigLocationListSize =
        (sizeof(kConfigLocationList) / sizeof(kConfigLocationList[0]));
```

获取这个 kConfigLocationListSize 的值不是直接赋值成 3，而是通过除法获取的，这样当开发商想要添加文件目录的时候，只需要修改上面 kConfigLocationList 变量就可以，这样可以减少代码的耦合性。会遍历手里里这三个目录，查找叫 AUDIO_POLICY_XML_CONFIG_FILE_NAME 这个名字的文件。

```
#include <stdio.h>
 
       int printf(const char *format, ...);
       int fprintf(FILE *stream, const char *format, ...);
       int sprintf(char *str, const char *format, ...);
       int snprintf(char *str, size_t size, const char *format, ...);
```

snprintf 函数的意思是将后面的参数按照 format 的形式组合称字符串放入到最大不大于 size 的字符串 str 中。

The  functions  snprintf() and vsnprintf() write at most size bytes (including the terminating null byte ('\0')) to str.

所以最终 audioPolicyXmlConfigFile 的值为 / system/etc/audio_policy_configuration.xml.

总结：

        到目前为止只是介绍配置文件在哪里，还没有真正的解析。

下篇：

        解析配置文件。

如果觉得这篇文章有用，可以扫免费红包支持。

![](https://img-blog.csdn.net/20180126130219431?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvV0FOX0VYRQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)