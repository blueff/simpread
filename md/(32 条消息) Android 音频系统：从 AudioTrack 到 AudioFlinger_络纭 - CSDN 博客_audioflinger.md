> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/zyuanyun/article/details/60890534)

1. Android 音频框架概述
=================

![](https://img-blog.csdn.net/20170301225246685?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Audio 是整个 Android 平台非常重要的一个组成部分，负责音频数据的采集和输出、音频流的控制、音频设备的管理、音量调节等，主要包括如下部分：

*   **Audio Application Framework**：音频应用框架  
    *   AudioTrack：负责回放数据的输出，属 Android 应用框架 API 类
    *   AudioRecord：负责录音数据的采集，属 Android 应用框架 API 类
    *   AudioSystem： 负责音频事务的综合管理，属 Android 应用框架 API 类
*   **Audio Native Framework**：音频本地框架  
    *   AudioTrack：负责回放数据的输出，属 Android 本地框架 API 类
    *   AudioRecord：负责录音数据的采集，属 Android 本地框架 API 类
    *   AudioSystem： 负责音频事务的综合管理，属 Android 本地框架 API 类
*   **Audio Services**：音频服务  
    *   AudioPolicyService：音频策略的制定者，负责音频设备切换的策略抉择、音量调节策略等
    *   AudioFlinger：音频策略的执行者，负责输入输出流设备的管理及音频流数据的处理传输
*   **Audio HAL**：音频硬件抽象层，负责与音频硬件设备的交互，由 AudioFlinger 直接调用

与 Audio 强相关的有 MultiMedia，MultiMedia 负责音视频的编解码，MultiMedia 将解码后的数据通过 AudioTrack 输出，而 AudioRecord 采集的录音数据交由 MultiMedia 进行编码。

本文分析基于 **Android 7.0 - Nougat**。

//  
// 声明：本文由 [http://blog.csdn.net/zyuanyun](http://blog.csdn.net/zyuanyun) 原创，转载请注明出处，谢谢！  
//

2. AudioTrack API 概述
====================

播放声音可以使用 MediaPlayer 和 AudioTrack，两者都提供 Java API 给应用开发者使用。两者的差别在于：MediaPlayer 可以播放多种格式的音源，如 mp3、flac、wma、ogg、wav 等，而 AudioTrack 只能播放解码后的 PCM 数据流。从上面 Android 音频系统架构图来看：MediaPlayer 在 Native 层会创建对应的音频解码器和一个 AudioTrack，解码后的数据交由 AudioTrack 输出。所以 MediaPlayer 的应用场景更广，一般情况下使用它也更方便；只有一些对声音时延要求非常苛刻的应用场景才需要用到 AudioTrack。

2.1. AudioTrack Java API
------------------------

AudioTrack Java API 两种数据传输模式：

<table><thead><tr><th>Transfer Mode</th><th>Description</th></tr></thead><tbody><tr><td>MODE_STATIC</td><td>应用进程将回放数据一次性付给 AudioTrack，适用于数据量小、时延要求高的场景</td></tr><tr><td>MODE_STREAM</td><td>用进程需要持续调用 write() 写数据到 FIFO，写数据时有可能遭遇阻塞（等待 AudioFlinger::PlaybackThread 消费之前的数据），基本适用所有的音频场景</td></tr></tbody></table>

AudioTrack Java API 音频流类型：

<table><thead><tr><th>Stream Type</th><th>Description</th></tr></thead><tbody><tr><td>STREAM_VOICE_CALL</td><td>电话语音</td></tr><tr><td>STREAM_SYSTEM</td><td>系统声音</td></tr><tr><td>STREAM_RING</td><td>铃声声音，如来电铃声、闹钟铃声等</td></tr><tr><td>STREAM_MUSIC</td><td>音乐声音</td></tr><tr><td>STREAM_ALARM</td><td>警告音</td></tr><tr><td>STREAM_NOTIFICATION</td><td>通知音</td></tr><tr><td>STREAM_DTMF</td><td>DTMF 音（拨号盘按键音）</td></tr></tbody></table>

Android 为什么要定义这么多的流类型？这与 Android 的音频管理策略有关，例如：

*   音频流的音量管理，调节一个类型的音频流音量，不会影响到其他类型的音频流
*   根据流类型选择合适的输出设备；比如插着有线耳机期间，音乐声（STREAM_MUSIC）只会输出到有线耳机，而铃声（STREAM_RING）会同时输出到有线耳机和外放

这些属于 AudioPolicyService 的内容，本文不展开分析了。应用开发者应该根据应用场景选择相应的流类型，以便系统为这道流选择合适的输出设备。

一个 AudioTrack Java API 的测试例子（MODE_STREAM 模式）：

```
//Test case 1: setStereoVolume() with max volume returns SUCCESS
    @LargeTest
    public void testSetStereoVolumeMax() throws Exception {
        // constants for test
        final String TEST_NAME = "testSetStereoVolumeMax";
        final int TEST_SR = 22050;
        final int TEST_CONF = AudioFormat.CHANNEL_OUT_STEREO;
        final int TEST_FORMAT = AudioFormat.ENCODING_PCM_16BIT;
        final int TEST_MODE = AudioTrack.MODE_STREAM;
        final int TEST_STREAM_TYPE = AudioManager.STREAM_MUSIC;

        //-------- initialization --------------
        // 稍后详细分析 getMinBufferSize
        int minBuffSize = AudioTrack.getMinBufferSize(TEST_SR, TEST_CONF, TEST_FORMAT);
        // 创建一个 AudioTrack 实例
        AudioTrack track = new AudioTrack(TEST_STREAM_TYPE, TEST_SR, TEST_CONF, TEST_FORMAT, 
                minBuffSize, TEST_MODE);
        byte data[] = new byte[minBuffSize/2];
        //--------    test        --------------
        // 调用 write() 写入回放数据
        track.write(data, 0, data.length);
        track.write(data, 0, data.length);
        // 调用 play() 开始播放
        track.play();
        float maxVol = AudioTrack.getMaxVolume();
        assertTrue(TEST_NAME, track.setStereoVolume(maxVol, maxVol) == AudioTrack.SUCCESS);
        //-------- tear down      --------------
        // 播放完成后，调用 release() 释放 AudioTrack 实例
        track.release();
    }
```

详细说明下 getMinBufferSize() 接口，字面意思是返回最小数据缓冲区的大小，它是声音能正常播放的最低保障，从函数参数来看，返回值取决于采样率、采样深度、声道数这三个属性。MODE_STREAM 模式下，应用程序重点参考其返回值然后确定分配多大的数据缓冲区。如果数据缓冲区分配得过小，那么播放声音会频繁遭遇 underrun，underrun 是指生产者（AudioTrack）提供数据的速度跟不上消费者（AudioFlinger::PlaybackThread）消耗数据的速度，反映到现实的后果就是声音断续卡顿，严重影响听觉体验。

```
// AudioTrack.java
/**
     * Returns the estimated minimum buffer size required for an AudioTrack
     * object to be created in the {@link #MODE_STREAM} mode.
     * The size is an estimate because it does not consider either the route or the sink,
     * since neither is known yet.  Note that this size doesn't
     * guarantee a smooth playback under load, and higher values should be chosen according to
     * the expected frequency at which the buffer will be refilled with additional data to play.
     * For example, if you intend to dynamically set the source sample rate of an AudioTrack
     * to a higher value than the initial source sample rate, be sure to configure the buffer size
     * based on the highest planned sample rate.
     * @param sampleRateInHz the source sample rate expressed in Hz.
     *   {@link AudioFormat#SAMPLE_RATE_UNSPECIFIED} is not permitted.
     * @param channelConfig describes the configuration of the audio channels.
     *   See {@link AudioFormat#CHANNEL_OUT_MONO} and
     *   {@link AudioFormat#CHANNEL_OUT_STEREO}
     * @param audioFormat the format in which the audio data is represented.
     *   See {@link AudioFormat#ENCODING_PCM_16BIT} and
     *   {@link AudioFormat#ENCODING_PCM_8BIT},
     *   and {@link AudioFormat#ENCODING_PCM_FLOAT}.
     * @return {@link #ERROR_BAD_VALUE} if an invalid parameter was passed,
     *   or {@link #ERROR} if unable to query for output properties,
     *   or the minimum buffer size expressed in bytes.
     */
    static public int getMinBufferSize(int sampleRateInHz, int channelConfig, int audioFormat) {
        int channelCount = 0;
        switch(channelConfig) {
        case AudioFormat.CHANNEL_OUT_MONO:
        case AudioFormat.CHANNEL_CONFIGURATION_MONO:
            channelCount = 1; // 单声道
            break;
        case AudioFormat.CHANNEL_OUT_STEREO:
        case AudioFormat.CHANNEL_CONFIGURATION_STEREO:
            channelCount = 2; // 双声道
            break;
        default:
            if (!isMultichannelConfigSupported(channelConfig)) {
                loge("getMinBufferSize(): Invalid channel configuration.");
                return ERROR_BAD_VALUE;
            } else {
                channelCount = AudioFormat.channelCountFromOutChannelMask(channelConfig);
            }
        }

        if (!AudioFormat.isPublicEncoding(audioFormat)) {
            loge("getMinBufferSize(): Invalid audio format.");
            return ERROR_BAD_VALUE;
        }

        // sample rate, note these values are subject to change
        // Note: AudioFormat.SAMPLE_RATE_UNSPECIFIED is not allowed
        if ( (sampleRateInHz < AudioFormat.SAMPLE_RATE_HZ_MIN) ||
                (sampleRateInHz > AudioFormat.SAMPLE_RATE_HZ_MAX) ) {
            loge("getMinBufferSize(): " + sampleRateInHz + " Hz is not a supported sample rate.");
            return ERROR_BAD_VALUE; // 采样率支持：4KHz~192KHz
        }

        // 调用 JNI 方法，下面分析该函数
        int size = native_get_min_buff_size(sampleRateInHz, channelCount, audioFormat);
        if (size <= 0) {
            loge("getMinBufferSize(): error querying hardware");
            return ERROR;
        }
        else {
            return size;
        }
    }

// android_media_AudioTrack.cpp
// ----------------------------------------------------------------------------
// returns the minimum required size for the successful creation of a streaming AudioTrack
// returns -1 if there was an error querying the hardware.
static jint android_media_AudioTrack_get_min_buff_size(JNIEnv *env,  jobject thiz,
    jint sampleRateInHertz, jint channelCount, jint audioFormat) {

    size_t frameCount;
    // 调用 AudioTrack::getMinFrameCount，这里不深究，到 native 层再分析
    // 这个函数用于确定至少设置多少个 frame 才能保证声音正常播放，也就是最低帧数
    const status_t status = AudioTrack::getMinFrameCount(&frameCount, AUDIO_STREAM_DEFAULT,
            sampleRateInHertz);
    if (status != NO_ERROR) {
        ALOGE("AudioTrack::getMinFrameCount() for sample rate %d failed with status %d",
                sampleRateInHertz, status);
        return -1;
    }
    const audio_format_t format = audioFormatToNative(audioFormat);
    if (audio_has_proportional_frames(format)) {
        const size_t bytesPerSample = audio_bytes_per_sample(format);
        return frameCount * channelCount * bytesPerSample; // PCM 数据最小缓冲区大小
    } else {
        return frameCount;
    }
}
```

可见`最小缓冲区的大小 = 最低帧数 * 声道数 * 采样深度，（采样深度以字节为单位）`，到这里大家应该有所明悟了吧，在视频中，如果帧数过低，那么画面会有卡顿感，对于音频，道理也是一样的。最低帧数如何求得，我们到 native 层再解释。

关于 MediaPlayer、AudioTrack，更多更详细的 API 接口说明请参考 Android Developer：

*   MediaPlayer：[https://developer.android.com/reference/android/media/MediaPlayer.html](https://developer.android.com/reference/android/media/MediaPlayer.html)
*   AudioTrack：[https://developer.android.com/reference/android/media/AudioTrack.html](https://developer.android.com/reference/android/media/AudioTrack.html)

2.2. AudioTrack Native API
--------------------------

AudioTrack Native API 四种数据传输模式：

<table><thead><tr><th>Transfer Mode</th><th>Description</th></tr></thead><tbody><tr><td>TRANSFER_CALLBACK</td><td>在 AudioTrackThread 线程中通过 audioCallback 回调函数主动从应用进程那里索取数据，ToneGenerator 采用这种模式</td></tr><tr><td>TRANSFER_OBTAIN</td><td>应用进程需要调用 obtainBuffer()/releaseBuffer() 填充数据，目前我还没有见到实际的使用场景</td></tr><tr><td>TRANSFER_SYNC</td><td>应用进程需要持续调用 write() 写数据到 FIFO，写数据时有可能遭遇阻塞（等待 AudioFlinger::PlaybackThread 消费之前的数据），基本适用所有的音频场景；对应于 AudioTrack Java API 的 MODE_STREAM 模式</td></tr><tr><td>TRANSFER_SHARED</td><td>应用进程将回放数据一次性付给 AudioTrack，适用于数据量小、时延要求高的场景；对应于 AudioTrack Java API 的 MODE_STATIC 模式</td></tr></tbody></table>

AudioTrack Native API 音频流类型：

<table><thead><tr><th>Stream Type</th><th>Description</th></tr></thead><tbody><tr><td>AUDIO_STREAM_VOICE_CALL</td><td>电话语音</td></tr><tr><td>AUDIO_STREAM_SYSTEM</td><td>系统声音</td></tr><tr><td>AUDIO_STREAM_RING</td><td>铃声声音，如来电铃声、闹钟铃声等</td></tr><tr><td>AUDIO_STREAM_MUSIC</td><td>音乐声音</td></tr><tr><td>AUDIO_STREAM_ALARM</td><td>警告音</td></tr><tr><td>AUDIO_STREAM_NOTIFICATION</td><td>通知音</td></tr><tr><td>AUDIO_STREAM_DTMF</td><td>DTMF 音（拨号盘按键音）</td></tr></tbody></table>

AudioTrack Native API 输出标识：

<table><thead><tr><th>AUDIO_OUTPUT_FLAG</th><th>Description</th></tr></thead><tbody><tr><td>AUDIO_OUTPUT_FLAG_DIRECT</td><td>表示音频流直接输出到音频设备，不需要软件混音，一般用于 HDMI 设备声音输出</td></tr><tr><td>AUDIO_OUTPUT_FLAG_PRIMARY</td><td>表示音频流需要输出到主输出设备，一般用于铃声类声音</td></tr><tr><td>AUDIO_OUTPUT_FLAG_FAST</td><td>表示音频流需要快速输出到音频设备，一般用于按键音、游戏背景音等对时延要求高的场景</td></tr><tr><td>AUDIO_OUTPUT_FLAG_DEEP_BUFFER</td><td>表示音频流输出可以接受较大的时延，一般用于音乐、视频播放等对时延要求不高的场景</td></tr><tr><td>AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD</td><td>表示音频流没有经过软件解码，需要输出到硬件解码器，由硬件解码器进行解码</td></tr></tbody></table>

我们根据不同的播放场景，使用不同的输出标识，如按键音、游戏背景音对输出时延要求很高，那么就需要置 AUDIO_OUTPUT_FLAG_FAST，具体可以参考 ToneGenerator、SoundPool 和 OpenSL ES。

一个 AudioTrack Natvie API 的测试例子（MODE_STATIC/TRANSFER_SHARED 模式），代码文件位置：frameworks/base/media/tests/audiotests/shared_mem_test.cpp：

```
int AudioTrackTest::Test01() {

    sp<MemoryDealer> heap;
    sp<IMemory> iMem;
    uint8_t* p;

    short smpBuf[BUF_SZ];
    long rate = 44100;
    unsigned long phi;
    unsigned long dPhi;
    long amplitude;
    long freq = 1237;
    float f0;

    f0 = pow(2., 32.) * freq / (float)rate;
    dPhi = (unsigned long)f0;
    amplitude = 1000;
    phi = 0;
    Generate(smpBuf, BUF_SZ, amplitude, phi, dPhi);  // fill buffer

    for (int i = 0; i < 1024; i++) {
        // 分配一块匿名共享内存
        heap = new MemoryDealer(1024*1024, "AudioTrack Heap Base");

        iMem = heap->allocate(BUF_SZ*sizeof(short));

        // 把音频数据拷贝到这块匿名共享内存上
        p = static_cast<uint8_t*>(iMem->pointer());
        memcpy(p, smpBuf, BUF_SZ*sizeof(short));

        // 构造一个 AudioTrack 实例，该 AudioTrack 的数据方式是 MODE_STATIC
        // 音频数据已经一次性拷贝到共享内存上了，不用再调用 track->write() 填充数据了
        sp<AudioTrack> track = new AudioTrack(AUDIO_STREAM_MUSIC,// stream type
               rate,
               AUDIO_FORMAT_PCM_16_BIT,// word length, PCM
               AUDIO_CHANNEL_OUT_MONO,
               iMem);

        // 检查 AudioTrack 实例是否构造成功饿了
        status_t status = track->initCheck();
        if(status != NO_ERROR) {
            track.clear();
            ALOGD("Failed for initCheck()");
            return -1;
        }

        // start play
        ALOGD("start");
        track->start(); // 开始播放

        usleep(20000);

        ALOGD("stop");
        track->stop(); // 停止播放
        iMem.clear();
        heap.clear();
        usleep(20000);
    }

    return 0;
}
```

上个小节还存在一个问题：AudioTrack::getMinFrameCount() 如何计算最低帧数呢？

首先要了解音频领域中，帧（frame）的概念：帧表示一个完整的声音单元，所谓的声音单元是指一个采样样本；如果是双声道，那么一个完整的声音单元就是 2 个样本，如果是 5.1 声道，那么一个完整的声音单元就是 6 个样本了。帧的大小（一个完整的声音单元的数据量）等于声道数乘以采样深度，即 `frameSize = channelCount * bytesPerSample`。帧的概念非常重要，无论是框架层还是内核层，都是以帧为单位去管理音频数据缓冲区的。

其次还得了解音频领域中，传输延迟（latency）的概念：传输延迟表示一个周期的音频数据的传输时间。可能有些读者一脸懵逼，一个周期的音频数据，这又是啥？我们再引入周期（period）的概念：Linux ALSA 把数据缓冲区划分为若干个块，dma 每传输完一个块上的数据即发出一个硬件中断，cpu 收到中断信号后，再配置 dma 去传输下一个块上的数据；一个块即是一个周期，周期大小（periodSize) 即是一个数据块的帧数。再回到传输延迟（latency），传输延迟等于周期大小除以采样率，即 `latency = periodSize / sampleRate`。

最后了解下音频重采样：音频重采样是指这样的一个过程——把一个采样率的数据转换为另一个采样率的数据。Android 原生系统上，音频硬件设备一般都工作在一个固定的采样率上（如 48 KHz），因此所有音轨数据都需要重采样到这个固定的采样率上，然后再输出。为什么这么做？系统中可能存在多个音轨同时播放，而每个音轨的采样率可能是不一致的；比如在播放音乐的过程中，来了一个提示音，这时需要把音乐和提示音混音并输出到硬件设备，而音乐的采样率和提示音的采样率不一致，问题来了，如果硬件设备工作的采样率设置为音乐的采样率的话，那么提示音就会失真；因此最简单见效的解决方法是：硬件设备工作的采样率固定一个值，所有音轨在 AudioFlinger 都重采样到这个采样率上，混音后输出到硬件设备，保证所有音轨听起来都不失真。

sample、frame、period、latency 这些概念与 Linux ALSA 及硬件设备的关系非常密切，这里点到即止，如有兴趣深入了解的话，可参考：[Linux ALSA 音频系统：逻辑设备篇](http://blog.csdn.net/zyuanyun/article/details/59180272#t6)

了解这些前置知识后，我们再分析 AudioTrack::getMinFrameCount() 这个函数：

```
status_t AudioTrack::getMinFrameCount(
        size_t* frameCount,
        audio_stream_type_t streamType,
        uint32_t sampleRate)
{
    if (frameCount == NULL) {
        return BAD_VALUE;
    }

    // 通过 binder 调用到 AudioFlinger::sampleRate()，取得硬件设备的采样率
    uint32_t afSampleRate;
    status_t status;
    status = AudioSystem::getOutputSamplingRate(&afSampleRate, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output sample rate for stream type %d; status %d",
                streamType, status);
        return status;
    }
    // 通过 binder 调用到 AudioFlinger::frameCount()，取得硬件设备的周期大小
    size_t afFrameCount;
    status = AudioSystem::getOutputFrameCount(&afFrameCount, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output frame count for stream type %d; status %d",
                streamType, status);
        return status;
    }
    // 通过 binder 调用到 AudioFlinger::latency()，取得硬件设备的传输延迟
    uint32_t afLatency;
    status = AudioSystem::getOutputLatency(&afLatency, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output latency for stream type %d; status %d",
                streamType, status);
        return status;
    }

    // When called from createTrack, speed is 1.0f (normal speed).
    // This is rechecked again on setting playback rate (TODO: on setting sample rate, too).
    // 根据 afSampleRate、afFrameCount、afLatency 计算出一个最低帧数
    *frameCount = calculateMinFrameCount(afLatency, afFrameCount, afSampleRate, sampleRate, 1.0f);

    // The formula above should always produce a non-zero value under normal circumstances:
    // AudioTrack.SAMPLE_RATE_HZ_MIN <= sampleRate <= AudioTrack.SAMPLE_RATE_HZ_MAX.
    // Return error in the unlikely event that it does not, as that's part of the API contract.
    if (*frameCount == 0) {
        ALOGE("AudioTrack::getMinFrameCount failed for streamType %d, sampleRate %u",
                streamType, sampleRate);
        return BAD_VALUE;
    }
    ALOGV("getMinFrameCount=%zu: afFrameCount=%zu, afSampleRate=%u, afLatency=%u",
            *frameCount, afFrameCount, afSampleRate, afLatency);
    return NO_ERROR;
}

// 有兴趣的可以研究 calculateMinFrameCount() 的实现，需大致了解重采样算法原理
static size_t calculateMinFrameCount(
        uint32_t afLatencyMs, uint32_t afFrameCount, uint32_t afSampleRate,
        uint32_t sampleRate, float speed)
{
    // Ensure that buffer depth covers at least audio hardware latency
    uint32_t minBufCount = afLatencyMs / ((1000 * afFrameCount) / afSampleRate);
    if (minBufCount < 2) {
        minBufCount = 2;
    }
    ALOGV("calculateMinFrameCount afLatency %u  afFrameCount %u  afSampleRate %u  "
            "sampleRate %u  speed %f  minBufCount: %u",
            afLatencyMs, afFrameCount, afSampleRate, sampleRate, speed, minBufCount);
    return minBufCount * sourceFramesNeededWithTimestretch(
            sampleRate, afFrameCount, afSampleRate, speed);
}

static inline size_t sourceFramesNeededWithTimestretch(
        uint32_t srcSampleRate, size_t dstFramesRequired, uint32_t dstSampleRate,
        float speed) {
    // required is the number of input frames the resampler needs
    size_t required = sourceFramesNeeded(srcSampleRate, dstFramesRequired, dstSampleRate);
    // to deliver this, the time stretcher requires:
    return required * (double)speed + 1 + 1; // accounting for rounding dependencies
}

// Returns the source frames needed to resample to destination frames.  This is not a precise
// value and depends on the resampler (and possibly how it handles rounding internally).
// Nevertheless, this should be an upper bound on the requirements of the resampler.
// If srcSampleRate and dstSampleRate are equal, then it returns destination frames, which
// may not be true if the resampler is asynchronous.
static inline size_t sourceFramesNeeded(
        uint32_t srcSampleRate, size_t dstFramesRequired, uint32_t dstSampleRate) {
    // +1 for rounding - always do this even if matched ratio (resampler may use phases not ratio)
    // +1 for additional sample needed for interpolation
    return srcSampleRate == dstSampleRate ? dstFramesRequired :
            size_t((uint64_t)dstFramesRequired * srcSampleRate / dstSampleRate + 1 + 1);
}
```

我们不深入分析 calculateMinFrameCount() 函数了，并不是说这个函数的流程有多复杂，而是它涉及到音频重采样的背景原理，说清楚 how 很容易，但说清楚 why 就很困难了。目前我们只需要知道：这个函数根据硬件设备的配置信息（采样率、周期大小、传输延迟）和音轨的采样率，计算出一个最低帧数（应用程序至少设置多少个帧才能保证声音正常播放）。

说点题外话，Anroid 2.2 时，AudioTrack::getMinFrameCount() 的处理很简单：

```
status_t AudioTrack::getMinFrameCount(
        int* frameCount,
        int streamType,
        uint32_t sampleRate)
{
    int afSampleRate;
    if (AudioSystem::getOutputSamplingRate(&afSampleRate, streamType) != NO_ERROR) {
        return NO_INIT;
    }
    int afFrameCount;
    if (AudioSystem::getOutputFrameCount(&afFrameCount, streamType) != NO_ERROR) {
        return NO_INIT;
    }
    uint32_t afLatency;
    if (AudioSystem::getOutputLatency(&afLatency, streamType) != NO_ERROR) {
        return NO_INIT;
    }

    // Ensure that buffer depth covers at least audio hardware latency
    uint32_t minBufCount = afLatency / ((1000 * afFrameCount) / afSampleRate);
    if (minBufCount < 2) minBufCount = 2;

    *frameCount = (sampleRate == 0) ? afFrameCount * minBufCount :
              afFrameCount * minBufCount * sampleRate / afSampleRate;
    return NO_ERROR;
}
```

从这段来看，最低帧数也是基于重采样来计算的，只不过这里的处理很粗糙：afFrameCount 是硬件设备处理单个数据块的帧数，afSampleRate 是硬件设备配置的采样率，sampleRate 是音轨的采样率，如果要把音轨数据重采样到 afSampleRate 上，那么反推算出应用程序最少传入的帧数为 afFrameCount * sampleRate / afSampleRate，而为了播放流畅，实际上还要大一点，所以再乘以一个系数（可参照 framebuffer 双缓冲，一个缓冲缓存当前的图像，一个缓冲准备下一幅的图像，这样图像切换更流畅），然后就得出一个可以保证播放流畅的最低帧数 `minFrameCount = (afFrameCount * sampleRate / afSampleRate) * minBufCount`。

为什么 Android 7.0 这方面的处理比 Android 2.2 复杂那么多呢？我想是两个原因：

1.  Android 7.0 充分考虑了边界处理
2.  Android 2.2 只支持采样率 4~48 KHz 的音轨，但 Android 7.0 支持采样率 4~192 KHz 的音轨，因此现在对重采样处理提出更严格的要求

3. AudioFlinger 概述
==================

AudioPolicyService 与 AudioFlinger 是 Android 音频系统的两大基本服务。前者是音频系统策略的制定者，负责音频设备切换的策略抉择、音量调节策略等；后者是音频系统策略的执行者，负责音频流设备的管理及音频流数据的处理传输，所以 AudioFlinger 也被认为是 Android 音频系统的引擎。

3.1. AudioFlinger 代码文件结构
------------------------

```
$ tree ./frameworks/av/services/audioflinger/
./frameworks/av/services/audioflinger/
├── Android.mk
├── AudioFlinger.cpp
├── AudioFlinger.h
├── AudioHwDevice.cpp
├── AudioHwDevice.h
├── AudioMixer.cpp
├── AudioMixer.h
├── AudioMixerOps.h
├── audio-resampler
│   ├── Android.mk
│   ├── AudioResamplerCoefficients.cpp
│   └── filter_coefficients.h
├── AudioResampler.cpp
├── AudioResamplerCubic.cpp
├── AudioResamplerCubic.h
├── AudioResamplerDyn.cpp
├── AudioResamplerDyn.h
├── AudioResamplerFirGen.h
├── AudioResamplerFirOps.h
├── AudioResamplerFirProcess.h
├── AudioResamplerFirProcessNeon.h
├── AudioResampler.h
├── AudioResamplerSinc.cpp
├── AudioResamplerSincDown.h
├── AudioResamplerSinc.h
├── AudioResamplerSincUp.h
├── AudioStreamOut.cpp
├── AudioStreamOut.h
├── AudioWatchdog.cpp
├── AudioWatchdog.h
├── BufferProviders.cpp
├── BufferProviders.h
├── Configuration.h
├── Effects.cpp
├── Effects.h
├── FastCapture.cpp
├── FastCaptureDumpState.cpp
├── FastCaptureDumpState.h
├── FastCapture.h
├── FastCaptureState.cpp
├── FastCaptureState.h
├── FastMixer.cpp
├── FastMixerDumpState.cpp
├── FastMixerDumpState.h
├── FastMixer.h
├── FastMixerState.cpp
├── FastMixerState.h
├── FastThread.cpp
├── FastThreadDumpState.cpp
├── FastThreadDumpState.h
├── FastThread.h
├── FastThreadState.cpp
├── FastThreadState.h
├── MODULE_LICENSE_APACHE2
├── NOTICE
├── PatchPanel.cpp
├── PatchPanel.h
├── PlaybackTracks.h
├── RecordTracks.h
├── ServiceUtilities.cpp
├── ServiceUtilities.h
├── SpdifStreamOut.cpp
├── SpdifStreamOut.h
├── StateQueue.cpp
├── StateQueue.h
├── StateQueueInstantiations.cpp
├── test-resample.cpp
├── tests
│   ├── Android.mk
│   ├── build_and_run_all_unit_tests.sh
│   ├── mixer_to_wav_tests.sh
│   ├── resampler_tests.cpp
│   ├── run_all_unit_tests.sh
│   ├── test-mixer.cpp
│   └── test_utils.h
├── Threads.cpp
├── Threads.h
├── TrackBase.h
└── Tracks.cpp

2 directories, 77 files
```

记得刚接触 Android 时，版本是 Android 2.2-Froyo，AudioFlinger 只有 3 个源文件：AudioFlinger.cpp、AudioMixer.cpp、AudioResampler.cpp。

现在文件多了许多，代码量就不用说了。但是接口及其基本流程一直没有改变的，只是更加模块化了，Google 把多个子类抽取出来独立成文件，比如 Threads.cpp、Tracks.cpp、Effects.cpp，而 AudioFlinger.cpp 只包含对外提供的服务接口了。另外相比以前，增加更多的功能特性，如 teesink、Offload、FastMixer、FastCapture、FastThread、PatchPanel 等，这里不对这些功能特性扩展描述，有兴趣的可以自行分析。

*   **AudioResampler.cpp**：重采样处理类，可进行采样率转换和声道转换；由录制线程 AudioFlinger::RecordThread 直接使用
*   **AudioMixer.cpp**：混音处理类，包括重采样、音量调节、声道转换等，其中的重采样复用了 AudioResampler；由回放线程 AudioFlinger::MixerThread 直接使用
*   **Effects.cpp**：音效处理类
*   **Tracks.cpp**：音频流管理类，可控制音频流的状态，如 start、stop、pause
*   **Threads.cpp**：回放线程和录制线程类；回放线程从 FIFO 读取回放数据并混音处理，然后写数据到输出流设备；录制线程从输入流设备读取录音数据并重采样处理，然后写数据到 FIFO
*   **AudioFlinger.cpp**：AudioFlinger 对外提供的服务接口

本文内容主要涉及 AudioFlinger.cpp、Threads.cpp、Tracks.cpp 这三个文件。

3.2. AudioFlinger 服务启动
----------------------

从 Android 7.0 开始，AudioFlinger 在系统启动时由 audioserver 加载（之前版本由 mediaserver 加载），详见 frameworks/av/media/audioserver/main_audioserver.cpp：

```
int main(int argc __unused, char **argv)
{
    // ......

    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p", sm.get());
    AudioFlinger::instantiate();
    AudioPolicyService::instantiate();
    RadioService::instantiate();
    SoundTriggerHwService::instantiate();
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```

可见 audioserver 把音频相关的服务都加载了，包括 AudioFlinger、AudioPolicyService、RadioService、SoundTriggerHwService。

main_audioserver.cpp 编译生成的可执行文件存放在 /system/bin/audioserver，系统启动时由 init 进程运行，详见 frameworks/av/media/audioserver/audioserver.rc：

```
service audioserver /system/bin/audioserver
    class main
    user audioserver
    # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
    group audio camera drmrpc inet media mediadrm net_bt net_bt_admin net_bw_acct
    ioprio rt 4
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
```

AudioFlinger 服务启动后，其他进程可以通过 ServiceManager 来获取其代理对象 IAudioFlinger，通过 IAudioFlinger 可以向 AudioFlinger 发出各种服务请求，从而完成自己的音频业务。

3.3. AudioFlinger 服务接口
----------------------

AudioFlinger 对外提供的主要的服务接口如下：

<table><thead><tr><th>Interface</th><th>Description</th></tr></thead><tbody><tr><td>sampleRate</td><td>获取硬件设备的采样率</td></tr><tr><td>format</td><td>获取硬件设备的音频格式</td></tr><tr><td>frameCount</td><td>获取硬件设备的周期帧数</td></tr><tr><td>latency</td><td>获取硬件设备的传输延迟</td></tr><tr><td>setMasterVolume</td><td>调节主输出设备的音量</td></tr><tr><td>setMasterMute</td><td>静音主输出设备</td></tr><tr><td>setStreamVolume</td><td>调节指定类型的音频流的音量，这种调节不影响其他类型的音频流的音量</td></tr><tr><td>setStreamMute</td><td>静音指定类型的音频流</td></tr><tr><td>setVoiceVolume</td><td>调节通话音量</td></tr><tr><td>setMicMute</td><td>静音麦克风输入</td></tr><tr><td>setMode</td><td>切换音频模式：音频模式有 4 种，分别是 Normal、Ringtone、Call、Communicatoin</td></tr><tr><td>setParameters</td><td>设置音频参数：往下调用 HAL 层相应接口，常用于切换音频通道</td></tr><tr><td>getParameters</td><td>获取音频参数：往下调用 HAL 层相应接口</td></tr><tr><td>openOutput</td><td>打开输出流：打开输出流设备，并创建 PlaybackThread 对象</td></tr><tr><td>closeOutput</td><td>关闭输出流：移除并销毁 PlaybackThread 上面挂着的所有的 Track，退出 PlaybackThread，关闭输出流设备</td></tr><tr><td>openInput</td><td>打开输入流：打开输入流设备，并创建 RecordThread 对象</td></tr><tr><td>closeInput</td><td>关闭输入流：退出 RecordThread，关闭输入流设备</td></tr><tr><td>createTrack</td><td>新建输出流管理对象： 找到对应的 PlaybackThread，创建输出流管理对象 Track，然后创建并返回该 Track 的代理对象 TrackHandle</td></tr><tr><td>openRecord</td><td>新建输入流管理对象：找到 RecordThread，创建输入流管理对象 RecordTrack，然后创建并返回该 RecordTrack 的代理对象 RecordHandle</td></tr></tbody></table>

可以归纳出 AudioFlinger 响应的服务请求主要有：

*   获取硬件设备的配置信息
*   音量调节
*   静音操作
*   音频模式切换
*   音频参数设置
*   输入输出流设备管理
*   音频流管理

就本文范围而言，主要涉及 openOutput() 和 createTrack() 这两个接口，后面也会详细分析这两个接口的流程。

3.4. AudioFlinger 回放录制线程
------------------------

AndioFlinger 作为 Android 的音频系统引擎，重任之一是负责输入输出流设备的管理及音频流数据的处理传输，这是由回放线程（PlaybackThread 及其派生的子类）和录制线程（RecordThread）进行的，我们简单看看回放线程和录制线程类关系：

![](https://img-blog.csdn.net/20170301225332422?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

*   **ThreadBase**：PlaybackThread 和 RecordThread 的基类
*   **RecordThread**：录制线程类，由 ThreadBase 派生
*   **PlaybackThread**：回放线程基类，同由 ThreadBase 派生
*   **MixerThread**：混音回放线程类，由 PlaybackThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_PRIMARY、AUDIO_OUTPUT_FLAG_FAST、AUDIO_OUTPUT_FLAG_DEEP_BUFFER 的音频流，MixerThread 可以把多个音轨的数据混音后再输出
*   **DirectOutputThread**：直输回放线程类，由 PlaybackThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_DIRECT 的音频流，这种音频流数据不需要软件混音，直接输出到音频设备即可
*   **DuplicatingThread**：复制回放线程类，由 MixerThread 派生，负责复制音频流数据到其他输出设备，使用场景如主声卡设备、蓝牙耳机设备、USB 声卡设备同时输出
*   **OffloadThread**：硬解回放线程类，由 DirectOutputThread 派生，负责处理标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流，这种音频流未经软件解码的（一般是 MP3、AAC 等格式的数据），需要输出到硬件解码器，由硬件解码器解码成 PCM 数据

PlaybackThread 中有个极为重要的函数 threadLoop()，当 PlaybackThread 被强引用时，threadLoop() 会真正运行起来进入循环主体，处理音频流数据相关事务，threadLoop() 大致流程如下（以 MixerThread 为例）：

```
bool AudioFlinger::PlaybackThread::threadLoop()
{
    // ......

    while (!exitPending())
    {
        // ......

        { // scope for mLock

            Mutex::Autolock _l(mLock);

            processConfigEvents_l();

            // ......

            if ((!mActiveTracks.size() && systemTime() > mStandbyTimeNs) ||
                                   isSuspended()) {
                // put audio hardware into standby after short delay
                if (shouldStandby_l()) {

                    threadLoop_standby();

                    mStandby = true;
                }

                // ......
            }
            // mMixerStatusIgnoringFastTracks is also updated internally
            mMixerStatus = prepareTracks_l(&tracksToRemove);

            // ......
        } // mLock scope ends

        // ......

        if (mBytesRemaining == 0) {
            mCurrentWriteLength = 0;
            if (mMixerStatus == MIXER_TRACKS_READY) {
                // threadLoop_mix() sets mCurrentWriteLength
                threadLoop_mix();
            }
            // ......
        }

        // ......

        if (!waitingAsyncCallback()) {
            // mSleepTimeUs == 0 means we must write to audio hardware
            if (mSleepTimeUs == 0) {
                // ......
                if (mBytesRemaining) {
                    // FIXME rewrite to reduce number of system calls
                    ret = threadLoop_write();
                    lastWriteFinished = systemTime();
                    delta = lastWriteFinished - mLastWriteTime;
                    if (ret < 0) {
                        mBytesRemaining = 0;
                    } else {
                        mBytesWritten += ret;
                        mBytesRemaining -= ret;
                        mFramesWritten += ret / mFrameSize;
                    }
                }
                // ......
            }
            // ......
        }

        // Finally let go of removed track(s), without the lock held
        // since we can't guarantee the destructors won't acquire that
        // same lock.  This will also mutate and push a new fast mixer state.
        threadLoop_removeTracks(tracksToRemove);
        tracksToRemove.clear();

        // ......
    }

    threadLoop_exit();

    if (!mStandby) {
        threadLoop_standby();
        mStandby = true;
    }

    // ......
    return false;
}
```

1.  threadLoop() 循环的条件是 exitPending() 返回 false，如果想要 PlaybackThread 结束循环，则可以调用 requestExit() 来请求退出；
2.  processConfigEvents_l() ：处理配置事件；当有配置改变的事件发生时，需要调用 sendConfigEvent_l() 来通知 PlaybackThread，这样 PlaybackThread 才能及时处理配置事件；常见的配置事件是切换音频通路；
3.  检查此时此刻是否符合 standby 条件，比如当前并没有 ACTIVE 状态的 Track（mActiveTracks.size() = 0），那么调用 threadLoop_standby() 关闭音频硬件设备以节省能耗；
4.  prepareTracks_l()： 准备音频流和混音器，该函数非常复杂，这里不详细分析了，仅列一下流程要点：  
    *   遍历 mActiveTracks，逐个处理 mActiveTracks 上的 Track，检查该 Track 是否为 ACTIVE 状态；
    *   如果 Track 设置是 ACTIVE 状态，则再检查该 Track 的数据是否准备就绪了；
    *   根据音频流的音量值、格式、声道数、音轨的采样率、硬件设备的采样率，配置好混音器参数；
    *   如果 Track 的状态是 PAUSED 或 STOPPED，则把该 Track 添加到 tracksToRemove 向量中；
5.  threadLoop_mix()：读取所有置了 ACTIVE 状态的音频流数据，混音器开始处理这些数据；
6.  threadLoop_write()： 把混音器处理后的数据写到输出流设备；
7.  threadLoop_removeTracks()： 把 tracksToRemove 上的所有 Track 从 mActiveTracks 中移除出来；这样下一次循环时就不会处理这些 Track 了。

这里说说 PlaybackThread 与输出流设备的关系：PlaybackThread 实例与输出流设备是一一对应的，比方说 OffloadThread 只会将音频数据输出到 compress_offload 设备中，MixerThread(with FastMixer) 只会将音频数据输出到 low_latency 设备中。

从 Audio HAL 中，我们通常看到如下 4 种输出流设备，分别对应着不同的播放场景：

*   **primary_out**：主输出流设备，用于铃声类声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_PRIMARY 的音频流和一个 MixerThread 回放线程实例
*   **low_latency**：低延迟输出流设备，用于按键音、游戏背景音等对时延要求高的声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_FAST 的音频流和一个 MixerThread 回放线程实例
*   **deep_buffer**：音乐音轨输出流设备，用于音乐等对时延要求不高的声音输出，对应着标识为 AUDIO_OUTPUT_FLAG_DEEP_BUFFER 的音频流和一个 MixerThread 回放线程实例
*   **compress_offload**：硬解输出流设备，用于需要硬件解码的数据输出，对应着标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流和一个 OffloadThread 回放线程实例

其中 primary_out 设备是必须声明支持的，而且系统启动时就已经打开 primary_out 设备并创建好对应的 MixerThread 实例。其他类型的输出流设备并非必须声明支持的，主要是看硬件上有无这个能力。

可能有人产生这样的疑问：既然 primary_out 设备一直保持打开，那么能耗岂不是很大？这里阐释一个概念：输出流设备属于逻辑设备，并不是硬件设备。所以即使输出流设备一直保持打开，只要硬件设备不工作，那么就不会影响能耗。那么硬件设备什么时候才会打开呢？答案是 PlaybackThread 将音频数据写入到输出流设备时。

下图简单描述 AudioTrack、PlaybackThread、输出流设备三者的对应关系：

![](https://img-blog.csdn.net/20170301225411811?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们可以这么说：输出流设备决定了它对应的 PlaybackThread 是什么类型。怎么理解呢？意思是说：只有支持了该类型的输出流设备，那么该类型的 PlaybackThread 才有可能被创建。举个例子：只有硬件上具备硬件解码器，系统才建立 compress_offload 设备，然后播放 mp3 格式的音乐文件时，才会创建 OffloadThread 把数据输出到 compress_offload 设备上；反之，如果硬件上并不具备硬件解码器，系统则不应该建立 compress_offload 设备，那么播放 mp3 格式的音乐文件时，通过 MixerThread 把数据输出到其他输出流设备上。

那么有无可能出现这种情况：底层并不支持 compress_offload 设备，但偏偏有个标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流送到 AudioFlinger 了呢？这是不可能的。系统启动时，会检查并保存输入输出流设备的支持信息；播放器在播放 mp3 文件时，首先看 compress_offload 设备是否支持了，如果支持，那么不进行软件解码，直接把数据标识为 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD；如果不支持，那么先进行软件解码，然后把解码好的数据标识为 AUDIO_OUTPUT_FLAG_DEEP_BUFFER，前提是 deep_buffer 设备是支持了的；如果 deep_buffer 设备也不支持，那么把数据标识为 AUDIO_OUTPUT_FLAG_PRIMARY。

系统启动时，就已经打开 primary_out、low_latency、deep_buffer 这三种输出流设备，并创建对应的 MixerThread 了；而此时 DirectOutputThread 与 OffloadThread 不会被创建，直到标识为 AUDIO_OUTPUT_FLAG_DIRECT/AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流需要输出时，才开始创建 DirectOutputThread/OffloadThread 和打开 direct_out/compress_offload 设备。这一点请参考如下代码，注释非常清晰：

```
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    // ......
{
    // ......

    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices
    // open all output streams needed to access attached devices
    // ......
    for (size_t i = 0; i < mHwModules.size(); i++) {
        // ......
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        // This also validates mAvailableOutputDevices list
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            // ......
            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }
            // ......
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            address,
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            // ......
        }
        // open input streams needed to access attached devices to validate
        // mAvailableInputDevices list
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            // ......
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            // ......
        }
    }
    // ......

    updateDevicesAndOutputs();
}
```

其中 `mpClientInterface->openOutput()` 最终会调用到 `AudioFlinger::openOutput()`：打开输出流设备，并创建 PlaybackThread 对象：

```
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    // ......

    Mutex::Autolock _l(mLock);

    sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
    // ......
}

sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    // ......
    // 分配全局唯一的 audio_io_handle_t，可以理解它是回放线程的索引号
    if (*output == AUDIO_IO_HANDLE_NONE) {
        *output = nextUniqueId(); 
    }

    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

    //......

    // 打开音频输出流设备，HAL 层根据 flags 选择打开相关类型的输出流设备
    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;

    if (status == NO_ERROR) {

        PlaybackThread *thread;
        if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            // AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 音频流，创建 OffloadThread 实例
            thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created offload output: ID %d thread %p", *output, thread);
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                || !isValidPcmSinkFormat(config->format)
                || !isValidPcmSinkChannelMask(config->channel_mask)) {
            // AUDIO_OUTPUT_FLAG_DIRECT 音频流，创建 DirectOutputThread 实例
            thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created direct output: ID %d thread %p", *output, thread);
        } else {
            // 其他标识的音频流，创建 MixerThread 实例
            thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created mixer output: ID %d thread %p", *output, thread);
        }
        // 把 audio_io_handle_t 和 PlaybackThread 添加到键值对向量 mPlaybackThreads 中
        // 键值对向量 mPlaybackThreads 中，由于 audio_io_handle_t 和 PlaybackThread 是一
        // 一对应的关系，所以拿到一个 audio_io_handle_t，就能找到它对应的 PlaybackThread
        // 所以可以理解 audio_io_handle_t 为 PlaybackThread 的索引号
        mPlaybackThreads.add(*output, thread);
        return thread;
    }

    return 0;
}
```

3.5. AudioFlinger OffloadThread
-------------------------------

> 系统启动时，就已经打开 primary_out、low_latency、deep_buffer 这三种输出流设备，并创建对应的 MixerThread 了；而此时 DirectOutputThread 与 OffloadThread 不会被创建，直到标识为 AUDIO_OUTPUT_FLAG_DIRECT/AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 的音频流需要输出时，才开始创建 DirectOutputThread/OffloadThread 和打开 direct_out/compress_offload 设备。

这里不知大家有无疑问：为什么 DirectOutputThread 与 OffloadThread 会被单独对待？DirectOuputThread 使用率较低，尚可以理解，但 OffloadThread 的使用率还是很高的，为什么不让 OffloadThread/compress_offload 设备也进入待命状态呢？

要回答这个问题：我们首先得明白 compress_offload 设备是什么东东，与其他输出流设备有什么不同。先看个图：

![](https://img-blog.csdn.net/20170301225549613?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

红色的是 Offload 音频流，它与其他音频流有什么本质的不同？Offload 音频流是未经 NuPlayerDecoder 进行解码的（NuPlayerDecoder 设置了 Passthrough 模式），所以必须把这些音频流数据送到 DSP，让 DSP 对其解码，解码后的 PCM 数据再送到 Codec 输出。

compress_offload 设备，说白了，就是驱动 DSP 解码数据、Codec 输出声音。 而 DSP 要解码数据，首先得知道数据的编码信息，如编码器 codec_id、采样率 sample_rate、声道数 channel_mask、比特率 bit_rate 等信息，由于 DSP 并没有实现 DataSource/Parser 部件，不能自己解析数据的编码信息，所以得有 “人” 告诉它，这个 “人” 无疑是 compress_offload 设备。

AudioTrack 构造函数有个 offloadInfo 的参数，参数原型定义如下：

```
/* Additional information about compressed streams offloaded to
 * hardware playback
 * The version and size fields must be initialized by the caller by using
 * one of the constants defined here.
 */
typedef struct {
    uint16_t version;                   // version of the info structure
    uint16_t size;                      // total size of the structure including version and size
    uint32_t sample_rate;               // sample rate in Hz
    audio_channel_mask_t channel_mask;  // channel mask
    audio_format_t format;              // audio format
    audio_stream_type_t stream_type;    // stream type
    uint32_t bit_rate;                  // bit rate in bits per second
    int64_t duration_us;                // duration in microseconds, -1 if unknown
    bool has_video;                     // true if stream is tied to a video stream
    bool is_streaming;                  // true if streaming, false if local playback
} audio_offload_info_t;
```

NuPlayer DataSource/Parser 解析 mp3、flac 等文件得到数据编码信息，并在构造 AudioTrack 实例时作为参数传入，AudioFlinger 将基于这些编码信息打开 compress_offload 设备。

到这里，大家明白了吗？每个 mp3/flac 文件的编码信息可能是不一样的，比如 a.mp3 文件的编码信息是 mp3&44.1KHZ&16bit… ，而 b.flac 文件的编码信息是 flac&48KHz&24bit…； 播放 a.mp3 时，AudioFlinger 打开一个配置为 mp3&44.1KHz&16bit… 的 compress_offload 设备，接着播放 b.flac，就需要关闭之前的 compress_offload 设备，重新打开一个配置为 flac&48KHz&24bit… 的 compress_offload 设备。所以系统不会提前打开 compress_offload 设备，只有等到播放 mp3、flac 时取到明确的数据编码信息，才基于这些编码信息打开 compress_offload 设备。

编码信息包含很多条目，切换音源时，是否编码信息有一点点不一样，都需要重新打开 compress_offload 设备呢？不能运行时更新信息到 DSP 吗？其实 stagefright 和 compress_offload 是支持运行期更新某些信息的，也就是无缝切换，至于是哪些信息，依赖于 DSP 算法实现；有兴趣深入的可以参考 sendMetaDataToHal() 和 compress_set_gapless_metadata() 。

最后附上 NuPlayer Offload Playback 初始化流程，摘自《Linux Audio Compress Offload Playback and PCM Offload Playback》：

> 1.  NuPlayer gets the file format metadata based on the type created by the source parser and decoder. For compress offload, DecoderPassThrough is created, causing the software decoder to enter Passthrough mode and not perform decoding.
> 2.  NuPlayer calls CanOffloadStream, which in turn calls the APM API isOffloadSupported to  
>     decide if a stream can be offloaded.
> 3.  NuPlayer creates a new NuPlayerRenderer, which then creates a new AudioTrack. Based on  
>     the information NuPlayer has received from the APM, it sets the flag FLAG_OFFLOAD_AUDIO, which passes to NuPlayerRenderer.
> 4.  AudioTrack then calls getOutput(), which gets the profile information from the APM. The  
>     APM then calls openOutput(), which goes to AudioFlinger. Based on the AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD flag previously set by AudioPlayer, AudioFlinger creates the offload thread. AudioFlinger also calls open_output_stream() which goes to the HAL. The HAL then creates the offload callback thread used to notify the write and drain completion events.
> 5.  AudioTrack creates the track to be used for this stream.
> 6.  NuPlayerRenderer issues a start command to AudioSink, which then calls start_output_stream(). The HAL sends the appropriate mixer control to the back-end (BE) platform driver to configure the output device.
> 7.  The HAL opens the compress driver and sends the codec and stream parameters to the  
>     compress driver though the IOCTL SNDRV_COMPRESS_SET_PARAMS. The compress driver opens an ASM session accordingly and requests the ASM driver to allocate the buffers to be used for this session.

3.6. AudioFlinger 音频流管理
-----------------------

从 AudioTrack、PlaybackThread、输出流设备三者的关系图中，我们看到 AudioTrack 把音频流数据送入到对应的 PlaybackThread 中，那么应用进程想控制这些音频流的话，比如开始播放 start()、停止播放 stop()、暂停播放 pause()，怎么办呢？注意应用进程与 AudioFlinger 并不在一个进程上。这就需要 AudioFlinger 提供音频流管理功能，并提供一套通讯接口可以让应用进程跨进程控制 AudioFlinger 中的音频流状态（通讯接口参考下一章的描述，暂且不表）。

AudioFlinger 音频流管理由 AudioFlinger::PlaybackThread::Track 实现，Track 与 AudioTrack 是一对一的关系，一个 AudioTrack 创建后，那么 AudioFlinger 会创建一个 Track 与之对应；PlaybackThread 与 AudioTrack/Track 是一对多的关系，一个 PlaybackThread 可以挂着多个 Track。

具体来说：AudioTrack 创建后，AudioPolicyManager 根据 AudioTrack 的输出标识和流类型，找到对应的输出流设备和 PlaybackThread（如果没有找到的话，则系统会打开对应的输出流设备并新建一个 PlaybackThread），然后创建一个 Track 并挂到这个 PlaybackThread 下面。

PlaybackThread 有两个私有成员向量与此强相关：

*   **mTracks**：该 PlaybackThread 创建的所有 Track 均添加保存到这个向量中
*   **mActiveTracks**：只有需要播放（设置了 ACTIVE 状态）的 Track 会添加到这个向量中；PlaybackThread 会从该向量上找到所有设置了 ACTIVE 状态的 Track，把这些 Track 数据混音后写到输出流设备

音频流控制最常用的三个接口：

*   **AudioFlinger::PlaybackThread::Track::start**：开始播放：把该 Track 置 ACTIVE 状态，然后添加到 mActiveTracks 向量中，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变
*   **AudioFlinger::PlaybackThread::Track::stop**：停止播放：把该 Track 置 STOPPED 状态，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变
*   **AudioFlinger::PlaybackThread::Track::pause**：暂停播放：把该 Track 置 PAUSING 状态，最后调用 AudioFlinger::PlaybackThread::broadcast_l() 告知 PlaybackThread 情况有变

AudioFlinger::PlaybackThread::threadLoop() 得悉情况有变后，调用 prepareTracks_l() 重新准备音频流和混音器：ACTIVE 状态的 Track 会添加到 mActiveTracks，此外的 Track 会从 mActiveTracks 上移除出来，然后重新准备 AudioMixer。

可见这三个音频流控制接口是非常简单的，主要是设置一下 Track 的状态，然后发个事件通知 PlaybackThread 就行，复杂的处理都在 AudioFlinger::PlaybackThread::threadLoop() 中了。

3.7. AudioFlinger 混音器处理
-----------------------

// TODO: …

4. AudioTrack 实例创建
==================

现在我们开始分析 AudioTrack 的创建过程，特别留意 AudioTrack 与 AudioFlinger 如何建立联系、用于 AudioTrack 与 AudioFlinger 交换数据的匿名共享内存如何分配。

4.1. AudioTrack & AudioFlinger 相关类
----------------------------------

首先看一下 AudioTrack & AudioFlinger 的类图，理一下 AudioFlinger 的主要类及其关系、AudioTrack 与 AudioFlinger 之间的联系，后面将以该图为脉络展开分析。

![](https://img-blog.csdn.net/20170301225623396?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

*   **AudioFlinger::PlaybackThread**：回放线程基类，不同输出标识的音频流对应不同类型的 PlaybackThread 实例（分为四种：MixerThread、DirectOutputThread、DuplicatingThread、OffloadThread），具体见 `3.4. AudioFlinger 回放录制线程` 小节，所有的 PlaybackThread 实例都会添加到 AudioFlinger.mPlaybackThreads 向量中；这个向量的定义： `DefaultKeyedVector< audio_io_handle_t, sp<PlaybackThread> > mPlaybackThreads;`，可见 audio_io_handle_t 是与 PlaybackThread 是一一对应的，由已知的 audio_io_handle_t 就能找到对应的 PlaybackThread；audio_io_handle_t 在创建 PlaybackThread 时由系统分配，这个值是全局唯一的
*   **AudioFlinger::PlaybackThread::Track**：音频流管理类，创建一块匿名共享内存用于 AudioTrack 与 AudioFlinger 之间的数据交换（方便起见，这块匿名共享内存，以后均简单称为 FIFO），同时实现 start()、stop()、pause() 等音频流常用控制手段；注意，多个 Track 对象可能都注册到同一个 PlaybackThread 中（尤其对于 MixerThread 而言，一个 MixerThread 往往挂着多个 Track 对象），这多个 Track 对象都会添加到 PlaybackThread.mTracks 向量中统一管理
*   **AudioFlinger::TrackHandle**：Track 对象只负责音频流管理业务，对外并没有提供跨进程的 Binder 调用接口，而应用进程又需要对音频流进行控制，所以需要一个对象来代理 Track 的跨进程通讯，这个角色就是 TrackHandle，AudioTrack 通过它与 Track 交互
*   **AudioTrack**：Android 音频系统对外提供的一个 API 类，负责音频流数据输出；每个音频流对应着一个 AudioTrack 实例，不同输出标识的 AudioTrack 会匹配到不同的 AudioFlinger::PlaybackThread；AudioTrack 与 AudioFlinger::PlaybackThread 之间通过 FIFO 来交换音频数据，AudioTrack 是 FIFO 生产者，AudioFlinger::PlaybackThread 是 FIFO 消费者
*   **AudioTrack::AudioTrackThread**：数据传输模式为 TRANSFER_CALLBACK 时，需要创建该线程，它通过调用 audioCallback 回调函数主动从用户进程处索取数据并填充到 FIFO 上；数据传输模式为 TRANSFER_SYNC 时，则不需要创建这个线程，因为用户进程会持续调用 AudioTrack.write() 填充数据到 FIFO；数据传输模式为 TRANSFER_SHARED 时，也不需要创建这个线程，因为用户进程会创建一块匿名共享内存，并把要播放的音频数据一次性拷贝到这块匿名共享内存上了
*   **IAudioTrack**：IAudioTrack 是链结 AudioTrack 与 AudioFlinger 的桥梁；它在 AudioTrack 端的对象是 BpAudioTrack，在 AudioFlinger 端的对象是 BnAudioTrack，从图中不难看出，AudioFlinger::TrackHandle 继承自 BnAudioTrack，而 AudioFlinger::TrackHandle 恰恰是 AudioFlinger::PlaybackThread::Track 的代理对象，所以 AudioTrack 得到 IAudioTrack 实例后，就可以调用 IAudioTrack 的接口与 AudioFlinger::PlaybackThread::Track 交互

**audio_io_handle_t**：

这里再详细说明一下 audio_io_handle_t，它是 AudioTrack/AudioRecord/AudioSystem、AudioFlinger、AudioPolicyManager 之间一个重要的链结点。`3.4. AudioFlinger 回放录制线程` 小节在 AudioFlinger::openOutput_l() 注释中大致说明了它的来历及其作用，现在回顾下：当打开输出流设备及创建 PlaybackThread 时，系统会分配一个全局唯一的值作为 audio_io_handle_t，并把 audio_io_handle_t 和 PlaybackThread 添加到键值对向量 mPlaybackThreads 中，由于 audio_io_handle_t 和 PlaybackThread 是一一对应的关系，因此拿到一个 audio_io_handle_t，就能遍历键值对向量 mPlaybackThreads 找到它对应的 PlaybackThread，可以简单理解 audio_io_handle_t 为 PlaybackThread 的索引号或线程 id。由于 audio_io_handle_t 具有 PlaybackThread 索引特性，所以应用进程想获取 PlaybackThread 某些信息的话，只需要传入对应的 audio_io_handle_t 即可。例如 AudioFlinger::format(audio_io_handle_t output)，这是 AudioFlinger 的一个服务接口，用户进程可以通过该接口获取某个 PlaybackThread 配置的音频格式：

```
audio_format_t AudioFlinger::format(audio_io_handle_t output) const
{
    Mutex::Autolock _l(mLock);
    // checkPlaybackThread_l() 根据传入的 audio_io_handle_t，从键值对向量
    // mPlaybackThreads 中找到它对应的 PlaybackThread
    PlaybackThread *thread = checkPlaybackThread_l(output);
    if (thread == NULL) {
        ALOGW("format() unknown thread %d", output);
        return AUDIO_FORMAT_INVALID;
    }
    return thread->format();
}

AudioFlinger::PlaybackThread *AudioFlinger::checkPlaybackThread_l(audio_io_handle_t output) const
{
    return mPlaybackThreads.valueFor(output).get();
}
```

4.2. AudioTrack 构造过程
--------------------

当我们构造一个 AudioTrack 实例时（以 MODE_STREAM/TRANSFER_SYNC 模式为例，这也是最常用的模式了，此时 sharedBuffer 为空），系统都发生了什么事？阐述下大致流程：

1.  如果 cbf（audioCallback 回调函数）非空，那么创建 AudioTrackThread 线程处理 audioCallback 回调函数（MODE_STREAM 模式时，cbf 为空）；
2.  根据 streamType（流类型）、flags（输出标识）等参数调用 AudioSystem::getOutputForAttr()；经过一系列的调用，进入 AudioPolicyManager::getOutputForDevice()：  
    *   如果输出标识置了 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 或 AUDIO_OUTPUT_FLAG_DIRECT，那么最终调用 AudioFlinger::openOutput() 打开输出标识对应的输出流设备并创建相应的 PlaybackThread，保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；
    *   如果输出标识是其他类型，那么根据策略选择一个输出流设备和 PlaybackThread，并保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；别忘了在 `3.4. AudioFlinger 回放录制线程` 小节中提到：系统启动时，就已经打开 primary_out、low_latency、deep_buffer 这三种输出流设备，并创建对应的 PlaybackThread 了；
3.  通过 Binder 机制调用 AudioFlinger::createTrack()（注意 step2 中 AudioTrack 已经拿到一个 audio_io_handle_t 了，此时把这个 audio_io_handle_t 传入给 createTrack()）：  
    *   根据传入的 audio_io_handle_t 找到它对应的 PlaybackThread；
    *   PlaybackThread 新建一个音频流管理对象 Track；Track 构造时会分配一块匿名共享内存用于 AudioFlinger 与 AudioTrack 的数据交换缓冲区（FIFO）及其控制块（audio_track_cblk_t），并创建一个 AudioTrackServerProxy 对象（PlaybackThread 将使用它从 FIFO 上取得可读数据的位置）；
    *   最后新建一个 Track 的通讯代理 TrackHandle，并以 IAudioTrack 作为返回值给 AudioTrack（TrackHandle、BnAudioTrack、BpAudioTrack、IAudioTrack 的关系见上一个小节）；
4.  通过 IAudioTrack 接口，取得 AudioFlinger 中的 FIFO 控制块（audio_track_cblk_t），由此再计算得到 FIFO 的首地址；
5.  创建一个 AudioTrackClientProxy 对象（AudioTrack 将使用它从 FIFO 上取得可用空间的位置）；

AudioTrack 由此建立了和 AudioFlinger 的全部联系工作：

*   通过 IAudioTrack 接口可以控制该音轨的状态，例如 start、stop、pause
*   持续写入数据到 FIFO 上，实现音频连续播放
*   通过 audio_io_handle_t，可以找到它对应的 PlaybackThread，从而查询该 PlaybackThread 的相关信息，如所设置的采样率、格式等等

构造 1 个 AudioTrack 实例时，AudioFlinger 会有 1 个 PlaybackThread 实例、1 个 Track 实例、1 个 TrackHandle 实例、1 个 AudioTrackServerProxy 实例、1 块 FIFO 与之对应。

当同时构造 1 个 `AudioTrack with AUDIO_OUTPUT_FLAG_PRIMARY`、1 个 `AudioTrack with AUDIO_OUTPUT_FLAG_FAST`、3 个 `AudioTrack with AUDIO_OUTPUT_FLAG_DEEP_BUFFER`、1 个 `AudioTrack with AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD`、1 个 `AudioTrack with AUDIO_OUTPUT_FLAG_DIRECT` 时（事实上，Android 音频策略不允许出现这种情形的），AudioFlinger 拥有的 PlaybackThread、Track、TrackHandle 实例如下图所示：

![](https://img-blog.csdn.net/20170301225817460?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

最后附上相关代码的流程分析，我本意是不多贴代码的，但不上代码总觉得缺点什么，这里我尽量把代码精简，提取主干，忽略细节。

```
AudioTrack::AudioTrack(
        audio_stream_type_t streamType,    // 音频流类型：如 Music、Voice-Call、DTMF、Alarm 等等
        uint32_t sampleRate,               // 采样率：如 16KHz、44.1KHz、48KHz 等等
        audio_format_t format,             // 音频格式：如 PCM、MP3、AAC 等等
        audio_channel_mask_t channelMask,  // 声道数：如 Mono（单声道）、Stereo（双声道）
        const sp<IMemory>& sharedBuffer,   // 共享内存缓冲区：数据模式是 MODE_STATIC 时使用，数据模式是 MODE_STREAM 时为空
        audio_output_flags_t flags,        // 输出标识位，详见 AUDIO_OUTPUT_FLAG 描述
        callback_t cbf,                    // 回调函数
        void* user,                        // 回调函数的参数
        uint32_t notificationFrames,
        int sessionId,
        transfer_type transferType,        // 数据传输类型
        const audio_offload_info_t *offloadInfo,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes,
        bool doNotReconnect)
    : mStatus(NO_INIT),
      mIsTimed(false),
      mPreviousPriority(ANDROID_PRIORITY_NORMAL),
      mPreviousSchedulingGroup(SP_DEFAULT),
      mPausedPosition(0),
      mSelectedDeviceId(AUDIO_PORT_HANDLE_NONE)
{
    mStatus = set(streamType, sampleRate, format, channelMask,
            0 /*frameCount*/, flags, cbf, user, notificationFrames,
            sharedBuffer, false /*threadCanCallJava*/, sessionId, transferType, offloadInfo,
            uid, pid, pAttributes, doNotReconnect);
}

status_t AudioTrack::set(
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t frameCount,
        audio_output_flags_t flags,        
        callback_t cbf,
        void* user,
        uint32_t notificationFrames,
        const sp<IMemory>& sharedBuffer,
        bool threadCanCallJava,
        int sessionId,
        transfer_type transferType,
        const audio_offload_info_t *offloadInfo,
        int uid,
        pid_t pid,
        const audio_attributes_t* pAttributes,
        bool doNotReconnect)
{
    // 参数格式合法性检查、音轨音量初始化

    // 如果 cbf 非空，那么创建 AudioTrackThread 线程处理 audioCallback 回调函数
    if (cbf != NULL) {
        mAudioTrackThread = new AudioTrackThread(*this, threadCanCallJava);
        mAudioTrackThread->run("AudioTrack", ANDROID_PRIORITY_AUDIO, 0 /*stack*/);
        // thread begins in paused state, and will not reference us until start()
    }

    // create the IAudioTrack
    status_t status = createTrack_l();

    //......
}

status_t AudioTrack::createTrack_l()
{
    // 获取 IAudioFlinger，通过 binder 请求 AudioFlinger 服务
    const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
    if (audioFlinger == 0) {
        ALOGE("Could not get audioflinger");
        return NO_INIT;
    }

    //......

    // AudioSystem::getOutputForAttr() 经过一系列的调用，进入 AudioPolicyManager::getOutputForDevice()
    // 如果输出标识置了 AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD 或 AUDIO_OUTPUT_FLAG_DIRECT，
    // 那么最终调用 AudioFlinger::openOutput() 打开输出标识对应的输出流设备并创建相关的
    // PlaybackThread，保存该 PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack；
    // 如果输出标识是其他类型，那么根据策略选择一个输出流设备和 PlaybackThread，并保存该
    // PlaybackThread 对应的 audio_io_handle_t 给 AudioTrack
    audio_io_handle_t output;
    status = AudioSystem::getOutputForAttr(attr, &output,
                                           (audio_session_t)mSessionId, &streamType, mClientUid,
                                           mSampleRate, mFormat, mChannelMask,
                                           mFlags, mSelectedDeviceId, mOffloadInfo);

    //......

    // 向 AudioFlinger 发出 createTrack 请求
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
    //......

    // AudioFlinger 创建 Track 对象时会分配一个 FIFO，这里获取 FIFO 的控制块
    sp<IMemory> iMem = track->getCblk();
    if (iMem == 0) {
        ALOGE("Could not get control block");
        return NO_INIT;
    }
    // 匿名共享内存首地址
    void *iMemPointer = iMem->pointer();
    if (iMemPointer == NULL) {
        ALOGE("Could not get control block pointer");
        return NO_INIT;
    }
    mAudioTrack = track; // 保存 AudioFlinger::PlaybackThread::Track 的代理对象 IAudioTrack
    mCblkMemory = iMem; // 保存匿名共享内存首地址

    // 控制块位于 AudioFlinger 分配的匿名共享内存的首部
    audio_track_cblk_t* cblk = static_cast<audio_track_cblk_t*>(iMemPointer);
    mCblk = cblk;
    mOutput = output; // 保存返回的 audio_io_handle_t，用它可以找到对应的 PlaybackThread
    //......

    // update proxy
    if (mSharedBuffer == 0) {
        // 当 mSharedBuffer 为空，意味着音轨数据模式为 MODE_STREAM，那么创建 AudioTrackClientProxy 对象
        mStaticProxy.clear();
        mProxy = new AudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
    } else {
        // 当 mSharedBuffer 非空，意味着音轨数据模式为 MODE_STATIC，那么创建 StaticAudioTrackClientProxy 对象
        mStaticProxy = new StaticAudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
        mProxy = mStaticProxy;
    }

    //......
}
```

AudioFlinger::createTrack()，顾名思义，创建一个 Track 对象，将用于音频流的控制：

```
sp<IAudioTrack> AudioFlinger::createTrack(
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t *frameCount,
        IAudioFlinger::track_flags_t *flags,
        const sp<IMemory>& sharedBuffer,
        audio_io_handle_t output,
        pid_t tid,
        int *sessionId,
        int clientUid,
        status_t *status)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;
    sp<Client> client;
    status_t lStatus;
    int lSessionId;

    //......

    {
        Mutex::Autolock _l(mLock);
        // 根据传入来的 audio_io_handle_t，找到对应的 PlaybackThread
        PlaybackThread *thread = checkPlaybackThread_l(output);
        if (thread == NULL) {
            ALOGE("no playback thread found for output handle %d", output);
            lStatus = BAD_VALUE;
            goto Exit;
        }

        //......

        // 在 PlaybackThread 上创建一个音频流管理对象 Track
        track = thread->createTrack_l(client, streamType, sampleRate, format,
                channelMask, frameCount, sharedBuffer, lSessionId, flags, tid, clientUid, &lStatus);
        //......

        setAudioHwSyncForSession_l(thread, (audio_session_t)lSessionId);
    }

    //......

    // 创建 Track 的通讯代理 TrackHandle 并返回它
    trackHandle = new TrackHandle(track);

Exit:
    *status = lStatus;
    return trackHandle;
}

sp<AudioFlinger::PlaybackThread::Track> AudioFlinger::PlaybackThread::createTrack_l(
        const sp<AudioFlinger::Client>& client,
        audio_stream_type_t streamType,
        uint32_t sampleRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        size_t *pFrameCount,
        const sp<IMemory>& sharedBuffer,
        int sessionId,
        IAudioFlinger::track_flags_t *flags,
        pid_t tid,
        int uid,
        status_t *status)
{
    size_t frameCount = *pFrameCount;
    sp<Track> track;
    status_t lStatus;

    bool isTimed = (*flags & IAudioFlinger::TRACK_TIMED) != 0;

    // ......

    { // scope for mLock
        Mutex::Autolock _l(mLock);

        // ......

        if (!isTimed) {
            // 创建 Track，等会再看看 Track 构造函数干些啥
            track = new Track(this, client, streamType, sampleRate, format,
                              channelMask, frameCount, NULL, sharedBuffer,
                              sessionId, uid, *flags, TrackBase::TYPE_DEFAULT);
        } else {
            // 创建 TimedTrack，带时间戳的 Track？这里不深究
            track = TimedTrack::create(this, client, streamType, sampleRate, format,
                    channelMask, frameCount, sharedBuffer, sessionId, uid);
        }

        // ......

        // 把创建的 Track 添加到 mTracks 向量中，方便 PlaybackThread 统一管理
        mTracks.add(track);

        // ......
    }

    lStatus = NO_ERROR;

Exit:
    *status = lStatus;
    return track;
}

// ----------------------------------------------------------------------------
// 如下是 TrackHandle 的相关代码，可以看到，TrackHandle 其实就是一个壳子，是 Track 的包装类
// 所有 TrackHandle 接口都是调向 Track 的
// Google 为什么要搞这么一则？Track 是 PlaybackThread 内部使用的，不适宜对外暴露，但应用进程
// 又确实需要控制音频流的状态（start、stop、pause），所以就采取这么一种方式实现

AudioFlinger::TrackHandle::TrackHandle(const sp<AudioFlinger::PlaybackThread::Track>& track)
    : BnAudioTrack(),
      mTrack(track)
{
}

AudioFlinger::TrackHandle::~TrackHandle() {
    // just stop the track on deletion, associated resources
    // will be freed from the main thread once all pending buffers have
    // been played. Unless it's not in the active track list, in which
    // case we free everything now...
    mTrack->destroy();
}

sp<IMemory> AudioFlinger::TrackHandle::getCblk() const {
    return mTrack->getCblk();
}

status_t AudioFlinger::TrackHandle::start() {
    return mTrack->start();
}

void AudioFlinger::TrackHandle::stop() {
    mTrack->stop();
}

void AudioFlinger::TrackHandle::flush() {
    mTrack->flush();
}

void AudioFlinger::TrackHandle::pause() {
    mTrack->pause();
}
// ----------------------------------------------------------------------------
```

最后，我们看看 Track 的构造过程，主要分析数据 FIFO 及它的控制块是如何分配的：

```
AudioFlinger::PlaybackThread::Track::Track(
            PlaybackThread *thread,
            const sp<Client>& client,
            audio_stream_type_t streamType,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            const sp<IMemory>& sharedBuffer,
            int sessionId,
            int uid,
            IAudioFlinger::track_flags_t flags,
            track_type type)
    :   TrackBase(thread, client, sampleRate, format, channelMask, frameCount,
                  (sharedBuffer != 0) ? sharedBuffer->pointer() : buffer,
                  sessionId, uid, flags, true /*isOut*/,
                  (type == TYPE_PATCH) ? ( buffer == NULL ? ALLOC_LOCAL : ALLOC_NONE) : ALLOC_CBLK,
                  type),
    mFillingUpStatus(FS_INVALID),
    // mRetryCount initialized later when needed
    mSharedBuffer(sharedBuffer),
    mStreamType(streamType),
    mName(-1),  // see note below
    mMainBuffer(thread->mixBuffer()),
    mAuxBuffer(NULL),
    mAuxEffectId(0), mHasVolumeController(false),
    mPresentationCompleteFrames(0),
    mFastIndex(-1),
    mCachedVolume(1.0),
    mIsInvalid(false),
    mAudioTrackServerProxy(NULL),
    mResumeToStopping(false),
    mFlushHwPending(false)
{
    // client == 0 implies sharedBuffer == 0
    ALOG_ASSERT(!(client == 0 && sharedBuffer != 0));

    ALOGV_IF(sharedBuffer != 0, "sharedBuffer: %p, size: %d", sharedBuffer->pointer(),
            sharedBuffer->size());

    // 检查 FIFO 控制块（audio_track_cblk_t）是否分配好了，上面代码并未分配 audio_track_cblk_t
    // 因此只可能是构造 TrackBase 时分配的，等下再看看 TrackBase 的构造函数
    if (mCblk == NULL) {
        return;
    }

    if (sharedBuffer == 0) {
        // 数据传输模式为 MODE_STREAM 模式，创建一个 AudioTrackServerProxy 对象
        // PlaybackThread 将持续使用它从 FIFO 上取得可读数据的位置
        mAudioTrackServerProxy = new AudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize, !isExternalTrack(), sampleRate);
    } else {
        // 数据传输模式为 MODE_STATIC 模式，创建一个 StaticAudioTrackServerProxy 对象
        mAudioTrackServerProxy = new StaticAudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize);
    }
    mServerProxy = mAudioTrackServerProxy;

    // 为 Track 分配一个名称，AudioMixer 会根据 TrackName 找到对应的 Track
    mName = thread->getTrackName_l(channelMask, format, sessionId);
    if (mName < 0) {
        ALOGE("no more track names available");
        return;
    }
    // ......
}

AudioFlinger::ThreadBase::TrackBase::TrackBase(
            ThreadBase *thread,
            const sp<Client>& client,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            int sessionId,
            int clientUid,
            IAudioFlinger::track_flags_t flags,
            bool isOut,
            alloc_type alloc,
            track_type type)
    :   RefBase(),
        mThread(thread),
        mClient(client),
        mCblk(NULL),
        // mBuffer
        mState(IDLE),
        mSampleRate(sampleRate),
        mFormat(format),
        mChannelMask(channelMask),
        mChannelCount(isOut ?
                audio_channel_count_from_out_mask(channelMask) :
                audio_channel_count_from_in_mask(channelMask)),
        mFrameSize(audio_is_linear_pcm(format) ?
                mChannelCount * audio_bytes_per_sample(format) : sizeof(int8_t)),
        mFrameCount(frameCount),
        mSessionId(sessionId),
        mFlags(flags),
        mIsOut(isOut),
        mServerProxy(NULL),
        mId(android_atomic_inc(&nextTrackId)),
        mTerminated(false),
        mType(type),
        mThreadIoHandle(thread->id())
{
    // ......

    // ALOGD("Creating track with %d buffers @ %d bytes", bufferCount, bufferSize);
    size_t size = sizeof(audio_track_cblk_t);
    size_t bufferSize = (buffer == NULL ? roundup(frameCount) : frameCount) * mFrameSize;
    if (buffer == NULL && alloc == ALLOC_CBLK) {
        // 这个 size 将是分配的匿名共享内存的大小
        // 等于控制块的大小（sizeof(audio_track_cblk_t)加上数据 FIFO的大小（bufferSize）
        // 待会看到这块内存的结构，就明白这样分配的意义了
        size += bufferSize;
    }

    if (client != 0) {
        // 分配一块匿名共享内存
        mCblkMemory = client->heap()->allocate(size);
        if (mCblkMemory == 0 ||
                (mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer())) == NULL) {
            ALOGE("not enough memory for AudioTrack size=%u", size);
            client->heap()->dump("AudioTrack");
            mCblkMemory.clear();
            return;
        }
    } else {
        // this syntax avoids calling the audio_track_cblk_t constructor twice
        mCblk = (audio_track_cblk_t *) new uint8_t[size];
        // assume mCblk != NULL
    }

    // construct the shared structure in-place.
    if (mCblk != NULL) {
        // 这是 C++ 的 placement new（定位创建对象）语法：new(@BUFFER) @CLASS();
        // 可以在特定内存位置上构造一个对象
        // 这里，在匿名共享内存首地址上构造了一个 audio_track_cblk_t 对象
        // 这样 AudioTrack 与 AudioFlinger 都能访问这个 audio_track_cblk_t 对象了
        new(mCblk) audio_track_cblk_t();

        // 如下分配数据 FIFO，将用于 AudioTrack 与 AudioFlinger 的数据交换
        switch (alloc) {
        // ......
        case ALLOC_CBLK:
            // clear all buffers
            if (buffer == NULL) {
                // 数据传输模式为 MODE_STREAM/TRANSFER_SYNC 时，数据 FIFO 的分配
                // 数据 FIFO 的首地址紧靠控制块（audio_track_cblk_t）之后
                //   |                                                         |
                //   | -------------------> mCblkMemory <--------------------- |
                //   |                                                         |
                //   +--------------------+------------------------------------+
                //   | audio_track_cblk_t |             Buffer                 |
                //   +--------------------+------------------------------------+
                //   ^                    ^
                //   |                    |
                //   mCblk               mBuffer
                mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
                memset(mBuffer, 0, bufferSize);
            } else {
                // 数据传输模式为 MODE_STATIC/TRANSFER_SHARED 时，直接指向 sharedBuffer
                // sharedBuffer 是应用进程分配的匿名共享内存，应用进程已经一次性把数据
                // 写到 sharedBuffer 来了，AudioFlinger 可以直接从这里读取
                //   +--------------------+    +-----------------------------------+
                //   | audio_track_cblk_t |    |            sharedBuffer           |
                //   +--------------------+    +-----------------------------------+
                //   ^                         ^
                //   |                         |
                //   mCblk                    mBuffer
                mBuffer = buffer;
            }
            break;
        // ......
        }

        // ......
    }
}
```

5. AudioTrack 数据写入
==================

AudioTrack 实例构造后，应用程序接着可以写入音频数据了。如之前所描述：AudioTrack 与 AudioFlinger 是 生产者 - 消费者 的关系：

*   **AudioTrack**：AudioTrack 在 FIFO 中找到一块可用空间，把用户传入的音频数据写入到这块可用空间上，然后更新写位置（对于 AudioFinger 来说，意味 FIFO 上有更多的可读数据了）；如果用户传入的数据量比可用空间要大，那么要把用户传入的数据拆分多次写入到 FIFO 中（AudioTrack 和 AudioFlinger 是不同的进程，AudioFlinger 同时也在不停地读取数据，所以 FIFO 可用空间是在不停变化的）
*   **AudioFlinger**：AudioFlinger 在 FIFO 中找到一块可读数据块，把可读数据拷贝到目的缓冲区上，然后更新读位置（对于 AudioTrack 来说，意味着 FIFO 上有更多的可用空间了）；如果 FIFO 上可读数据量比预期的要小，那么要进行多次的读取，才能积累到预期的数据量（AudioTrack 和 AudioFlinger 是不同的进程，AudioTrack 同时也在不停地写入数据，所以 FIFO 可读的数据量是在不停变化的）

上面的过程中，如果 AudioTrack 总能及时生产数据，并且 AudioFlinger 总能及时消耗掉这些数据，那么整个过程将是非常和谐的；但系统可能会发生异常，出现如下的状态：

*   Block：AudioFlinger 长时间不读取 FIFO 上的可读数据，使得 AudioTrack 长时间获取不到可用空间，无法写入数据；这种情况的根本原因大多是底层驱动发生阻塞异常，导致 AudioFlinger 无法继续写数据到硬件设备中，AudioFlinger 本身并没有错
*   Underrun：AudioTrack 写入数据的速度跟不上 AudioFlinger 读取数据的速度，使得 AudioFlinger 不能及时获取到预期的数据量，反映到现实的后果就是声音断续；这种情况的根本原因大多是应用程序不能及时写入数据或者缓冲区分配过小，AudioTrack 本身并没有错；AudioFlinger 针对这点做了容错处理：当发现 underrun 时，先陷入短时间的睡眠，不急着读取数据，让应用程序准备更多的数据（如果某一天做应用的哥们意识到自己的错误原来由底层的兄弟默默埋单了，会不会感动得哭了 ^_^）

5.1. AudioTrack 写数据流程
---------------------

我们看一下 AudioTrack 写数据的代码，流程很简单：obtainBuffer() 在 FIFO 中找到一块可用区间，memcpy() 把用户传入的音频数据拷贝到这个可用区间上，releaseBuffer() 更新写位置。

```
ssize_t AudioTrack::write(const void* buffer, size_t userSize, bool blocking)
{
    if (mTransfer != TRANSFER_SYNC) {
        return INVALID_OPERATION;
    }

    if (isDirect()) {
        AutoMutex lock(mLock);
        int32_t flags = android_atomic_and(
                            ~(CBLK_UNDERRUN | CBLK_LOOP_CYCLE | CBLK_LOOP_FINAL | CBLK_BUFFER_END),
                            &mCblk->mFlags);
        if (flags & CBLK_INVALID) {
            return DEAD_OBJECT;
        }
    }

    if (ssize_t(userSize) < 0 || (buffer == NULL && userSize != 0)) {
        // Sanity-check: user is most-likely passing an error code, and it would
        // make the return value ambiguous (actualSize vs error).
        ALOGE("AudioTrack::write(buffer=%p, size=%zu (%zd)", buffer, userSize, userSize);
        return BAD_VALUE;
    }

    size_t written = 0;
    Buffer audioBuffer;

    while (userSize >= mFrameSize) {
        // 单帧数据量 frameSize = channelCount * bytesPerSample
        //   对于双声道，16位采样的音频数据来说，frameSize = 2 * 2 = 4(bytes)
        // 用户传入的数据帧数 frameCount = userSize / frameSize
        audioBuffer.frameCount = userSize / mFrameSize;

        // obtainBuffer() 从 FIFO 上得到一块可用区间
        status_t err = obtainBuffer(&audioBuffer,
                blocking ? &ClientProxy::kForever : &ClientProxy::kNonBlocking);
        if (err < 0) {
            if (written > 0) {
                break;
            }
            if (err == TIMED_OUT || err == -EINTR) {
                err = WOULD_BLOCK;
            }
            return ssize_t(err);
        }

        // toWrite 是 FIFO 可用区间的大小，可能比 userSize（用户传入数据的大小）要小
        //   因此用户传入的数据可能要拆分多次拷贝到 FIFO 上
        // 注意：AudioTrack 和 AudioFlinger 是不同的进程，AudioFlinger 同时也在不停地
        //   消耗数据，所以 FIFO 可用区间是在不停变化的
        size_t toWrite = audioBuffer.size;
        memcpy(audioBuffer.i8, buffer, toWrite); // 把用户数据拷贝到 FIFO 可用区间
        buffer = ((const char *) buffer) + toWrite; // 未拷贝数据的位置
        userSize -= toWrite; // 未拷贝数据的大小
        written += toWrite; // 已拷贝数据的大小

        // releaseBuffer() 更新 FIFO 写位置
        // 对于 AudioFinger 来说，意味 FIFO 上有更多的可读数据
        releaseBuffer(&audioBuffer);
    }

    if (written > 0) {
        mFramesWritten += written / mFrameSize;
    }
    return written;
}
```

5.2. AudioFlinger 读数据流程
-----------------------

AudioFlinger 消费数据的流程稍微复杂一点，`3.4. AudioFlinger 回放录制线程` 小节中描述了 AudioFlinger::PlaybackThread::threadLoop() 工作流程，这里不累述了，我们把焦点放在 “如何从 FIFO 读取数据” 节点上。

我们以 DirectOutputThread/OffloadThread 为例说明（MixerThread 读数据也是类似的过程，只不过是在 AudioMixer 中进行的，`3.7. AudioFlinger 混音器处理` 小节中有相关描述）。

```
void AudioFlinger::DirectOutputThread::threadLoop_mix()
{
    // mFrameCount 是硬件设备（PCM 设备）处理单个数据块的帧数（周期大小）
    //   上层必须积累了足够多（mFrameCount）的数据，才写入到 PCM 设备
    //   所以 mFrameCount 也就是 AudioFlinger 预期的数据量
    size_t frameCount = mFrameCount;
    // mSinkBuffer 目的缓冲区，threadLoop_write() 会把 mSinkBuffer 上的数据写到 PCM 设备
    int8_t *curBuf = (int8_t *)mSinkBuffer;
    // output audio to hardware
    // FIFO 上可读的数据量可能要比预期的要小，因此可能需要多次读取才能积累足够的数据量
    // 注意：AudioTrack 和 AudioFlinger 是不同的进程，AudioTrack 同时也在不停地生产数据
    //   所以 FIFO 可读的数据量是在不停变化的
    while (frameCount) {
        AudioBufferProvider::Buffer buffer;
        buffer.frameCount = frameCount;
        // getNextBuffer() 从 FIFO 上获取可读数据块
        status_t status = mActiveTrack->getNextBuffer(&buffer);
        if (status != NO_ERROR || buffer.raw == NULL) {
            memset(curBuf, 0, frameCount * mFrameSize);
            break;
        }
        // memcpy() 把 FIFO 可读数据拷贝到 mSinkBuffer 目的缓冲区
        memcpy(curBuf, buffer.raw, buffer.frameCount * mFrameSize);
        frameCount -= buffer.frameCount;
        curBuf += buffer.frameCount * mFrameSize;
        // releaseBuffer() 更新 FIFO 读位置
        // 对于 AudioTrack 来说，意味着 FIFO 上有更多的可用空间
        mActiveTrack->releaseBuffer(&buffer);
    }
    mCurrentWriteLength = curBuf - (int8_t *)mSinkBuffer;
    mSleepTimeUs = 0;
    mStandbyTimeNs = systemTime() + mStandbyDelayNs;
    mActiveTrack.clear();
}
```

5.3. 环形 FIFO 管理
---------------

在上述过程中，不知大家有无意识到：整个过程中，最难的是如何协调生产者与消费者之间的步调。上文所说的 FIFO 是环形 FIFO，AudioTrack 写指针、AudioFlinger 读指针都是基于 FIFO 当前的读写位置来计算的。

*   AudioTrack 与 AudioFlinger 不在同一个进程上，怎么保证读写指针的线程安全
*   读写指针越过 FIFO 后，怎么处理
*   AudioTrack 写数据完成后，需要同步状态给 AudioFlinger，让 AudioFlinger 知道当前有可读数据了，而 AudioFlinger 读数据完成后，也需要同步状态给 AudioTrack，让 AudioTrack 知道当前有可用空间了；这里采取什么同步机制

我们回顾下创建 AudioTrack 对象时，FIFO 及其控制块的结构如下所示：

*   MODE_STREAM 模式下的匿名共享内存结构：

```
|                                                         |
  | -------------------> mCblkMemory <--------------------- |
  |                                                         |
  +--------------------+------------------------------------+
  | audio_track_cblk_t |               FIFO                 |
  +--------------------+------------------------------------+
  ^                    ^
  |                    |
mCblk               mBuffer

mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer());
new(mCblk) audio_track_cblk_t();
mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
```

*   MODE_STATIC 模式下的匿名共享内存结构：

```
+--------------------+    +-----------------------------------+
  | audio_track_cblk_t |    |         FIFO (sharedBuffer)       |
  +--------------------+    +-----------------------------------+
  ^                         ^
  |                         |
mCblk                    mBuffer

mCblk = (audio_track_cblk_t *) new uint8_t[size];
new(mCblk) audio_track_cblk_t();
mBuffer = sharedBuffer->pointer()
```

FIFO 管理相关的类图：

![](https://img-blog.csdn.net/20170301225908508?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvenl1YW55dW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

*   **AudioTrackClientProxy**：MODE_STREAM 模式下，生产者 AudioTrack 使用它在 FIFO 中找到可用空间的位置
*   **AudioTrackServerProxy**：MODE_STREAM 模式下，消费者 AudioFlinger::PlaybackThread 使用它在 FIFO 中找到可读数据的位置
*   **StaticAudioTrackClientProxy**：MODE_STATIC 模式下，生产者 AudioTrack 使用它在 FIFO 中找到可用空间的位置
*   **StaticAudioTrackServerProxy**：MODE_STATIC 模式下，消费者 AudioFlinger::PlaybackThread 使用它在 FIFO 中找到可读数据的位置
*   **AudioRecordClientProxy**：消费者 AudioRecord 使用它在 FIFO 中找到可读数据的位置
*   **AudioTrackServerProxy**：生产者 AudioFlinger::RecordThread 使用它在 FIFO 中找到可用空间的位置

到这里，我决定结束本文了。环形 FIFO 管理是 Android 音频系统的精髓，一个小节并不足以描述其原理及实现细节；Android 环形 FIFO 的实现可说得上精妙绝伦，其他项目如果要用到环形 FIFO，不妨多借鉴它。因此我想另写一篇博文详细分析 Android 环形 FIFO 的原理及实现，初定提纲如下，以作备忘：

1.  传统环形 FIFO 的原理
2.  Android 环形 FIFO 的原理
3.  读写指针的线程安全
4.  Futex 进程同步机制
5.  Android 环形 FIFO 的实现