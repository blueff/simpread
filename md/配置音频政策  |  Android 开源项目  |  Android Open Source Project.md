\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[source.android.google.cn\](https://source.android.google.cn/devices/audio/implement-policy#code\_reorg)

 .devsite-banner-announcement { background: #202124; color: white; font-weight: 500; } .devsite-banner-announcement a { color: white; font-weight: 700; } .devsite-banner-announcement a:link, .devsite-banner-announcement a:visited {background: 0;} .devsite-banner-announcement a:hover, .devsite-banner-announcement a:focus { text-decoration: none; } .devsite-banner-announcement .devsite-banner-message-text { margin: 0 auto; } 。

*   [AOSP](/)
*   [开发](/devices)
*   [音频](/devices/audio)

配置音频政策
======

*   [Table of contents](#top_of_page)
*   [XML 格式的优势](#xml_advantages)
*   [文件格式和位置](#xml_file_format)
*   [文件包含](#file_inclusions)
*   [音频政策代码组织](#code_reorg)
*   [使用参数框架的配置](#policy_config)
*   [音频政策导向 API](#policy_routing_apis)
*   [多声道支持](#multichannel)
*   [媒体编解码器](#codecs)

Android 10 版本对音频政策管理器进行了重大重构，有助于更灵活地支持复杂的车载使用情形：

*   提供了 OEM 专用路由策略。
*   针对使用相同音量曲线的旧版音频流类型组提供了可自定义音量组。
*   路由策略由音频政策引擎声明，而不是采用硬编码格式。
*   音量曲线和组由音频政策引擎管理。
*   进行了内部重构，目的是为将来分离公共代码和可配置代码做准备，并提供更丰富的音频设备管理功能。例如，在政策规则中使用所有设备属性，而不仅仅是其类型。

Android 7.0 引入了音频政策配置文件格式 (XML)，用于描述音频拓扑。

以前的 Android 版本需要使用 `device/<company>/<device>/audio/audio_policy.conf` 来声明您的产品上存在的音频设备（您可以在 `device/samsung/tuna/audio/audio_policy.conf` 中查看此文件针对 Galaxy Nexus 音频硬件的示例）。但是，CONF 是一种简单的专有格式，有较大的局限性，无法描述电视和汽车等行业的复杂拓扑。

Android 7.0 已弃用 `audio_policy.conf` 并添加了对使用 XML 文件格式定义音频拓扑的支持，该文件格式更易于阅读，可使用各种修改和解析工具，并且足够灵活，可以描述复杂的音频拓扑。

**注意**：Android 7.0 仍支持使用 `audio_policy.conf`；系统会默认使用这种旧版格式。要使用 XML 文件格式，需要在设备 Makefile 中添加编译选项 `USE_XML_AUDIO_POLICY_CONF := 1`。

XML 格式的优势
---------

与在 CONF 文件中一样，XML 文件支持定义输出输入流配置文件、可用于播放和捕获的设备以及音频属性的数量和类型。此外，XML 格式还提供以下增强功能：

*   在 Android 10 中，允许同时使用多个正在运行的录音应用。
    *   由于存在并发情况，开始录音的请求永远不会遭到拒绝。
    *   `registerAudioRecordingCallback(AudioManager.AudioRecordingCallback cb)` 回调会将捕获路径更改告知客户端。
*   在以下情况下，客户端会获取无声音频样本：
    *   处于对隐私敏感的使用情形（例如，`VOICE_COMMUNICATION`）下。
    *   客户端没有前台服务或前台界面。
    *   政策识别到了特殊的角色：
        *   无障碍服务：即使处于对隐私敏感的使用情形，也可以录音。
        *   Google 助理：如果相应界面位于顶层，则视为隐私敏感。
*   音频配置文件的结构类似于 HDMI 简单音频描述符，支持每种音频格式使用一组不同的采样率 / 声道掩码。
*   设备和流之间所有可能的连接都有显式定义。以前，隐式规则可以使连接到同一 HAL 模块的所有设备互连，从而阻止音频政策控制使用音频补丁程序 API 请求的连接。在 XML 格式中，拓扑描述定义了连接限制。
*   对 “包含” 的支持可避免出现重复的标准 A2DP、USB 或重新导向提交定义。
*   音量曲线可自定义。以前，音量表采用硬编码格式。在 XML 格式中，音量表通过描述来定义，并且可自定义。

`frameworks/av/services/audiopolicy/config/audio_policy_configuration.xml` 中的模板展示了很多已在使用的上述功能。

文件格式和位置
-------

新的音频政策配置文件是 `audio_policy_configuration.xml`，位于 `/system/etc`。以下示例展示了采用 XML 文件格式的简单音频政策配置。

#### 显示音频政策示例

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<audioPolicyConfiguration version="1.0" xmlns:xi="http://www.w3.org/2001/XInclude">
    <globalConfiguration speaker\_drc\_enabled="true"/>
    <modules>
        <module >
            <attachedDevices>
                <item>Speaker</item>
                <item>Earpiece</item>
                <item>Built-In Mic</item>
            </attachedDevices>
            <defaultOutputDevice>Speaker</defaultOutputDevice>
            <mixPorts>
                <mixPort >
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO\_CHANNEL\_OUT\_STEREO"/>
                </mixPort>
                <mixPort >
                    <profile 
                             samplingRates="8000,16000,48000"
                             channelMasks="AUDIO\_CHANNEL\_IN\_MONO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <devicePort tag>
                   <profile 
                            samplingRates="48000" channelMasks="AUDIO\_CHANNEL\_IN\_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO\_CHANNEL\_OUT\_STEREO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="48000" channelMasks="AUDIO\_CHANNEL\_OUT\_STEREO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000,48000"
                             channelMasks="AUDIO\_CHANNEL\_IN\_MONO"/>
                </devicePort>
                <devicePort tag>
                    <profile 
                             samplingRates="8000,16000,48000"
                             channelMasks="AUDIO\_CHANNEL\_IN\_MONO"/>
                </devicePort>
            </devicePorts>
            <routes>
                <route type="mix" sink="Earpiece" sources="primary output"/>
                <route type="mix" sink="Speaker" sources="primary output"/>
                <route type="mix" sink="Wired Headset" sources="primary output"/>
                <route type="mix" sink="primary input" sources="Built-In Mic,Wired Headset Mic"/>
            </routes>
        </module>
        <xi:include href="a2dp\_audio\_policy\_configuration.xml"/>
    </modules>

    <xi:include href="audio\_policy\_volumes.xml"/>
    <xi:include href="default\_volume\_tables.xml"/>
</audioPolicyConfiguration>


``` 

顶层结构中包含与各个音频 HAL 硬件模块对应的模块，其中每个模块都有一系列混音端口、设备端口和导向：

*   **混音端口**描述了可以在音频 HAL 处打开以供播放和捕获的流的可能配置文件。
*   **设备端口**描述了可以附上其类型（以及（可选）地址和音频属性，如果相关）的设备。
*   **导向**已从混音端口描述符中分离出来，支持描述从设备到设备或从流到设备的导向。

音量表是定义用于将界面索引转换为音量（以 dB 为单位）的曲线的点的简单列表。单独的包含文件提供默认曲线，但每个对应于指定使用情形和设备类别的曲线都可以被替换。

#### 显示音量表示例

```
<?xml version="1.0" encoding="UTF-8"?>
<volumes>
    <reference >
        <point>0,0</point>
        <point>100,0</point>
    </reference>
    <reference >
        <point>0,-9600</point>
        <point>100,-9600</point>
    </reference>
    <reference >
        <point>1,-4950</point>
        <point>33,-3350</point>
        <point>66,-1700</point>
        <point>100,0</point>
    </reference>
</volumes>


``` #### 显示音量示例

```
<?xml version="1.0" encoding="UTF-8"?>
<volumes>
    <volume stream="AUDIO\_STREAM\_VOICE\_CALL" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_VOICE\_CALL" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_VOICE\_CALL" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_VOICE\_CALL" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_SYSTEM" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_SYSTEM" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_SYSTEM" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_SYSTEM" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_RING" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_RING" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_RING" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_RING" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA"ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_MUSIC" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_MUSIC" deviceCategory="DEVICE\_CATEGORY\_SPEAKER">
        <point>1,-5500</point>
        <point>20,-4300</point>
        <point>86,-1200</point>
        <point>100,0</point>
    </volume>
    <volume stream="AUDIO\_STREAM\_MUSIC" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_MUSIC" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_ALARM" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ALARM" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ALARM" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ALARM" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_NOTIFICATION" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_NOTIFICATION" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_NOTIFICATION" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_NOTIFICATION" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_BLUETOOTH\_SCO" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_BLUETOOTH\_SCO" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_BLUETOOTH\_SCO" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_BLUETOOTH\_SCO" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_ENFORCED\_AUDIBLE" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ENFORCED\_AUDIBLE" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ENFORCED\_AUDIBLE" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ENFORCED\_AUDIBLE" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_DTMF" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_DTMF" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_DTMF" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_DTMF" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_TTS" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="SILENT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_TTS" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_TTS" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="SILENT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_TTS" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="SILENT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_ACCESSIBILITY" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ACCESSIBILITY" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ACCESSIBILITY" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="DEFAULT\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_ACCESSIBILITY" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="DEFAULT\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_REROUTING" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_REROUTING" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_REROUTING" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_REROUTING" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="FULL\_SCALE\_VOLUME\_CURVE"/>

    <volume stream="AUDIO\_STREAM\_PATCH" deviceCategory="DEVICE\_CATEGORY\_HEADSET" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_PATCH" deviceCategory="DEVICE\_CATEGORY\_SPEAKER" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_PATCH" deviceCategory="DEVICE\_CATEGORY\_EARPIECE" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
    <volume stream="AUDIO\_STREAM\_PATCH" deviceCategory="DEVICE\_CATEGORY\_EXT\_MEDIA" ref="FULL\_SCALE\_VOLUME\_CURVE"/>
</volumes>


``` 

文件包含
----

XML 包含 (XInclude) 方法可用于包含位于其他 XML 文件中的音频政策配置信息。所有包含的文件必须遵循上述结构，同时满足以下限制条件：

*   文件只能包含顶层元素。
*   文件不能包含 XInclude 元素。

使用 “包含” 可避免将标准 Android 开源项目 (AOSP) 音频 HAL 模块配置信息复制到所有音频政策配置文件（这样做容易出错）。我们为以下音频 HAL 提供了标准音频政策配置 XML 文件：

*   **A2DP**：`a2dp_audio_policy_configuration.xml`
*   **重新导向子混音**：`rsubmix_audio_policy_configuration.xml`
*   **USB：**`usb_audio_policy_configuration.xml`

音频政策代码组织
--------

平台将 `AudioPolicyManager.cpp` 拆分为多个模块，使其易于维护和配置。`frameworks/av/services/audiopolicy` 的组织包括以下模块：

<table><tbody><tr><th>模块</th><th>说明</th></tr><tr><td><code>/<wbr>managerdefault</code></td><td>包含所有应用通用的常规接口和行为实现。类似于剥离了引擎功能和通用概念的 <code>AudioPolicyManager.<wbr>cpp</code>。</td></tr><tr><td><code>/<wbr>common</code></td><td>定义基类（例如，输入输出音频流配置文件、音频设备描述符、音频补丁程序和音频端口等的数据结构）。该模块之前在 <code>AudioPolicyManager.<wbr>cpp</code> 中定义。</td></tr><tr><td><code>/<wbr>engine</code></td><td><p>实现规则，这些规则定义应将哪些设备和音量用于指定使用情形。该模块会实现标准接口（包含通用部分），例如，为指定的播放或捕获使用情形获取适当的设备，或设置可以改变导向选择的已连接设备或外部状态（即强制使用的调用状态）。</p><p>两种版本（自定义版本和默认版本）中都提供该模块；使用编译选项 <code>USE_CONFIGURABLE_AUDIO_POLICY</code> 进行选择。</p></td></tr><tr><td><code>/<wbr>engineconfigurable</code></td><td>依赖参数框架的政策引擎实现（请参阅下文）。配置基于参数框架，相关政策由 XML 文件定义。</td></tr><tr><td><code>/<wbr>enginedefault</code></td><td>基于以前的 Android 音频政策管理器实现的政策引擎实现。这是默认模块，包含 Nexus 和 AOSP 实现对应的硬编码规则。</td></tr><tr><td><code>/<wbr>service</code></td><td>包含 Binder 接口、线程和锁定实现（包含连接框架其余部分的接口）。</td></tr></tbody></table>

使用参数框架的配置
---------

平台对音频策略代码进行了组织，使其易于理解和维护，同时也支持完全由配置文件定义的音频政策。组织和音频政策设计基于 Intel 的参数框架，该框架基于插件和规则，用于处理各种参数。

通过使用可配置音频政策，供应商和原始设备制造商 (OEM) 可以：

*   用 XML 来描述系统的结构及其参数。
*   编写（用 C++ 语言）或重复使用后端（插件）来访问所描述的参数。
*   定义（用 XML 或特定于网域的语言）指定参数必须取指定值所依据的条件 / 规则。

AOSP 中包含使用参数框架的音频政策配置文件示例，路径为：`Frameworks/av/services/audiopolicy/engineconfigurable/parameter-framework/example/Settings/PolicyConfigurableDomains.xml`。如需了解详情，请参阅介绍[参数框架](https://01.org/parameter-framework/)的 Intel 文档。

音频政策导向 API
----------

Android 6.0 引入了一个公共 Enumeration and Selection API，位于音频补丁程序 / 音频端口基础架构之上。借助它，应用开发者可以为连接的音频录制或轨道指明特定设备输出或输入的偏好设置。

在 Android 7.0 中，Enumeration and Selection API 通过了 CTS 测试的验证，并经过扩展，可导向原生 C/C++ (OpenSL ES) 音频流。原生流的导向仍然由 Java 完成，另外还添加了 `AudioRouting` 接口，该接口会取代、合并和弃用特定于 `AudioTrack` 和 `AudioRecord` 类的显式导向方法。

如需详细了解 Enumeration and Selection API，请参阅 [Android 配置接口](https://developer.android.google.cn/ndk/guides/audio/opensl-for-android.html?hl=fi#configuration-interface)和 `OpenSLES_AndroidConfiguration.h`。要详细了解音频导向，请参阅 [AudioRouting](https://developer.android.google.cn/reference/android/media/AudioRouting.html)。

多声道支持
-----

如果您的硬件和驱动程序通过 HDMI 支持多声道音频，则可以将音频流直接输出到音频硬件（这样可以绕过 AudioFlinger 混音器，也就不会被降混成两个声道）。音频 HAL 必须展示输出流配置文件是否支持多声道音频功能。如果 HAL 展示其功能，则默认的政策管理器会允许通过 HDMI 进行多声道播放。有关实现的详情，请参阅 `device/samsung/tuna/audio/audio_hw.c`。

要指定您的产品包含多声道音频输出，请修改音频政策配置文件，对产品的多声道输出进行描述。以下示例来自 Galaxy Nexus，其展示了动态声道掩码，这表示音频政策管理器会在连接后查询 HDMI 接收器支持的声道掩码。

```
audio\_hw\_modules {
  primary {
    outputs {
        ...
        hdmi {
          sampling\_rates 44100|48000
          channel\_masks dynamic
          formats AUDIO\_FORMAT\_PCM\_16\_BIT
          devices AUDIO\_DEVICE\_OUT\_AUX\_DIGITAL
          flags AUDIO\_OUTPUT\_FLAG\_DIRECT
        }
        ...
    }
    ...
  }
  ...
}


``` 

您也可以指定静态声道掩码，例如 `AUDIO_CHANNEL_OUT_5POINT1`。当将内容发送到不支持多声道音频的音频设备时，AudioFlinger 的混音器会自动将该内容降混成立体声。

媒体编解码器
------

确保针对您的产品正确声明您的硬件和驱动程序支持的音频编解码器。如需了解详情，请参阅[将编解码器展示给框架](/devices/media#expose)。

此页内容对您有帮助吗？

此页内容对您有帮助吗？

此页内容对您有帮助吗？

此页内容对您有帮助吗？