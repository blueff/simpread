> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/Qidi_Huang/article/details/56834419)

Android 7.0 Audio 的 Resample 过程详解
=================================

Qidi 2017.02.23 （Markdown & Haroopad）

##### 【前言】

处理过音频文件的工程师都知道音频数据存在采样率（Sample Rate）这个指标。在位深度（Bit Depth）一定的情况下，采样率越高，理论上来说播放出来的声音就越细腻，录制的声音也就越保真，反之亦然。

但在较早的 Android 系统版本上，不管音频文件原来的采样率几何，统统都被重采样（Resample）到 44.1KHz 进行播放，录制的时候则是被固定为 8KHz 进行采样。尽管这样的处理方式被广大音质爱好者所诟病，但在当时它确实是一种实现设备兼容的有效方法。

作为 Android Audio BSP 工程师，有必要了解系统实现 Resample 的过程。现在 Android 系统已经发布到了 7.0 版本，一起看看在最新的版本上这个 Resample 的过程是怎样实现的吧。

##### 【背景知识】

我们知道在 Android 系统中，当应用层 APP 播放一个音频文件时，Framework 层的 **AudioPolicyService（APS）**会接收上层 APP 传递来的音频参数（例如格式、声道、采样率等），并调用 **AudioFlinger** 的`createTrack()`方法对应创建 1 个 Track，再调用`openOutput()`方法来打开 1 个 outputStream，然后使用这个 outputStream 来创建相应的 Playback 线程（依据应用场景可以是 OffloadThread、DirectOutputThread、MixerThread），最终在 Playback 线程中匹配之前创建的 Track，开始自 APP 至 HAL 的数据传输。

##### 【Resample 过程分析】

那么我们对 Android Audio Resample 过程的分析就从 AudioFlinger 开始。在`AudioFlinger::openOutput()`中可以看到，在 Playback 线程被成功创建之后，即被加入到 mPlaybackThreads 向量中进行管理了。具体代码如下：

```
sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    ......

    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(    // 打开1个outputStream
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
            thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created offload output: ID %d thread %p", *output, thread);
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                || !isValidPcmSinkFormat(config->format)
                || !isValidPcmSinkChannelMask(config->channel_mask)) {
            thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created direct output: ID %d thread %p", *output, thread);
        } else {
            thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);    // 默认情况下，创建MixerThread类型的Playback线程
            ALOGV("openOutput_l() created mixer output: ID %d thread %p", *output, thread);
        }
        mPlaybackThreads.add(*output, thread);    // 将新创建的线程加入向量
        return thread;
    }

    return 0;
}
```

随后 Playback 线程运行，对应的`AudioFlinger::Playback::threadLoop()`方法被执行，在该方法中调用了`prepareTracks_l()`函数。这个函数实际上是对应于`AudioFlinger::MixerThread::prepareTracks_l()`这个函数。threadLoop() 函数代码细节如下：

```
bool AudioFlinger::PlaybackThread::threadLoop()
{
    Vector< sp<Track> > tracksToRemove;
    ......

    while (!exitPending())
    {
        cpuStats.sample(myName);
        Vector< sp<EffectChain> > effectChains;
        { // scope for mLock
            Mutex::Autolock _l(mLock);
            processConfigEvents_l();
            ......
            saveOutputTracks();
            ......
            if ((!mActiveTracks.size() && systemTime() > mStandbyTimeNs) ||
                                   isSuspended()) {
                // put audio hardware into standby after short delay
                if (shouldStandby_l()) {

                    threadLoop_standby();

                    mStandby = true;
                }
                ......
            }
            // mMixerStatusIgnoringFastTracks is also updated internally
            mMixerStatus = prepareTracks_l(&tracksToRemove);    // 调用prepareTracks_l()，为Playback线程匹配已注册的Track
            ......
            // prevent any changes in effect chain list and in each effect chain
            // during mixing and effect process as the audio buffers could be deleted
            // or modified if an effect is created or deleted
            lockEffectChains_l(effectChains);
        } // mLock scope ends
        ......
        // enable changes in effect chain
        unlockEffectChains(effectChains);
        // Finally let go of removed track(s), without the lock held
        // since we can't guarantee the destructors won't acquire that
        // same lock.  This will also mutate and push a new fast mixer state.
        threadLoop_removeTracks(tracksToRemove);
        tracksToRemove.clear();
        // FIXME I don't understand the need for this here;
        //       it was in the original code but maybe the
        //       assignment in saveOutputTracks() makes this unnecessary?
        clearOutputTracks();
        // Effect chains will be actually deleted here if they were removed from
        // mEffectChains list during mixing or effects processing
        effectChains.clear();
        // FIXME Note that the above .clear() is no longer necessary since effectChains
        // is now local to this block, but will keep it for now (at least until merge done).
    }
    threadLoop_exit();
    if (!mStandby) {
        threadLoop_standby();
        mStandby = true;
    }
    ......
    return false;
}
```

Resample 的过程就发生在`prepareTracks_l()`函数中，所以我们来好好阅读一下。在该函数中，通过一个 for 循环遍历所有处于 active 状态的 track。每一次循环中，都要进行如下 2 步操作：  
1. 通过`reqSampleRate = track->mAudioTrackServerProxy->getSampleRate()`来获取硬件设备所支持的采样率；  
2. 之后调用`mAudioMixer->setParameter(name, AudioMixer::RESAMPLE, AudioMixer::SAMPLE_RATE, (void*)(uintptr_t)reqSampleRate)`，通过对比音频文件采样率和音频设备支持的采样率，判断是否创建新的 Resampler 对象，或者从已有的 Resampler 对象列表中返回 1 个；

prepareTracks_l() 函数代码细节如下：

```
AudioFlinger::PlaybackThread::mixer_state AudioFlinger::MixerThread::prepareTracks_l(
        Vector< sp<Track> > *tracksToRemove)
{
    ......
    // find out which tracks need to be processed
    size_t count = mActiveTracks.size();    // 获取处于active状态的track的数量
    ......
    for (size_t i=0 ; i<count ; i++) {
        const sp<Track> t = mActiveTracks[i].promote();
        if (t == 0) {
            continue;
        }
        // this const just means the local variable doesn't change
        Track* const track = t.get();    // 获取对应的track
        ......
        audio_track_cblk_t* cblk = track->cblk();

        // The first time a track is added we wait
        // for all its buffers to be filled before processing it
        int name = track->name();
        ......
        if ((framesReady >= minFrames) && track->isReady() &&
                !track->isPaused() && !track->isTerminated())
        {
            ......
            int param = AudioMixer::VOLUME;
            if (track->mFillingUpStatus == Track::FS_FILLED) {
                // no ramp for the first volume setting
                track->mFillingUpStatus = Track::FS_ACTIVE;
                if (track->mState == TrackBase::RESUMING) {
                    track->mState = TrackBase::ACTIVE;
                    param = AudioMixer::RAMP_VOLUME;
                }
                mAudioMixer->setParameter(name, AudioMixer::RESAMPLE, AudioMixer::RESET, NULL);
            // FIXME should not make a decision based on mServer
            } else if (cblk->mServer != 0) {
                // If the track is stopped before the first frame was mixed,
                // do not apply ramp
                param = AudioMixer::RAMP_VOLUME;
            }

            // compute volume for this track
            ......
            // Delegate volume control to effect in track effect chain if needed
            ......

            // XXX: these things DON'T need to be done each time
            mAudioMixer->setBufferProvider(name, track);
            mAudioMixer->enable(name);

            mAudioMixer->setParameter(name, param, AudioMixer::VOLUME0, &vlf);    // 设置左声道音量
            mAudioMixer->setParameter(name, param, AudioMixer::VOLUME1, &vrf);    // 设置右声道音量
            mAudioMixer->setParameter(name, param, AudioMixer::AUXLEVEL, &vaf);   // 设置辅助声道音量
            mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::FORMAT, (void *)track->format());    // 设置音频数据格式
            mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::CHANNEL_MASK, (void *)(uintptr_t)track->channelMask());    // 设置音频声道数
            mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::MIXER_CHANNEL_MASK, (void *)(uintptr_t)mChannelMask);
            // limit track sample rate to 2 x output sample rate, which changes at re-configuration
            uint32_t maxSampleRate = mSampleRate * AUDIO_RESAMPLER_DOWN_RATIO_MAX;
            uint32_t reqSampleRate = track->mAudioTrackServerProxy->getSampleRate();    // 获取音频设备所支持的采样率
            if (reqSampleRate == 0) {
                reqSampleRate = mSampleRate;
            } else if (reqSampleRate > maxSampleRate) {
                reqSampleRate = maxSampleRate;
            }
            mAudioMixer->setParameter(
                name,
                AudioMixer::RESAMPLE,
                AudioMixer::SAMPLE_RATE,    // 设置音频采样率(必要时会进行重采样)
                (void *)(uintptr_t)reqSampleRate);

            AudioPlaybackRate playbackRate = track->mAudioTrackServerProxy->getPlaybackRate();
            mAudioMixer->setParameter(
                name,
                AudioMixer::TIMESTRETCH,
                AudioMixer::PLAYBACK_RATE,    // 设置播放码率
                &playbackRate);

            /*
             * Select the appropriate output buffer for the track.
             *
             * Tracks with effects go into their own effects chain buffer
             * and from there into either mEffectBuffer or mSinkBuffer.
             *
             * Other tracks can use mMixerBuffer for higher precision
             * channel accumulation.  If this buffer is enabled
             * (mMixerBufferEnabled true), then selected tracks will accumulate
             * into it.
             *
             */
            if (mMixerBufferEnabled
                    && (track->mainBuffer() == mSinkBuffer
                            || track->mainBuffer() == mMixerBuffer)) {
                mAudioMixer->setParameter(
                        name,
                        AudioMixer::TRACK,
                        AudioMixer::MIXER_FORMAT, (void *)mMixerBufferFormat);    // 设置缓冲区数据格式
                mAudioMixer->setParameter(
                        name,
                        AudioMixer::TRACK,
                        AudioMixer::MAIN_BUFFER, (void *)mMixerBuffer);    // 分配主缓冲区
                // TODO: override track->mainBuffer()?
                mMixerBufferValid = true;
            } else {
                ......
            }
            mAudioMixer->setParameter(
                name,
                AudioMixer::TRACK,
                AudioMixer::AUX_BUFFER, (void *)track->auxBuffer());    // 分配副缓冲区

            // reset retry count
            track->mRetryCount = kMaxTrackRetries;

            // If one track is ready, set the mixer ready if:
            //  - the mixer was not ready during previous round OR
            //  - no other track is not ready
            if (mMixerStatusIgnoringFastTracks != MIXER_TRACKS_READY ||
                    mixerStatus != MIXER_TRACKS_ENABLED) {
                mixerStatus = MIXER_TRACKS_READY;
            }
        } else {
            // 出现underrun，以及相应处理操作
            ......
        }
    }

    // Push the new FastMixer state if necessary
    ......
    // Now perform the deferred reset on fast tracks that have stopped
    ......
    // remove all the tracks that need to be...
    removeTracks_l(*tracksToRemove);
    ......
    // sink or mix buffer must be cleared if all tracks are connected to an
    // effect chain as in this case the mixer will not write to the sink or mix buffer
    // and track effects will accumulate into it
    ......
    // if any fast tracks, then status is ready
    ......
    return mixerStatus;
}
```

在确认要使用的 Resampler 对象存在后，调用`invalidateState(1 << name)`使设置生效，开始执行重采样。invalidateState() 函数会调用`AudioMixer::process_validate()`，在该函数中首先通过语句`t.hook = getTrackHook(TRACKTYPE_RESAMPLE, t.mMixerChannelCount, t.mMixerInFormat, t.mMixerFormat);`获取执行重采样操作的函数，随后通过`state->hook = process_resampling;`中的`t.hook(&t, outTemp, numFrames, state->resampleTemp, aux)`语句进行调用。  
setParameter() 函数代码如下：

```
void AudioMixer::setParameter(int name, int target, int param, void *value)
{
    ......
    int valueInt = static_cast<int>(reinterpret_cast<uintptr_t>(value));
    int32_t *valueBuf = reinterpret_cast<int32_t*>(value);
    switch (target) {
    ......
    case RESAMPLE:
        switch (param) {
        case SAMPLE_RATE:
            ALOG_ASSERT(valueInt > 0, "bad sample rate %d", valueInt);
            if (track.setResampler(uint32_t(valueInt), mSampleRate)) {    // 新建或查找1个Resampler对象
                ALOGV("setParameter(RESAMPLE, SAMPLE_RATE, %u)",
                        uint32_t(valueInt));
                invalidateState(1 << name);    // 使设置生效，调用重采样的后续处理函数
            }
            break;
        case RESET:
            track.resetResampler();
            invalidateState(1 << name);
            break;
        case REMOVE:
            delete track.resampler;
            track.resampler = NULL;
            track.sampleRate = mSampleRate;
            invalidateState(1 << name);
            break;
        default:
            LOG_ALWAYS_FATAL("setParameter resample: bad param %d", param);
        }
        break;
    }
}
```

invalidateState() 函数代码如下：

```
void AudioMixer::invalidateState(uint32_t mask)
{
    if (mask != 0) {
        mState.needsChanged |= mask;
        mState.hook = process__validate;    // 使配置生效
    }
}
```

process__validate() 函数代码如下：

```
void AudioMixer::process__validate(state_t* state)
{
    ......
    uint32_t en = state->enabledTracks;
    while (en) {
        ......
        if (n & NEEDS_MUTE) {
            ......
        } else {
            ......
            if (n & NEEDS_RESAMPLE) {
                all16BitsStereoNoResample = false;
                resampling = true;
                t.hook = getTrackHook(TRACKTYPE_RESAMPLE, t.mMixerChannelCount,
                        t.mMixerInFormat, t.mMixerFormat);    // 获取Resample时track对象需要执行的函数（查看getTrackHook()可以看到被获取的函数是track__genericResample()）
                ALOGV_IF((n & NEEDS_CHANNEL_COUNT__MASK) > NEEDS_CHANNEL_2,
                        "Track %d needs downmix + resample", i);
            } else {
                ......
            }
        }
    }
    // select the processing hooks
    state->hook = process__nop;
    if (countActiveTracks > 0) {
        if (resampling) {
            if (!state->outputTemp) {
                state->outputTemp = new int32_t[MAX_NUM_CHANNELS * state->frameCount];
            }
            if (!state->resampleTemp) {
                state->resampleTemp = new int32_t[MAX_NUM_CHANNELS * state->frameCount];
            }
            state->hook = process__genericResampling;    // 在需要重采样操作的情况下，调用process_genericResampling()函数
        } else {
            ......
        }
    }
    ......
    // Now that the volume ramp has been done, set optimal state and
    // track hooks for subsequent mixer process
    ......
}
```

process_genericResampling() 函数代码如下：

```
// generic code with resampling
void AudioMixer::process__genericResampling(state_t* state)
{
    ......
    uint32_t e0 = state->enabledTracks;
    while (e0) {
        // process by group of tracks with same output buffer
        // to optimize cache use
        ......
        while (e1) {
            ......
            // this is a little goofy, on the resampling case we don't
            // acquire/release the buffers because it's done by
            // the resampler.
            if (t.needs & NEEDS_RESAMPLE) {
                t.hook(&t, outTemp, numFrames, state->resampleTemp, aux);    // 调用track__genericResample()函数执行Resample
            } else {
                ......
            }
        }
        convertMixerFormat(out, t1.mMixerFormat,
                outTemp, t1.mMixerInFormat, numFrames * t1.mMixerChannelCount);
    }
}
```

至此，Android 系统播放音频时的 Resample 过程就分析完成了。

具体的 Resample 处理实质是数字信号处理，是个数学运算过程。Android 系统中提供的算法有**线性插值**、**三次插值**、**FIR 滤波** 3 种。感兴趣的工程师同仁可以自行查阅相关资料书籍，这里不对数字信号处理的细节进行讨论。