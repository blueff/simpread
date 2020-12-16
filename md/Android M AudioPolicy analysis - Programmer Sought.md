> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.programmersought.com](https://www.programmersought.com/article/98514628191/)

AudioPolicy is mainly responsible for Audio "policy" related issues in the Android system. It and AudioFlinger together form two services of the Android Audio system. One is responsible for managing audio "routes", and one is responsible for managing audio "devices". In the Android M version of the system, these two services are loaded through MediaServer during system startup.

AudioPolicyService mainly completes the following tasks in the Android Audio system:

①Manage input and output devices, including device connection and disconnection status, device selection and switching, etc.

②Manage the audio strategy of the system, such as playing music during a call, or a series of processing of incoming calls when playing music

③The volume of the management system

④Some audio parameters of the upper layer can also be set to the bottom layer through AudioPolicyService

![](https://www.programmersought.com/images/425/a9cab9c93d612443fe9831e93e21edb1.png)

The AudioPolicyService service runs in the mediaserver process and starts when the mediaserver process starts.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
// frameworks/av/media/mediaserver/main_mediaserver.cpp
int main(int argc __unused, char** argv)
{
    ......
    ......
    AudioFlinger::instantiate();
    MediaPlayerService::instantiate();
    AudioPolicyService::instantiate();
    ......
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

AudioFlinger::instantiate() does not belong to the internal class of AudioFlinger, but an implementation of the BinderService class. Several services including AudioFlinger, AudioPolicy, etc. are inherited from this unified Binder service class, which is implemented in BinderService.h

```
// frameworks/native/include/binder/BinderService.h
static void instantiate() { publish(); }
```

There is only one line of code, that is, it calls its own publish() function, let’s analyze this publish() function

```
static status_t publish(bool allowIsolated = false) 
{
    sp<IServiceManager> sm(defaultServiceManager());
    return sm->addService( String16(SERVICE::getServiceName()),new SERVICE(), allowIsolated);
}
```

SERVICE is a template defined in the file. AudioPolicyService calls the instantiate() function, so the current SERVICE is AudioPolicyService

```
//  frameworks/native/include/binder/BinderService.h
template<typename SERVICE>
```

It can be seen that what the publish() function does gets the proxy of the ServiceManager, and then nwe an object of the service that calls instantiate and adds it to the ServiceManager.

So the next step is to analyze the constructor of AudioPolicyService

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
// frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
AudioPolicyService::AudioPolicyService() : BnAudioPolicyService(), 
                                    mpAudioPolicy(NULL),
                                    mAudioPolicyManager(NULL),   
                                    mAudioPolicyClient(NULL), 
                                    mPhoneState(AUDIO_MODE_INVALID)
{
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

Then I found that its constructor does not seem to do anything other than initialize some variables. In this case, where is the initialization of AudioPolicyService performed? Continue to analyze the above constructor. AudioPolicyService is Inherited from BnAudioPolicyService, push it up step by step, and finally found that its ancestor is RefBase. According to the characteristics of strong pointers, the target object will call onFirstRef() when it is referenced for the first time. Now that the goal is clear, let’s see Check AudioPolicyService::onFirstRef() if there is any information we care about.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
// frameworks/av/services/audiopolicy/service/AudioPolicyService.cpp
void AudioPolicyService::onFirstRef()
{
    ......
    //Used to play tone
    mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
    //Used to execute audio commands
    mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
    //Used to execute output commands
    mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);
    #ifdef USE_LEGACY_AUDIO_POLICY
    //Because USE_LEGACY_AUDIO_POLICY is undefined in the current source tree
    //So this is not true, go #else
    ......
    #else
    ALOGI("AudioPolicyService CSTOR in new mode");
    mAudioPolicyClient = new AudioPolicyClient(this);                                    
    mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
    #endif
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

It can be seen that AudioPolicyService created 3 AudioCommandThread and AudioPolicyManager when it was strongly referenced for the first time.

Take a look at what has been done after creating 3 AudioCommandThreads. First, it directly new an AudioPolicyClient, and the AudioPolicyClient class is defined in AudioPolicyService.h

```
//frameworks/av/services/audiopolicy/service/AudioPolicyService.h
class AudioPolicyClient : public AudioPolicyClientInterface
{
    ......
}
```

His implementation is in frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp. After the AudioPolicyClient is created, an AudioPolicyManager object is created by calling the createAudioPolicyManager method. Let's take a look at how the AudioPolicyManager is created in the createAudioPolicyManager method.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//frameworks/av/services/audiopolicy/manager/AudioPolicyFactory.cpp

extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

It can be seen that he directly new an AudioPolicyManager and then pass in the AudioPolicyClient just created.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//  frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    ......
    mEngine->setObserver(this);
    ......
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

mEngine->setObserver(this) binds mApmObserver to AudioPolicyManager object, so calling mApmObserver->xxx() in frameworks/av/services/audiopolicy/enginedefault/Engine.cpp is a member function of the called AudioPolicyManager class. To use the member functions in the AudioPOlicyManager class in the Engine class, use mApmObserver->xxx(),

To use the member functions of the Engine class in the AudioPolicyManager class, use mEngine->xxx()

Most of the functions in AudioPolicyService, such as audio policy management, input and output device management, device switching and volume adjustment, are implemented through AudioPolicyManager. AudioPolicyManager is passed by AudioPolicyService

```
mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
```

Created, take a look at the process of creating AudioPolicyManager by createAudioPolicyManager

```
// frameworks/av/services/audiopolicy/manager/AudioPolicyFactory.cpp
extern "C" AudioPolicyInterface* createAudioPolicyManager(
        AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManager(clientInterface);
}
```

It is created by directly new AudioPolicyManager, so we can directly analyze the constructor of AudioPolicyManager.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//frameworks/av/services/audiopolicy/managerdefault/AudioPolicyManager.cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    //1. Load the audio_policy.conf configuration file /vendor/etc/audio_policy.conf
    ConfigParsingUtils::loadAudioPolicyConfig(....);
    //2. Initialize the volume adjustment points corresponding to various audio streams
    mEngine->initializeVolumeCurves(mSpeakerDrcEnabled);
    //3. Load the audio policy hardware abstraction library
    mHwModules[i]->mHandle =  mpClientInterface->loadHwModule(mHwModules[i]->mName);
    //4. Turn on the output device
    mpClientInterface->openOutput(....);
    //5. Save the output device descriptor object
    addOutput(output, outputDesc);
    //6. Set the output device
    setOutputDevice(....);
    //7. Update the output device
    updateDevicesAndOutputs();
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

The general process is as follows

![](https://www.programmersought.com/images/85/a857cb8d6ef9cd8b336c50a1d3d23ced.JPEG)

 1. Load the audio_policy.conf configuration file /system/etc/audio_policy.conf (for the emulator, defaultAudioPolicyConfig())

During the creation of AudioPolicyManager, audio devices are loaded by loading the audio_policy.conf configuration file. Android defines a corresponding hardware abstraction layer for each audio interface. Hardware abstraction layer code reference writing

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

Each audio interface defines different input and output. An interface can have multiple inputs or outputs. Each input and output can support different devices. The audio interface parameters supported by the system can be obtained by reading the audio_policy.conf file. In AudioPolicyManager The /vendor/etc/audio_policy.conf configuration file will be loaded first. If the configuration file does not exist, the /system/etc/audio_policy.conf configuration file will be loaded. After AudioPolicyManager loads all audio interfaces, it knows all the audio interface parameters supported by the system and can provide decisions for audio output.

Audio_policy.conf defines multiple audio interfaces at the same time.Each audio interface contains several outputs and inputs, and each output and input supports multiple input and output modes at the same time, and each input and output mode supports several devices.

![](https://www.programmersought.com/images/524/492d766bfecf2f04adf8fc94e5711d34.JPEG)

```
ConfigParsingUtils::loadAudioPolicyConfig(....);
```

Divided into two parts, the first part is to parse the global tags, and the second part is to parse the audio_hw_modules tag. Its sub-tags all indicate hardware module. Both the primary and r_submix hardware modules are parsed to mHwModules. The sub-tags of the hardware module are outputs and inputs. , Each subtag of outputs is parsed to mOutputProfiles of mHwModules, and each subtag of inputs is parsed to mInputProfiles of mHwModules.

2. Initialize the volume adjustment points corresponding to various audio streams

```
mEngine->initializeVolumeCurves(mSpeakerDrcEnabled);
```

3. Load the audio policy hardware abstraction library

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);

//frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp
audio_module_handle_t AudioPolicyService::AudioPolicyClient::loadHwModule(const char *name)
{
    ALOGD("tian--test");
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return 0;
    }
    return af->loadHwModule(name);
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

AudioFlinger::loadHwModule() is called directly here. Let's enter AudioFlinger.cpp to see what it did

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//  When AudioPolicyManager is constructed, it will read the manufacturer’s audio description file audio_policy.conf
//Then open the audio interface based on this, and this process will eventually call AudioFlinger::loadHwModule
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (name == NULL) {
        return 0;
    }
    if (!settingsAllowed()) {
        return 0;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

Finally, the loadHwModule_l() function is called, continue to check

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//name value:
static const char * const audio_interfaces[] = {
    AUDIO_HARDWARE_MODULE_ID_PRIMARY,//Primary audio device, must exist
    AUDIO_HARDWARE_MODULE_ID_A2DP,//Bluetooth A2DP
    AUDIO_HARDWARE_MODULE_ID_USB,//USB audio
};

audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    //1. Has this interface been loaded?
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    //2. Load audio interface
audio_hw_device_t *dev;
int rc = load_audio_interface(name, &dev);
......
//3. Initialization
rc = dev->init_check(dev);
//4. Add to global variables
audio_module_handle_t handle = nextUniqueId();
mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

![](https://www.programmersought.com/images/732/bd6c84027917d1036a0d7154ef1167b4.png)

The second step is to load the specified audio interface such as primary, a2dp or usb function load_audio_interface()

Used to load the library files required by the device, then open the device and create an audio_hw_device_t instance. The library file name corresponding to the audio device interface has a certain format. For example, the module name of a2dp may be audio.a2dp.so or audio.a2dp .default.so, etc., there are two main search paths /system/lib/hw and /vendor/lib/hw, let's see how it is implemented

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//frameworks/av/services/audioflinger/AudioFlinger.cpp

static int load_audio_interface(const char *if_name, audio_hw_device_t **dev)
{
......
//1. Get the audio module              //Is defined as audio
rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
//2. Turn on the audio device
rc = audio_hw_device_open(mod, dev);
......
return rc;
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
    int i = 0;
    char prop[PATH_MAX] = {0};
    char path[PATH_MAX] = {0};
    char name[PATH_MAX] = {0};
    char prop_name[PATH_MAX] = {0};

    //Concatenated string
    if (inst)
        snprintf(name, PATH_MAX, "%s.%s", class_id, inst);
    else
        strlcpy(name, class_id, PATH_MAX);

    /*
     * Here we rely on the fact that calling dlopen multiple times on
     * the same .so will simply increment a refcount (and not load
     * a new copy of the library).
     * We also assume that dlopen() is thread-safe.
     */
     //Go to the prop system to find
    /* First try a property specific to the class and possibly instance */
    snprintf(prop_name, sizeof(prop_name), "ro.hardware.%s", name);
    if (property_get(prop_name, prop, NULL) > 0) {
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT; i++) {
        if (property_get(variant_keys[i], prop, NULL) == 0) {
            continue;
        }
        if (hw_module_exists(path, sizeof(path), name, prop) == 0) {
            goto found;
        }
    }

    /* Nothing found, try the default */
    if (hw_module_exists(path, sizeof(path), name, "default") == 0) {
        goto found;
    }

    return -ENOENT;

found:
    /* load the module, if this fails, we're doomed, and we should not try
     * to load a different variant. */
     //load
    return load(class_id, path, module);
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

The string path is spliced ​​into /system/lib/hw/audio.xxx.so and then the load function laod function is called through handle = dlopen(path, RTLD_NOW); /system/lib/hw/audio.xxx.so is opened and returned A handle.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//hardware/libhardware/hardware.c
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    ….
    //Open the dynamic library through the dlopen() method
    handle = dlopen(path, RTLD_NOW);
//Load hmi in the dynamic library through dlsym
……
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym)
    ….
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

Turn on the output device

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

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

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

AudioFlinger:: openOutput(.....) is called here.

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    ALOGI("openOutput(), module %d Device %x, SamplingRate %d, Format %#08x, Channels %x, flags %x",
              module,
              (devices != NULL) ? *devices : 0,
              config->sample_rate,
              config->format,
              config->channel_mask,
              flags);

    if (*devices == AUDIO_DEVICE_NONE) {
        return BAD_VALUE;
    }

    Mutex::Autolock _l(mLock);
   // ALOGD("tian-openOutput");
    sp<PlaybackThread> thread = openOutput_l(module, output, config, *devices, address, flags);
    if (thread != 0) {
        *latencyMs = thread->latency();

        // notify client processes of the new output creation
        thread->ioConfigChanged(AUDIO_OUTPUT_OPENED);

        // the first primary output opened designates the primary hw device
        if ((mPrimaryHardwareDev == NULL) && (flags & AUDIO_OUTPUT_FLAG_PRIMARY)) {
            ALOGI("Using module %d has the primary audio interface", module);
            mPrimaryHardwareDev = thread->getOutput()->audioHwDev;

            AutoMutex lock(mHardwareLock);
            mHardwareStatus = AUDIO_HW_SET_MODE;
            mPrimaryHardwareDev->hwDevice()->set_mode(mPrimaryHardwareDev->hwDevice(), mMode);
            mHardwareStatus = AUDIO_HW_IDLE;
        }
        return NO_ERROR;
    }

    return NO_INIT;
}
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

Let's take a look at the function openOutput_l()

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)

```
//The module in the input parameter is obtained by the previous loadNodule, which is the id number of an audio interface
//You can find the corresponding AudioHwDevice object in mAudioHwSevs through this id
//This method will add the opened output to the mPlaybackThreads thread
sp<AudioFlinger::PlaybackThread> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    //1. Find the corresponding audio interface
AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
if (outHwDev == NULL) {
        return 0;
    }

    audio_hw_device_t *hwDevHal = outHwDev->hwDevice();
    if (*output == AUDIO_IO_HANDLE_NONE) {
        *output = nextUniqueId();
    }
//2. Open an output stream for the device and create an audio output object of Audio HAL
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());
//3. Create playbackThread
    if (status == NO_ERROR) {
        PlaybackThread *thread;
        if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            ALOGD("tian-OffloadThread");
          //  thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created offload output: ID %d thread %p", *output, thread);
            
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                || !isValidPcmSinkFormat(config->format)
                || !isValidPcmSinkChannelMask(config->channel_mask)) {
                ALOGD("tian-DirectOutputThread");
            thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created direct output: ID %d thread %p", *output, thread);
        } else {
            //Generally, a mixing thread is created, and the output representing the AudioStreamOut object is also passed in
            thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
            ALOGV("openOutput_l() created mixer output: ID %d thread %p", *output, thread);
        }
        mPlaybackThreads.add(*output, thread);//Add playback thread
        return thread;
    }
```

![](https://www.programmersought.com/images/134/69c5a8ac3fa60e0848d784a6dd461da6.gif)