\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[source.android.google.cn\](https://source.android.google.cn/devices/audio/concurrent)

 .devsite-banner-announcement { background: #202124; color: white; font-weight: 500; } .devsite-banner-announcement a { color: white; font-weight: 700; } .devsite-banner-announcement a:link, .devsite-banner-announcement a:visited {background: 0;} .devsite-banner-announcement a:hover, .devsite-banner-announcement a:focus { text-decoration: none; } .devsite-banner-announcement .devsite-banner-message-text { margin: 0 auto; } 。

*   [AOSP](/)
*   [开发](/devices)
*   [音频](/devices/audio)

并发捕获
====

*   [Table of contents](#top_of_page)
*   [CDD 要求](#cdd-requirements)
*   [从音频 HAL 进行捕获的情况](#capture-situations-from-audio-hal)
    *   [AP 输入流的并发 Activity](#concurrent-activity-of-ap-input-streams)
    *   [在进行语音通话的同时从 AP 进行捕获](#concurrent-voice-call-and-capture-from-ap)
    *   [从 DSP 和 AP 进行并发捕获](#concurrent-capture-from-dsp-and-ap)
*   [对智能助理实现的影响](#implication-for-assistant-implementations)
    *   [有关数据使用和用户通知的要求](#requirements)
    *   [Android 10 的功能改进](#functional-improvements-for-android-10)
*   [音频 HAL 实现示例](#sample-audio-hal-implementation)

Android 10 改进了需要同时进行多个活动音频捕获的用户体验，例如，当用户想要使用无障碍服务提供的语音指令来控制 VoIP 通话或录像机时。

该音频框架实现了仅允许某些特权应用与常规应用同时进行音频捕获的政策。

该并发政策通过将应用捕获的音频设为静音（而不是阻止应用开始捕获）来实现。这样一来，在应用在其他应用完成捕获后才能恢复对麦克风的完全访问权限的情况下，框架便可以动态处理活动捕获用例的数量和类型方面的变化，而不会阻止应用开始捕获音频。

这对音频 HAL 和音频子系统造成的结果是：它们必须同时支持多个活动输入流，即使在某些情况下，只有一个输入流向活动客户端提供非静默音频。

CDD 要求
------

如需了解有关支持并发捕获的要求，请参阅 [CDD](/compatibility/android-cdd)。

从音频 HAL 进行捕获的情况
---------------

根据活动输入流的数量、输入设备的选择或预处理配置的不同，一种并发捕获情景可以导致多种不同的情况。

并发可以在下列对象之间发生：

*   来自应用处理器 (AP) 的多个输入流之间
*   输入流和语音通话之间
*   输入流和实现低功耗启动指令检测的音频 DSP 之间

### AP 输入流的并发 Activity

音频框架使用音频政策配置文件 `audio_policy_configuration.xml` 来确定可同时处于打开和活动状态的输入流的数量。

对于**处于打开和活动状态的配置文件**中列出的**每个输入配置文件（角色为 `sink` 的 `mixPort`），音频 HAL 必须至少支持它的一个实例**。

#### 设备选择

如果多个活动客户端连接到同一 HAL 输入流，则框架会根据用例优先级为此输入流选择相应的设备。

如果多个输入流同时处于活动状态，则每个输入流可以选择不同的设备。

如果技术兼容，建议音频 HAL 和子系统允许不同的输入流从不同的设备（比如蓝牙耳机和内置麦克风）捕获音频。

如果存在不兼容的情况（例如，两个设备共享同一个数字音频接口或后端），则音频 HAL 必须选择由哪个输入流控制设备选择。

在本例中：

*   当重复出现相同的情景时，所呈现的状态必须一致，并且必须提供相同的设备选择。
*   当并发状态结束时，必须将剩下的活动输入流转至该输入流上最初请求的设备。

如果音频 HAL 对有效用例定义了优先顺序，请遵循 `frameworks/av/services/audiopolicy/common/include/policy.h` 中的 `source_priority()` 指定的顺序

#### 预处理选择

音频框架可以请求使用 `addEffect()` 或 `removeEffect()` 方法对输入流进行预处理。

对指定输入流进行预处理时，音频框架仅启用该输入流上优先级最高的活动用例所对应的配置。 不过，用例激活和用例停用在时间上可能会存在一些重叠，从而导致在同一个输入流上同时运行两个活动的进程（例如，回声消除器的两个实例）。在这种情况下，HAL 实现会选择接受哪个请求；该实现会跟踪活动请求，并在其中任何一个进程被停用时恢复到正确的状态。

如果有多个捕获流同时处于活动状态，则不同的流上可能会运行不同的预处理请求。

HAL 和音频子系统实现应允许对不同的流应用不同的预处理，即使它们共享同一个输入设备也是如此。也就是说，应该在对来自主要捕获来源的流进行解复用之后再应用预处理。

如果由于技术原因无法在指定的音频子系统上实现这一点，音频 HAL 应该应用与[设备选择](#device-selection)中所列的类似的优先级确定规则。

### 在进行语音通话的同时从 AP 进行捕获

当语音通话处于活动状态时，可以从 AP 进行捕获。这种情况不是在 Android 10 中才出现的，也和并发捕获功能没有直接关系，但是有必要提及适用于该情景的准则。

在通话过程中，需要从 AP 进行两种不同类型的捕获。

*   [捕获通话 RX 和 TX 路径](#capturing-call-rx-and-tx)
*   [从输入设备（例如内置麦克风）捕获](#capturing-from-input-devices-when-a-call-is-active)

#### 捕获通话 RX 和 TX

捕获通话 RX 和 TX 的操作，可通过使用音频来源（`AudioSource.VOICE_UPLINK` 或 `AudioSource.VOICE_DOWNLINK`）和 / 或设备 `AudioDevice.IN_TELEPHONY_RX` 进行触发。

音频 HAL 应该公开一个输入配置文件（角色为 `sink` 的 `mixPort`），并提供设备 `AudioDevice.IN_TELEPHONY_RX` 的可用路径。

电话接通（音频模式为 `AudioMode.IN_CALL`）后，应该至少有一个来自设备 `AudioDevice.IN_TELEPHONY_RX` 的有效捕获流。

#### 当通话处于活动状态时从输入设备进行捕获

当通话处于活动状态（音频模式为 `AudioMode.IN_CALL`）时，应该可以按照 [AP 输入流的并发 Activity](#concurrent-activity-of-ap-input-streams) 部分中所述的方式打开和激活来自 AP 的输入流。

不过，设备选择和预处理的优先级应始终由语音通话来确定，以防与来自 AP 输入流的请求发生冲突。

### 从 DSP 和 AP 进行并发捕获

当音频子系统包含支持低功耗音频上下文或启动指令检测功能的 DSP 时，该实现应支持从 AP 和音频 DSP 进行并发捕获。这包括在初始检测阶段由 DSP 执行的捕获以及在 DSP 触发检测后由 AP 使用 `AudioSource.HOTWORD` 执行的捕获。

这应该通过并发捕获标记反映出来，该标记由声音触发器 HAL 通过以下实现描述符报告：`ISoundTriggerHw.Properties.concurrentCapture = true`。

音频 HAL 还应该公开特定于启动指令捕获（由 `AudioInputFlag.HW_HOTWORD` 标记标识）的输入配置文件。该实现应支持打开和激活此配置文件上的多个流，且打开和激活的流的数量应至少等于声音触发器 HAL 可以并发加载的音型的数量。

当其他输入配置文件处于活动状态时，应该可以从此输入配置文件进行捕获。

对智能助理实现的影响
----------

### 有关数据使用和用户通知的要求

由于并发使用麦克风如果被滥用，可能会泄露用户的隐私数据，因此我们需要对请求担任智能助理角色的预加载特权应用实施以下条件和保证。

*   除非用户在与智能助理进行互动，否则通过麦克风收集的数据不得离开设备。例如，在触发启动指令之后。
*   并发监听的应用应该在检测到启动指令后向用户提供视觉提示。这有助于用户知悉后续会话将通过其他应用（比如智能助理）进行。
*   用户应能够关闭麦克风或智能助理触发器。
*   存储录音后，用户应能够随时访问、查看和删除录音。

### Android 10 的功能改进

#### 智能助理不会相互阻塞

在 Android P 或更低版本中，当设备上存在两个始终保持开启状态的智能助理时，只有一个可以监听其启动指令。因此需要在两个智能助理之间进行切换。在 Android 10 中，默认的智能助理可以与另一个助理并发监听启动指令。这可以为使用两个智能助理的用户提供更加流畅的体验。

#### 使麦克风保持开启状态的应用

当 Shazam 或 Waze 之类的应用使麦克风保持开启状态时，默认的智能助理仍可监听启动指令。

对于非默认智能助理应用，在 Android 10 中的行为没有变化。

音频 HAL 实现示例
-----------

可在 [AOSP](https://android.googlesource.com/platform/hardware/qcom/audio/+/refs/heads/master) 中找到符合本文档中的准则的音频 HAL 实现示例。

此页内容对您有帮助吗？

此页内容对您有帮助吗？

此页内容对您有帮助吗？