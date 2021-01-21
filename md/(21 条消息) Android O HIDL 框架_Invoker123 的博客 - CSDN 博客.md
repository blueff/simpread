> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/Invoker123/article/details/84868260)

HIDL 简介
-------

  Android O 开始，Google 为了将 framework 和 HAL 层分割开来，使得 framework 可以独立于 HAL 层更新，设计了 HIDL。有了 HIDL，HAL 模块可以以一个独立的 Service 启动，framework 可以通过 Binder 来和 HAL 模块进行通信，厂家实现的 HAL 模块将会被放到设备的 / vendor 目录下。  
  HIDL 分为两种模式：Passthrough 模式和 Binderized 模式。Passthrough 模式是为了兼容 Android O 以前打开 HAL 的模式：首次调用该 HAL 接口时会调用 hw_get_module 打开 HAL 模块。但是 Passthrough 模式仅能用于 C++ 层，Java 层值能使用 Binderized 模式。而 Binderized 模式会启动一个 Service 进程，里面已经加载好 HAL 的 so 库，framework 直接通过 Binder 和该 HAL service 通信。

HIDL 用法
-------

  系统定义的所有的. hal 接口，都是通过 hidl-gen 工具转换成对应的代码。hidl-gen 源码路径：system/tools/hidl，是在 ubuntu 上可执行的二进制文件。  
  使用方法（可执行文件位于源码的 out/host/linux-x86/bin/hidl-gen）：`

```
usage: ./host/linux-x86/bin/hidl-gen [-p <root path>] -o <output path> -L <language> (-r <interface root>)+ [-t] fqname+
         -h: Prints this menu.
         -L <language>: The following options are available:
            check           : Parses the interface to see if valid but doesn't write any files.
            c++             : (internal) (deprecated) Generates C++ interface files for talking to HIDL interfaces.
            c++-headers     : (internal) Generates C++ headers for interface files for talking to HIDL interfaces.
            c++-sources     : (internal) Generates C++ sources for interface files for talking to HIDL interfaces.
            export-header   : Generates a header file from @export enumerations to help maintain legacy code.
            c++-impl        : Generates boilerplate implementation of a hidl interface in C++ (for convenience).
            c++-impl-headers: c++-impl but headers only
            c++-impl-sources: c++-impl but sources only
            java            : (internal) Generates Java library for talking to HIDL interfaces in Java.
            java-constants  : (internal) Like export-header but for Java (always created by -Lmakefile if @export exists).
            vts             : (internal) Generates vts proto files for use in vtsd.
            makefile        : (internal) Generates makefiles for -Ljava and -Ljava-constants.
            androidbp       : (internal) Generates Soong bp files for -Lc++-headers and -Lc++-sources.
            androidbp-impl  : Generates boilerplate bp files for implementation created with -Lc++-impl.
            hash            : Prints hashes of interface in `current.txt` format to standard out.
         -o <output path>: Location to output files.
         -p <root path>: Android build root, defaults to $ANDROID_BUILD_TOP or pwd.
         -r <package:path root>: E.g., android.hardware:hardware/interfaces.
         -t: generate build scripts (Android.bp) for tests.
```

BootControl 模块编译过程
------------------

  在 Android 的 frameworks/hardware（对应包前缀 android.frameworks._），hardware/interfaces（对应包前缀 android.hardware._），system/libhidl（对应包前缀 android.hidl._），system/hardware/interfaces（对应包前缀 android.system._）目录下均放置有 update-makefiles.sh 脚本，执行这个脚本就会在上述目录下找到 *.hal 文件的位置，自动生成对应的 mk（只在 hardware/interfaces 才生成）和 bp 文件。Google 设计使用 hidl-gen 工具编译 *.hal 文件来自动生成对应的 java，cpp，.h 等文件，通过 *.hal 来描述所有的 HIDL 接口和类型。在对应的 mk 和 bp 文件可以看到具体的 hidl-gen 生成规则和输出路径。本文将以 IBootControl.hal 为例讲 HIDL 的框架，不再讨论这个 HAL 的具体功能。使用 hidl-gen 工具后，在 out 目录生成的文件主要有 IBootControl.java（Java 层对 HIDL 的封装），BpHwBootControl.h（Binder Bp 端头文件），BnHwBootControl.h（Binder Bn 端头文件），BsBootControl.h（Passthrough 模式的封装），BootControlAll.cpp（包含 Binder Bp 端函数实现，Binder Bn 端函数实现，BsBootControl 部分函数实现还有 IBootControl 的 getService 实现等）。  
  可以看到 *.hal 文件的内容很简单，package 指明了这个 IBootControl.hal 文件所属的包，interface 声明了 IBootControl.hal 的接口。所有的 IXXX.hal 的接口均隐式继承自 IBase.hal 的接口，类比 Java 所有类都继承自 java.lang.Object。IBase.hal 包含了一些体现 HAL 接口共性的接口，类比 Java 所有的实例对象都能使用 Obejct 类的 notify() 方法。  
  IBootControl.hal 和 types.hal 编译自动生成出来的 cpp，.h 等文件会被放到 android.hardware.boot@1.0.so 里面，java 文件会被放到 android.hidl.base-V1.0-java.jar 动态库和 android.hardware.boot-V1.0-java-static 静态库里面。这些 so 和 jar 会被放在设备的 / system 下面，然而 default 目录下 service.cpp 则被编译成 android.hardware.boot@1.0-impl.so，跟 HAL 具体实现的 so 一起被放在 / vendor 下。Bootcontol 模块的目录结构如下：

```
├── 1.0
│   ├── Android.bp
│   ├── Android.mk
│   ├── default
│   │   ├── android.hardware.boot@1.0-service.rc
│   │   ├── Android.mk
│   │   ├── BootControl.cpp
│   │   ├── BootControl.h
│   │   └── service.cpp
│   ├── IBootControl.hal
│   ├── types.hal
│   └── vts
│       ├── Android.bp
│       ├── Android.mk
│       └── functional
│           ├── Android.bp
│           └── VtsHalBootV1_0TargetTest.cpp
└── Android.bp
```

_**hardware/interfaces/boot/1.0/IBootControl.hal**_

```
package android.hardware.boot@1.0;

/**
 * The Boot Control HAL is designed to allow for managing sets of redundant
 * partitions, called slots, that can be booted from independently. Slots
 * are sets of partitions whose names differ only by a given suffix.
 * They are identified here by a 0 indexed number and associated with their
 * suffix, which is appended to the base name for any particular partition
 * to find the one associated with that slot.
 * The primary use of this set up is to allow for background updates while
 * the device is running, and to provide a fallback in the event that the
 * update fails.
 */
interface IBootControl {
  /**
   * getNumberSlots() returns the number of available slots.
   * For instance, a system with a single set of partitions must return
   * 1, a system with A/B must return 2, A/B/C -> 3 and so on. A system with
   * less than two slots doesn't support background updates, for example if
   * running from a virtual machine with only one copy of each partition for the
   * purpose of testing.
   */
  getNumberSlots() generates (uint32_t numSlots);
  ...
  getSuffix(Slot slot) generates (string slotSuffix);
};
```

_**hardware/interfaces/boot/1.0/types.hal**_

```
package android.hardware.boot@1.0;

/**
 * A command result encapsulating whether the command succeeded and
 * an error string.
 */
struct CommandResult {
    bool success;
    string errMsg;
};

/**
 * An identifier for a slot number.
 */
typedef uint32_t Slot;

/**
 * A result encapsulating whether a function returned true, false or
 * failed due to an invalid slot number
 */
enum BoolResult : int32_t {
    FALSE = 0,
    TRUE = 1,
    INVALID_SLOT = -1
};
```

  再看看同级目录下的 Android.bp 和 Android.m k。  
  以 types.hal 和 IBootControl.hal 作为一个名为 "android.hardware.boot@1.0_hal" 的 filegroup，通过 genrule 定义的 hidl-gen 规则生成相应的文件。hidl-gen 的 - L 参数为 “c++source” 时，生成的是 cpp 文件；-L 参数为 “c+±headers” 时，生成的是头文件；-o 参数为 genDir 的变量，一般位于./soong/.intermediates 下面，根据编译出来的文件类型个模块所处的源码位置有所不同，例如 IBootControl.h 位于./soong/.intermediates/hardware/interfaces/boot/1.0/android.hardware.boot@1.0_genc++_headers/gen/android/hardware/boot/1.0/IBootControl.h，BootControlAll.cpp 位于./soong/.intermediates/hardware/interfaces/boot/1.0/android.hardware.boot@1.0_genc++/gen/android/hardware/boot/1.0/BootControlAll.cpp，“hardware/interfaces/boot/1.0”这部分为源码的位置，“android.hardware.boot@1.0_gencxxx”由模块的全限定名加生成文件类型确定，剩下的部分则可以参考 genrule 规则的 out 选项。  
  最后的 cc_library 选项将之前 genrule 产生的头文件和源文件编译成一个 so。  
  Android.m k 则用于编译出对应的 Java 动态库，静态库和 Java 文件，不再详细叙述。

_**hardware/interfaces/boot/1.0/Android.bp**_

```
// This file is autogenerated by hidl-gen. Do not edit manually.

filegroup {
    name: "android.hardware.boot@1.0_hal",
    srcs: [
        "types.hal",
        "IBootControl.hal",
    ],
}

genrule {
    name: "android.hardware.boot@1.0_genc++",
    tools: ["hidl-gen"],
    cmd: "$(location hidl-gen) -o $(genDir) -Lc++-sources -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport android.hardware.boot@1.0",
    srcs: [
        ":android.hardware.boot@1.0_hal",
    ],
    out: [
        "android/hardware/boot/1.0/types.cpp",
        "android/hardware/boot/1.0/BootControlAll.cpp",
    ],
}

genrule {
    name: "android.hardware.boot@1.0_genc++_headers",
    tools: ["hidl-gen"],
    cmd: "$(location hidl-gen) -o $(genDir) -Lc++-headers -randroid.hardware:hardware/interfaces -randroid.hidl:system/libhidl/transport android.hardware.boot@1.0",
    srcs: [
        ":android.hardware.boot@1.0_hal",
    ],
    out: [
        "android/hardware/boot/1.0/types.h",
        "android/hardware/boot/1.0/hwtypes.h",
        "android/hardware/boot/1.0/IBootControl.h",
        "android/hardware/boot/1.0/IHwBootControl.h",
        "android/hardware/boot/1.0/BnHwBootControl.h",
        "android/hardware/boot/1.0/BpHwBootControl.h",
        "android/hardware/boot/1.0/BsBootControl.h",
    ],
}

cc_library {
    name: "android.hardware.boot@1.0",
    defaults: ["hidl-module-defaults"],
    generated_sources: ["android.hardware.boot@1.0_genc++"],
    generated_headers: ["android.hardware.boot@1.0_genc++_headers"],
    export_generated_headers: ["android.hardware.boot@1.0_genc++_headers"],
    vendor_available: true,
    vndk: {
        enabled: true,
    },
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libhwbinder",
        "liblog",
        "libutils",
        "libcutils",
    ],
    export_shared_lib_headers: [
        "libhidlbase",
        "libhidltransport",
        "libhwbinder",
        "libutils",
    ],
}
```

  在 default 文件夹下，有一个 rc 文件，用来启动 HAL 服务进程的：

_**hardware/interfaces/boot/1.0/default/android.hardware.boot@1.0-service.rc**_

```
service boot-hal-1-0 /vendor/bin/hw/android.hardware.boot@1.0-service
    class early_hal
    user root
    group root
```

  再看看这个服务的来历。可以看到 service.cpp 被编译成 android.hardware.boot@1.0-service，其 main 函数也是该服务的入口。

_**hardware/interfaces/boot/1.0/default**_

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE := android.hardware.boot@1.0-impl
LOCAL_MODULE_RELATIVE_PATH := hw
LOCAL_PROPRIETARY_MODULE := true
LOCAL_SRC_FILES := \
    BootControl.cpp \

LOCAL_SHARED_LIBRARIES := \
    liblog \
    libhidlbase \
    libhidltransport \
    libhardware \
    libutils \
    android.hardware.boot@1.0 \

include $(BUILD_SHARED_LIBRARY)

include $(CLEAR_VARS)
LOCAL_MODULE_RELATIVE_PATH := hw
LOCAL_PROPRIETARY_MODULE := true
LOCAL_MODULE := android.hardware.boot@1.0-service
LOCAL_INIT_RC := android.hardware.boot@1.0-service.rc
LOCAL_SRC_FILES := \
    service.cpp

LOCAL_SHARED_LIBRARIES := \
    liblog \
    libhardware \
    libhidlbase \
    libhidltransport \
    libutils \
    android.hardware.boot@1.0 \

include $(BUILD_EXECUTABLE)
```

  启动服务：

_**hardware/interfaces/boot/1.0/default/service.cpp**_

```
int main (int /* argc */, char * /* argv */ []) {
    return defaultPassthroughServiceImplementation<IBootControl>();
}
```

  首先是调用 IBootControl::getService 获取服务，这时候会首次加载底层 HAL 的 so，然后调用 IBootControl::registerAsService 向 HwServiceManager 注册服务，这样，framework 就可以通过 Binder 跟 android.hardware.boot@1.0-service 这个厂商 Service 进程进行通信了。

_**system/libhidl/transport/include/hidl/LegacySupport.h**_

```
template<class Interface>
__attribute__((warn_unused_result))
status_t defaultPassthroughServiceImplementation(std::string name,
                                            size_t maxThreads = 1) {
    configureRpcThreadpool(maxThreads, true);
    status_t result = registerPassthroughServiceImplementation<Interface>(name);

    if (result != OK) {
        return result;
    }

    joinRpcThreadpool();
    return 0;
}
```

_**system/libhidl/transport/include/hidl/LegacySupport.h**_

```
template<class Interface>
__attribute__((warn_unused_result))
status_t registerPassthroughServiceImplementation(
        std::string name = "default") {
    sp<Interface> service = Interface::getService(name, true /* getStub */);

    if (service == nullptr) {
        ALOGE("Could not get passthrough implementation for %s/%s.",
            Interface::descriptor, name.c_str());
        return EXIT_FAILURE;
    }

    LOG_FATAL_IF(service->isRemote(), "Implementation of %s/%s is remote!",
            Interface::descriptor, name.c_str());

    status_t status = service->registerAsService(name);

    if (status == OK) {
        ALOGI("Registration complete for %s/%s.",
            Interface::descriptor, name.c_str());
    } else {
        ALOGE("Could not register service %s/%s (%d).",
            Interface::descriptor, name.c_str(), status);
    }

    return status;
}
```

_**out/soong/.intermediates/hardware/interfaces/boot/1.0/android.hardware.boot@1.0_genc++/gen/android/hardware/boot/1.0/BootControlAll.cpp**_

```
::android::status_t IBootControl::registerAsService(const std::string &serviceName) {
    ::android::hardware::details::onRegistration("android.hardware.boot@1.0", "IBootControl", serviceName);

    const ::android::sp<::android::hidl::manager::V1_0::IServiceManager> sm
            = ::android::hardware::defaultServiceManager();
    if (sm == nullptr) {
        return ::android::INVALID_OPERATION;
    }
    ::android::hardware::Return<bool> ret = sm->add(serviceName.c_str(), this);
    return ret.isOk() && ret ? ::android::OK : ::android::UNKNOWN_ERROR;
}
```

   ProcessState::self()->getContextObject(NULL)) 会返回一个跟 HwServiceManager 的 BpBinder，然后使用 fromBinder 将其转换为一个 BpHwServiceManager 对象。接着和以前 AIDL 的方式一样，调用 IServiceManager::add 添加服务。  
  HIDL 框架有几个函数要注意一下：fromBinder 接受一个 Binder 对象参数，如果该参数是 Bp 端的 Binder 对象，就会返回对应的 BpHwXXXX 对象；如果是 Bn 端的 Binder 对象，就会返回对应的 BnHwXXXX 对象的_hidl_mImpl 成员。toBinder 接受一个 IXXXX 接口对象参数，如果该参数是 Bp 端的 Binder 对象，就会返回内置的 IBinder 对象；如果是 Bn 端的 Binder 对象，’就返回对应的 BnHwXXXX 对象。可以看到，BpHwXXXX 并不直接继承自 IBinder，但是其拥有的 mImpl 成员就是一个 IBinder 对象。BnHwXXXX 直接继承自 IBinder，但是不继承自所有 HAL 节后的父类 IBase，所以只能用 IBase 类型的_hidl_mImpl 成员实现 IXXXX 接口。  
![](https://img-blog.csdnimg.cn/20181208155711973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ludm9rZXIxMjM=,size_16,color_FFFFFF,t_70)

_**system/libhidl/transport/ServiceManagement.cpp**_

```
sp<IServiceManager1_0> defaultServiceManager() {
    return defaultServiceManager1_1();
}
sp<IServiceManager1_1> defaultServiceManager1_1() {
    {
        AutoMutex _l(details::gDefaultServiceManagerLock);
        if (details::gDefaultServiceManager != NULL) {
            return details::gDefaultServiceManager;
        }

        if (access("/dev/hwbinder", F_OK|R_OK|W_OK) != 0) {
            // HwBinder not available on this device or not accessible to
            // this process.
            return nullptr;
        }

        waitForHwServiceManager();

        while (details::gDefaultServiceManager == NULL) {
            details::gDefaultServiceManager =
                    fromBinder<IServiceManager1_1, BpHwServiceManager, BnHwServiceManager>(
                        ProcessState::self()->getContextObject(NULL));
            if (details::gDefaultServiceManager == NULL) {
                LOG(ERROR) << "Waited for hwservicemanager, but got nullptr.";
                sleep(1);
            }
        }
    }

    return details::gDefaultServiceManager;
}
```

获取服务 IBootControl::getService
-----------------------------

  在 C++ 层，我们只需调用 IXXX::getService 便能获取到对应 HAL 的功能。  
  值得注意的是，我们在启动 BootControl 服务时调用 IBootControl::getService 的第二个参数 getStub 参数为 true，这意味着启动 BootControl 的时候使用的是 Passthrough 模式，这既是启动 HAL 服务的函数被命名为 defaultPassthroughServiceImplementation 的原因。目测除了首次启动 HAL 服务，其他情况下  
getStub 参数都是 false，例如应用程序获取 HAL 服务时。

_**out/soong/.intermediates/hardware/interfaces/boot/1.0/android.hardware.boot@1.0_genc++/gen/android/hardware/boot/1.0/BootControlAll.cpp**_

```
// static
::android::sp<IBootControl> IBootControl::getService(const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<IBootControl> iface = nullptr;

    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();
    if (sm == nullptr) {
        ALOGE("getService: defaultServiceManager() is null");
        return nullptr;
    }

    Return<Transport> transportRet = sm->getTransport(IBootControl::descriptor, serviceName);

    if (!transportRet.isOk()) {
        ALOGE("getService: defaultServiceManager()->getTransport returns %s", transportRet.description().c_str());
        return nullptr;
    }
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);

    #ifdef __ANDROID_TREBLE__

    #ifdef __ANDROID_DEBUGGABLE__
    const char* env = std::getenv("TREBLE_TESTING_OVERRIDE");
    const bool trebleTestingOverride =  env && !strcmp(env, "true");
    const bool vintfLegacy = (transport == Transport::EMPTY) && trebleTestingOverride;
    #else // __ANDROID_TREBLE__ but not __ANDROID_DEBUGGABLE__
    const bool trebleTestingOverride = false;
    const bool vintfLegacy = false;
    #endif // __ANDROID_DEBUGGABLE__

    #else // not __ANDROID_TREBLE__
    const char* env = std::getenv("TREBLE_TESTING_OVERRIDE");
    const bool trebleTestingOverride =  env && !strcmp(env, "true");
    const bool vintfLegacy = (transport == Transport::EMPTY);

    #endif // __ANDROID_TREBLE__

    for (int tries = 0; !getStub && (vintfHwbinder || (vintfLegacy && tries == 0)); tries++) {
        if (tries > 1) {
            ALOGI("getService: Will do try %d for %s/%s in 1s...", tries, IBootControl::descriptor, serviceName.c_str());
            sleep(1);
        }
        if (vintfHwbinder && tries > 0) {
            waitForHwService(IBootControl::descriptor, serviceName);
        }
        Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                sm->get(IBootControl::descriptor, serviceName);
        if (!ret.isOk()) {
            ALOGE("IBootControl: defaultServiceManager()->get returns %s", ret.description().c_str());
            break;
        }
        sp<::android::hidl::base::V1_0::IBase> base = ret;
        if (base == nullptr) {
            if (tries > 0) {
                ALOGW("IBootControl: found null hwbinder interface");
            }continue;
        }
        Return<sp<IBootControl>> castRet = IBootControl::castFrom(base, true /* emitError */);
        if (!castRet.isOk()) {
            if (castRet.isDeadObject()) {
                ALOGW("IBootControl: found dead hwbinder service");
                continue;
            } else {
                ALOGW("IBootControl: cannot call into hwbinder service: %s; No permission? Check for selinux denials.", castRet.description().c_str());
                break;
            }
        }
        iface = castRet;
        if (iface == nullptr) {
            ALOGW("IBootControl: received incompatible service; bug in hwservicemanager?");
            break;
        }
        return iface;
    }
    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(IBootControl::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = IBootControl::castFrom(baseInterface);
                    if (!getStub || trebleTestingOverride) {
                        iface = new BsBootControl(iface);
                    }
                }
            }
        }
    }
    return iface;
}
```

  defaultServiceManager 返回一个 BpHwServiceManager 对象，getTransport 会决定获取服务是通过 Passthrough 模式还是 Binderized 模式。getTransport 接受两个参数，第一个是 HAL 服务的全限定名，本文中是 android.hardware.boot@1.0::IBootControl；第二个是 instance 名，通过先后匹配 / system/manifest.xml 和 / vendor/manifest.xml 来决定使用哪种方式。  
  manifest.xml 的每一项如下所示。android.hardware.boot 对应 <name> 项，1.0 对应 < version > 项，IBootControl 对应 < interface > 项里面的 < name > 项，启动服务的函数 defaultPassthroughServiceImplementation 的第一个参数，如果没有，默认参数为 "default"。getTransport 会按上述的项去寻找符合要求的项，根据 < transport > 的值决定返回值，其中 hwbinder 是 Binderized 模式，passthrough 是 PassThrough 模式。

_**/system/manifest.xml**_

```
<hal format="hidl">
        <name>android.hardware.graphics.composer</name>
        <transport>hwbinder</transport>
        <version>2.1</version>
        <interface>
            <name>IComposer</name>
            <instance>vr</instance>
        </interface>
    </hal>
```

  回到 IBootControl::getService 函数。下面几个参数将决定走的是 Binderized 模式还是 Passthrough 模式：1.getStub 参数，目前我看到只在首次启动服务时是 true，在其他情况为 false；2.vintfHwbinder 在 getTransport 函数返回的是 Binderized 模式时为 true；3.vintfPassthru 在 getTransport 函数返回的是 PassThrough 模式为 true；4.vintfLegacy 在 Android Treble 架构下且未开启__ANDROID_DEBUGGABLE__宏时为 false，即默认为 false。下面情况均按 vintfLegacy 为 false 来说明。  
  当 getStub 参数为 false 且 vintfHwbinder 为 true 时，使用 Binderized 模式；当 getStub 参数为 true 或者 vintfPassthru 为 true 时，使用 PassThrough 模式。也就是说这个 android.hardware.boot@1.0-service 服务时以 PassThrough 模式启动的，但是跟 android.hardware.boot@1.0-service 服务通信时用的是 Binderized 模式。

PassThrough 模式
--------------

  可见 PassThrough 模式时通过 PassthroughServiceManager 进行的。

system/libhidl/transport/ServiceManagement.cpp

```
sp<IServiceManager1_0> getPassthroughServiceManager() {
    return getPassthroughServiceManager1_1();
}
sp<IServiceManager1_1> getPassthroughServiceManager1_1() {
    static sp<PassthroughServiceManager> manager(new PassthroughServiceManager());
    return manager;
}
```

  PassthroughServiceManager::get 函数的任务是使用 dlopen 打开对应的 *-impl 的 so 库，本文中是 android.hardware.boot@1.0-impl.so，找到其中 HIDL_FETCH_XXX 函数执行并返回执行后的返回值，本文中是 BootControl.cpp 中的 HIDL_FETCH_IBootControl 函数。

system/libhidl/transport/ServiceManagement.cpp

```
struct PassthroughServiceManager : IServiceManager1_1 {
    static void openLibs(const std::string& fqName,
            std::function<bool /* continue */(void* /* handle */,
                const std::string& /* lib */, const std::string& /* sym */)> eachLib) {
        //fqName looks like android.hardware.foo@1.0::IFoo
        size_t idx = fqName.find("::");

        if (idx == std::string::npos ||
                idx + strlen("::") + 1 >= fqName.size()) {
            LOG(ERROR) << "Invalid interface name passthrough lookup: " << fqName;
            return;
        }

        std::string packageAndVersion = fqName.substr(0, idx);
        std::string ifaceName = fqName.substr(idx + strlen("::"));

        const std::string prefix = packageAndVersion + "-impl";
        const std::string sym = "HIDL_FETCH_" + ifaceName;

        const int dlMode = RTLD_LAZY;
        void *handle = nullptr;

        dlerror(); // clear

        std::vector<std::string> paths = {HAL_LIBRARY_PATH_ODM, HAL_LIBRARY_PATH_VENDOR,
                                          HAL_LIBRARY_PATH_VNDK_SP, HAL_LIBRARY_PATH_SYSTEM};
#ifdef LIBHIDL_TARGET_DEBUGGABLE
        const char* env = std::getenv("TREBLE_TESTING_OVERRIDE");
        const bool trebleTestingOverride = env && !strcmp(env, "true");
        if (trebleTestingOverride) {
            const char* vtsRootPath = std::getenv("VTS_ROOT_PATH");
            if (vtsRootPath && strlen(vtsRootPath) > 0) {
                const std::string halLibraryPathVtsOverride =
                    std::string(vtsRootPath) + HAL_LIBRARY_PATH_SYSTEM;
                paths.push_back(halLibraryPathVtsOverride);
            }
        }
#endif
        for (const std::string& path : paths) {
            std::vector<std::string> libs = search(path, prefix, ".so");

            for (const std::string &lib : libs) {
                const std::string fullPath = path + lib;

                if (path != HAL_LIBRARY_PATH_SYSTEM) {
                    handle = android_load_sphal_library(fullPath.c_str(), dlMode);
                } else {
                    handle = dlopen(fullPath.c_str(), dlMode);
                }

                if (handle == nullptr) {
                    const char* error = dlerror();
                    LOG(ERROR) << "Failed to dlopen " << lib << ": "
                               << (error == nullptr ? "unknown error" : error);
                    continue;
                }

                if (!eachLib(handle, lib, sym)) {
                    return;
                }
            }
        }
    }

    Return<sp<IBase>> get(const hidl_string& fqName,
                          const hidl_string& name) override {
        sp<IBase> ret = nullptr;

        openLibs(fqName, [&](void* handle, const std::string &lib, const std::string &sym) {
            IBase* (*generator)(const char* name);
            *(void **)(&generator) = dlsym(handle, sym.c_str());
            if(!generator) {
                const char* error = dlerror();
                LOG(ERROR) << "Passthrough lookup opened " << lib
                           << " but could not find symbol " << sym << ": "
                           << (error == nullptr ? "unknown error" : error);
                dlclose(handle);
                return true;
            }

            ret = (*generator)(name.c_str());

            if (ret == nullptr) {
                dlclose(handle);
                return true; // this module doesn't provide this instance name
            }

            registerReference(fqName, name);
            return false;
        });

        return ret;
    }
```

  HIDL_FETCH_IBootControl 的作用就是 hw_get_module 加载对应 HAL 的实现了，最后用一个 BootControl 对象封装好 HAL 的实现并将其返回。BootControl 继承自 IBootControl。IBootControl::getService 在最后用该 BootControl 对象为参数构建一个 BsBootControl 返回。

_**hardware/interfaces/boot/1.0/default/BootControl.cpp**_

```
IBootControl* HIDL_FETCH_IBootControl(const char* /* hal */) {
    int ret = 0;
    boot_control_module_t* module = NULL;
    hw_module_t **hwm = reinterpret_cast<hw_module_t**>(&module);
    ret = hw_get_module(BOOT_CONTROL_HARDWARE_MODULE_ID, const_cast<const hw_module_t**>(hwm));
    if (ret)
    {
        ALOGE("hw_get_module %s failed: %d", BOOT_CONTROL_HARDWARE_MODULE_ID, ret);
        return nullptr;
    }
    module->init(module);
    return new BootControl(module);
}
```

Binderized 模式
-------------

  Binderized 模式就是通过 BpHwServiceManager::get 来获取 HAL 服务。

_**out/soong/.intermediates/system/libhidl/transport/manager/1.1/android.hidl.manager@1.1_genc++/gen/android/hidl/manager/1.1/ServiceManagerAll.cpp**_

```
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::get(const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name){
    ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>  _hidl_out = ::android::hidl::manager::V1_0::BpHwServiceManager::_hidl_get(this, this, fqName, name);

    return _hidl_out;
}
```

  在这里便可以看到明显的 Binder 框架了：Bp 端的 transact 对应 Bn 端的 onTransact，进而调用到 BnHwServiceManager::_hidl_get。

_**out/soong/.intermediates/system/libhidl/transport/manager/1.1/android.hidl.manager@1.1_genc++/gen/android/hidl/manager/1.1/ServiceManagerAll.cpp**_

```
// Methods from IServiceManager follow.
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>> BpHwServiceManager::_hidl_get(::android::hardware::IInterface *_hidl_this, ::android::hardware::details::HidlInstrumentor *_hidl_this_instrumentor, const ::android::hardware::hidl_string& fqName, const ::android::hardware::hidl_string& name) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this_instrumentor->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this_instrumentor->getInstrumentationCallbacks();
    #else
    (void) _hidl_this_instrumentor;
    #endif // __ANDROID_DEBUGGABLE__
    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::client");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&fqName);
        _hidl_args.push_back((void *)&name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service;

    _hidl_err = _hidl_data.writeInterfaceToken(BpHwServiceManager::descriptor);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.writeBuffer(&fqName, sizeof(fqName), &_hidl_fqName_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            fqName,
            &_hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::IInterface::asBinder(_hidl_this)->transact(1 /* get */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    {
        ::android::sp<::android::hardware::IBinder> _hidl__hidl_out_service_binder;
        _hidl_err = _hidl_reply.readNullableStrongBinder(&_hidl__hidl_out_service_binder);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        _hidl_out_service = ::android::hardware::fromBinder<::android::hidl::base::V1_0::IBase,::android::hidl::base::V1_0::BpHwBase,::android::hidl::base::V1_0::BnHwBase>(_hidl__hidl_out_service_binder);
    }

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&_hidl_out_service);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::CLIENT_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>(_hidl_out_service);

_hidl_error:
    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>(_hidl_status);
}
```

  之前提到，BnHwXXXX 的_hidl_mImpl 为对应的 IXXXX 接口实现。BnHwServiceManager::_hidl_get 调用了其_hidl_mImpl 的 get 函数。

_**out/soong/.intermediates/system/libhidl/transport/manager/1.1/android.hidl.manager@1.1_genc++/gen/android/hidl/manager/1.1/ServiceManagerAll.cpp**_

```
// Methods from IServiceManager follow.
::android::status_t BnHwServiceManager::_hidl_get(
        ::android::hidl::base::V1_0::BnHwBase* _hidl_this,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        TransactCallback _hidl_cb) {
    #ifdef __ANDROID_DEBUGGABLE__
    bool mEnableInstrumentation = _hidl_this->isInstrumentationEnabled();
    const auto &mInstrumentationCallbacks = _hidl_this->getInstrumentationCallbacks();
    #endif // __ANDROID_DEBUGGABLE__

    ::android::status_t _hidl_err = ::android::OK;
    if (!_hidl_data.enforceInterface(BnHwServiceManager::Pure::descriptor)) {
        _hidl_err = ::android::BAD_TYPE;
        return _hidl_err;
    }

    const ::android::hardware::hidl_string* fqName;
    const ::android::hardware::hidl_string* name;

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*fqName), &_hidl_fqName_parent,  reinterpret_cast<const void **>(&fqName));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*fqName),
            _hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.readBuffer(sizeof(*name), &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    _hidl_err = ::android::hardware::readEmbeddedFromParcel(
            const_cast<::android::hardware::hidl_string &>(*name),
            _hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { return _hidl_err; }

    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::server");
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)fqName);
        _hidl_args.push_back((void *)name);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_ENTRY, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service = static_cast<BnHwServiceManager*>(_hidl_this)->_hidl_mImpl->get(*fqName, *name);

    ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

    if (_hidl_out_service == nullptr) {
        _hidl_err = _hidl_reply->writeStrongBinder(nullptr);
    } else {
        ::android::sp<::android::hardware::IBinder> _hidl_binder = ::android::hardware::toBinder<
                ::android::hidl::base::V1_0::IBase>(_hidl_out_service);
        if (_hidl_binder.get() != nullptr) {
            _hidl_err = _hidl_reply->writeStrongBinder(_hidl_binder);
        } else {
            _hidl_err = ::android::UNKNOWN_ERROR;
        }
    }
    /* _hidl_err ignored! */

    atrace_end(ATRACE_TAG_HAL);
    #ifdef __ANDROID_DEBUGGABLE__
    if (UNLIKELY(mEnableInstrumentation)) {
        std::vector<void *> _hidl_args;
        _hidl_args.push_back((void *)&_hidl_out_service);
        for (const auto &callback: mInstrumentationCallbacks) {
            callback(InstrumentationEvent::SERVER_API_EXIT, "android.hidl.manager", "1.0", "IServiceManager", "get", &_hidl_args);
        }
    }
    #endif // __ANDROID_DEBUGGABLE__

    _hidl_cb(*_hidl_reply);
    return _hidl_err;
}
```

  BnHwServiceManager 的_hidl_mImpl 来自 ServiceManager，这个 ServiceManager 区别于 Android O 以前的 ServiceManager，这是一个 HAL 服务。所有通过 Binderized 模式添加或者获取服务都是通过 ServiceManager 进行的。

_**system/hwservicemanager/service.cpp**_

```
int main() {
    configureRpcThreadpool(1, true /* callerWillJoin */);

    ServiceManager *manager = new ServiceManager();

    if (!manager->add(serviceName, manager)) {
        ALOGE("Failed to register hwservicemanager with itself.");
    }

    TokenManager *tokenManager = new TokenManager();

    if (!manager->add(serviceName, tokenManager)) {
        ALOGE("Failed to register ITokenManager with hwservicemanager.");
    }

    sp<Looper> looper(Looper::prepare(0 /* opts */));

    int binder_fd = -1;

    IPCThreadState::self()->setupPolling(&binder_fd);
    if (binder_fd < 0) {
        ALOGE("Failed to aquire binder FD. Aborting...");
        return -1;
    }
    // Flush after setupPolling(), to make sure the binder driver
    // knows about this thread handling commands.
    IPCThreadState::self()->flushCommands();

    sp<BinderCallback> cb(new BinderCallback);
    if (looper->addFd(binder_fd, Looper::POLL_CALLBACK, Looper::EVENT_INPUT, cb,
            nullptr) != 1) {
        ALOGE("Failed to add hwbinder FD to Looper. Aborting...");
        return -1;
    }

    // Tell IPCThreadState we're the service manager
    sp<BnHwServiceManager> service = new BnHwServiceManager(manager);
    IPCThreadState::self()->setTheContextObject(service);
    // Then tell binder kernel
    ioctl(binder_fd, BINDER_SET_CONTEXT_MGR, 0);
    // Only enable FIFO inheritance for hwbinder
    // FIXME: remove define when in the kernel
#define BINDER_SET_INHERIT_FIFO_PRIO    _IO('b', 10)

    int rc = ioctl(binder_fd, BINDER_SET_INHERIT_FIFO_PRIO);
    if (rc) {
        ALOGE("BINDER_SET_INHERIT_FIFO_PRIO failed with error %d\n", rc);
    }

    rc = property_set("hwservicemanager.ready", "true");
    if (rc) {
        ALOGE("Failed to set \"hwservicemanager.ready\" (error %d). "\
              "HAL services will not start!\n", rc);
    }

    while (true) {
        looper->pollAll(-1 /* timeoutMillis */);
    }

    return 0;
}
```

  ServiceManager 服务的添加和获取是通过内部的 mServiceMap 的 map 数据结构进行的，这个 map 键值为服务的全限定名，对应的是 PackageInterfaceMap 对象。其中 PackageInterfaceMap 又包含一个 InstanceMap 的 map 数据结构，其以服务的 instance 名为键值，对应的是 HidlService 对象。HidlService 对象包含了 IXXXX 的实现，可以通过 HidlService::getService 函数获取。  
  所以，跟 Android O 以前的 ServiceManager 不同的是，以往的 ServiceManager 会将服务信息保存到内核中，而 HAL 的 ServiceManager 把 HAL 服务的信息保存在 map 数据结构中。  
  可以看到如果 IBootControl::getService 使用的是 PassThrough 模式，返回的是一个 BsBootControl，也就是跟 HAL 服务位于同一进程，跟 Android O 之前的调用方式兼容；如果使用的是 Binderized，返回的是一个 BpHwBootControl，具备跨进程通信的能力。

_**system/hwservicemanager/ServiceManager.cpp**_

```
// Methods from ::android::hidl::manager::V1_0::IServiceManager follow.
Return<sp<IBase>> ServiceManager::get(const hidl_string& fqName,
                                      const hidl_string& name) {
    pid_t pid = IPCThreadState::self()->getCallingPid();
    if (!mAcl.canGet(fqName, pid)) {
        return nullptr;
    }

    auto ifaceIt = mServiceMap.find(fqName);
    if (ifaceIt == mServiceMap.end()) {
        return nullptr;
    }

    const PackageInterfaceMap &ifaceMap = ifaceIt->second;
    const HidlService *hidlService = ifaceMap.lookup(name);

    if (hidlService == nullptr) {
        return nullptr;
    }

    return hidlService->getService();
}
```

Java 层的 Binderized 模式
---------------------

  Java 层获取 HAL 服务是通过 HwBinder#getService 获取的。这是个 native 方法。  
  可以看到 getTransport 返回的不是 Binderized 模式时，便会抛出异常。而当 getTransport 返回的是 Binderized 模式时，在后续的流程就会走 BpHwServiceManager::get 函数。所以说，Java 要使用 HIDL 的前提是这个 HAL 服务是使用 Binderized 模式的（PassThrough 模式不支持 Java）。

_**frameworks/base/core/jni/android_os_HwBinder.cpp**_

```
static jobject JHwBinder_native_getService(
        JNIEnv *env,
        jclass /* clazzObj */,
        jstring ifaceNameObj,
        jstring serviceNameObj) {

    using ::android::hidl::base::V1_0::IBase;
    using ::android::hidl::manager::V1_0::IServiceManager;

    if (ifaceNameObj == NULL) {
        jniThrowException(env, "java/lang/NullPointerException", NULL);
        return NULL;
    }
    if (serviceNameObj == NULL) {
        jniThrowException(env, "java/lang/NullPointerException", NULL);
        return NULL;
    }

    auto manager = hardware::defaultServiceManager();

    if (manager == nullptr) {
        LOG(ERROR) << "Could not get hwservicemanager.";
        signalExceptionForError(env, UNKNOWN_ERROR, true /* canThrowRemoteException */);
        return NULL;
    }

    const char *ifaceNameCStr = env->GetStringUTFChars(ifaceNameObj, NULL);
    if (ifaceNameCStr == NULL) {
        return NULL; // XXX exception already pending?
    }
    std::string ifaceName(ifaceNameCStr);
    env->ReleaseStringUTFChars(ifaceNameObj, ifaceNameCStr);
    ::android::hardware::hidl_string ifaceNameHStr;
    ifaceNameHStr.setToExternal(ifaceName.c_str(), ifaceName.size());

    const char *serviceNameCStr = env->GetStringUTFChars(serviceNameObj, NULL);
    if (serviceNameCStr == NULL) {
        return NULL; // XXX exception already pending?
    }
    std::string serviceName(serviceNameCStr);
    env->ReleaseStringUTFChars(serviceNameObj, serviceNameCStr);
    ::android::hardware::hidl_string serviceNameHStr;
    serviceNameHStr.setToExternal(serviceName.c_str(), serviceName.size());

    LOG(INFO) << "Looking for service "
              << ifaceName
              << "/"
              << serviceName;

    Return<IServiceManager::Transport> transportRet =
            manager->getTransport(ifaceNameHStr, serviceNameHStr);

    if (!transportRet.isOk()) {
        signalExceptionForError(env, UNKNOWN_ERROR, true /* canThrowRemoteException */);
        return NULL;
    }

    IServiceManager::Transport transport = transportRet;

#ifdef __ANDROID_TREBLE__
#ifdef __ANDROID_DEBUGGABLE__
    const char* testingOverride = std::getenv("TREBLE_TESTING_OVERRIDE");
    const bool vintfLegacy = (transport == IServiceManager::Transport::EMPTY)
            && testingOverride && !strcmp(testingOverride, "true");
#else // __ANDROID_TREBLE__ but not __ANDROID_DEBUGGABLE__
    const bool vintfLegacy = false;
#endif // __ANDROID_DEBUGGABLE__
#else // not __ANDROID_TREBLE__
    const bool vintfLegacy = (transport == IServiceManager::Transport::EMPTY);
#endif // __ANDROID_TREBLE__";

    if (transport != IServiceManager::Transport::HWBINDER && !vintfLegacy) {
        LOG(ERROR) << "service " << ifaceName << " declares transport method "
                   << toString(transport) << " but framework expects hwbinder.";
        signalExceptionForError(env, NAME_NOT_FOUND, true /* canThrowRemoteException */);
        return NULL;
    }

    Return<sp<hidl::base::V1_0::IBase>> ret = manager->get(ifaceNameHStr, serviceNameHStr);

    if (!ret.isOk()) {
        signalExceptionForError(env, UNKNOWN_ERROR, true /* canThrowRemoteException */);
        return NULL;
    }

    sp<hardware::IBinder> service = hardware::toBinder<hidl::base::V1_0::IBase>(ret);

    if (service == NULL) {
        signalExceptionForError(env, NAME_NOT_FOUND);
        return NULL;
    }

    LOG(INFO) << "Starting thread pool.";
    ::android::hardware::ProcessState::self()->startThreadPool();

    return JHwRemoteBinder::NewObject(env, service);
}
```