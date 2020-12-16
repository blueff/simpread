> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/WAN_EXE/article/details/79167423)

        上篇文章中只是讲到配置文件在哪里，怎么读取文件，但是真正的文件解析还没有开始。这篇就开始分析文件解析，认识这些解析的参数。

 接着上篇 [Audio 设备文件加载](http://blog.csdn.net/wan_exe/article/details/79164135)继续进行分析，知道主要是这个函数起到解析的作用。

```
static status_t deserializeAudioPolicyXmlConfig(AudioPolicyConfig &config) {
    char audioPolicyXmlConfigFile[AUDIO_POLICY_XML_CONFIG_FILE_PATH_MAX_LENGTH];
    status_t ret;
 
    for (int i = 0; i < kConfigLocationListSize; i++) {
        PolicySerializer serializer;
        snprintf(audioPolicyXmlConfigFile,
                 sizeof(audioPolicyXmlConfigFile),
                 "%s/%s",
                 kConfigLocationList[i],
                 AUDIO_POLICY_XML_CONFIG_FILE_NAME);
        ret = serializer.deserialize(audioPolicyXmlConfigFile, config);
        if (ret == NO_ERROR) {
            break;
        }
    }
    return ret;
}
```

也知道 audioPolicyXmlConfigFile 的配置文件是 / system/etc/auido_policy_configuration.xml 文件。传入的参数是 AudioPolicyConfig，这个类包含很多变量，

```
private:
    HwModuleCollection &mHwModules; /**< Collection of Module, with Profiles, i.e. Mix Ports. */
    DeviceVector &mAvailableOutputDevices;
    DeviceVector &mAvailableInputDevices;
    sp<DeviceDescriptor> &mDefaultOutputDevices;
    VolumeCurvesCollection *mVolumeCurves;
    bool &mIsSpeakerDrcEnabled;
```

也就是需要传递进来进行初始化的变量，解析的作用就是初始化这五个变量。

函数里面有个 PolicySerializer 类

/[frameworks](http://androidxref.com/8.0.0_r4/xref/frameworks/)/[av](http://androidxref.com/8.0.0_r4/xref/frameworks/av/)/[services](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/)/[audiopolicy](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/audiopolicy/)/[common](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/audiopolicy/common/)/[managerdefinitions](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/audiopolicy/common/managerdefinitions/)/[include](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/audiopolicy/common/managerdefinitions/include/)/[Serializer.h](http://androidxref.com/8.0.0_r4/xref/frameworks/av/services/audiopolicy/common/managerdefinitions/include/Serializer.h)

只有这个函数有作用

```
status_t PolicySerializer::deserialize(const char *configFile, AudioPolicyConfig &config)
{
    xmlDocPtr doc;
    doc = xmlParseFile(configFile);
    if (doc == NULL) {
        ALOGE("%s: Could not parse %s document.", __FUNCTION__, configFile);
        return BAD_VALUE;
    }
    xmlNodePtr cur = xmlDocGetRootElement(doc);
    if (cur == NULL) {
        ALOGE("%s: Could not parse %s document: empty.", __FUNCTION__, configFile);
        xmlFreeDoc(doc);
        return BAD_VALUE;
    }
    if (xmlXIncludeProcess(doc) < 0) {
         ALOGE("%s: libxml failed to resolve XIncludes on %s document.", __FUNCTION__, configFile);
    }

    if (xmlStrcmp(cur->name, (const xmlChar *) mRootElementName.c_str()))  {
        ALOGE("%s: No %s root element found in xml data %s.", __FUNCTION__, mRootElementName.c_str(),
              (const char *)cur->name);
        xmlFreeDoc(doc);
        return BAD_VALUE;
    }
 
    string version = getXmlAttribute(cur, versionAttribute);
    if (version.empty()) {
        ALOGE("%s: No version found in root node %s", __FUNCTION__, mRootElementName.c_str());
        return BAD_VALUE;
    }
    if (version != mVersion) {
        ALOGE("%s: Version does not match; expect %s got %s", __FUNCTION__, mVersion.c_str(),
              version.c_str());
        return BAD_VALUE;
    }
    // Lets deserialize children
    // Modules
    ModuleTraits::Collection modules;
    deserializeCollection<ModuleTraits>(doc, cur, modules, &config);
    config.setHwModules(modules);
 
    // deserialize volume section
    VolumeTraits::Collection volumes;
    deserializeCollection<VolumeTraits>(doc, cur, volumes, &config);
    config.setVolumes(volumes);
 
    // Global Configuration
    GlobalConfigTraits::deserialize(cur, config);
 
    xmlFreeDoc(doc);
    return android::OK;
}
```

来看这个函数的实现

```
<?xml version="1.0" encoding="UTF-8"?>
<!-- A2dp Audio HAL Audio Policy Configuration file -->
<module >
    <mixPorts>
        <mixPort >
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
        <mixPort >
            <profile 
                     samplingRates="44100,48000"
                     channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO"/>
        </mixPort>
    </mixPorts>
    <devicePorts>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100,48000"
                     channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO"/>
        </devicePort>
    </devicePorts>
    <routes>
        <route type="mix" sink="BT A2DP Out"
               sources="a2dp output"/>
        <route type="mix" sink="BT A2DP Headphones"
               sources="a2dp output"/>
        <route type="mix" sink="BT A2DP Speaker"
               sources="a2dp output"/>
        <route type="mix" sink="a2dp input"
               sources="BT A2DP In"/>
    </routes>
</module>
```

doc 是文件节点，cur 是文件根节点，可以参考 [libxml2 生成和解析配置树文件](http://blog.csdn.net/wan_exe/article/details/79177033)文章。这个函数会进一步解析包含的文件

```
<xi:include href="a2dp_audio_policy_configuration.xml"/>
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!-- A2dp Audio HAL Audio Policy Configuration file -->
<module >
    <mixPorts>
        <mixPort >
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </mixPort>
        <mixPort >
            <profile 
                     samplingRates="44100,48000"
                     channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO"/>
        </mixPort>
    </mixPorts>
    <devicePorts>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100"
                     channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
        </devicePort>
        <devicePort tag>
            <profile 
                     samplingRates="44100,48000"
                     channelMasks="AUDIO_CHANNEL_IN_MONO,AUDIO_CHANNEL_IN_STEREO"/>
        </devicePort>
    </devicePorts>
    <routes>
        <route type="mix" sink="BT A2DP Out"
               sources="a2dp output"/>
        <route type="mix" sink="BT A2DP Headphones"
               sources="a2dp output"/>
        <route type="mix" sink="BT A2DP Speaker"
               sources="a2dp output"/>
        <route type="mix" sink="a2dp input"
               sources="BT A2DP In"/>
    </routes>
</module>
```

可以看到 modules 的书写规则是需要包括 <mixPorts>,<devicePorts>,<routes > 几个节点的，解析完成之后会生成树。

下面这句话是获取 xml 版本号，值为 1.0

```
string version = getXmlAttribute(cur, versionAttribute);
```

可以看到 libxml2 已经做好解析的所有事情，接下来就是向参数赋值。

如果觉得这篇文章有用，可以扫免费红包支持。

![](https://img-blog.csdn.net/20180126130346875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvV0FOX0VYRQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)