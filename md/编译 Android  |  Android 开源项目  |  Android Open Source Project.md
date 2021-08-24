> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [source.android.com](https://source.android.com/setup/build/building)

 .devsite-banner-announcement { background: #202124; color: white; font-weight: 500; } .devsite-banner-announcement a { color: white; font-weight: 700; } .devsite-banner-announcement a:link, .devsite-banner-announcement a:visited {background: 0;} .devsite-banner-announcement a:hover, .devsite-banner-announcement a:focus { text-decoration: none; } .devsite-banner-announcement .devsite-banner-message-text { margin: 0 auto; } 。

*   [AOSP](https://source.android.com/)
*   [设置](https://source.android.com/setup)
*   [构建](https://source.android.com/setup/build)

该内容对您有帮助吗？

编译 Android
==========

*   目录
*   [设置环境](#initialize)
*   [选择目标](#choose-a-target)
    *   [lunch](#lunch)
    *   [tapas](#tapas)
*   [构建代码](#build-the-code)
*   [运行构建](#run-it)
    *   [使用 fastboot 刷机](#flash-a-device)
    *   [模拟 Android 设备](#emulate-an-android-device)
*   [了解 build 指纹](#understanding-build-fingerprints)
*   [排查常见构建错误](#troubleshooting-common-build-errors)
    *   [Java 版本不正确](#wrong-java-version)
    *   [不区分大小写的文件系统](#case-insensitive-filesystem)
    *   [没有 USB 权限](#no-usb-permission)

请按照以下说明构建 Android。

设置环境
----

使用 `envsetup.sh` 脚本初始化环境：

```
source build/envsetup.sh


```

或

```
. build/envsetup.sh


```

请参阅 [platform/build/envsetup.sh](https://android.googlesource.com/platform/build/+/refs/heads/master/envsetup.sh) 中的脚本，了解相关命令的说明，包括用于选择设备目标的 [lunch](#lunch)，以及用于构建未捆绑应用（例如[参考 TV 应用](/devices/tv/reference-tv-app)）的 [tapas](#tapas)。

您需要在每次运行 `repo sync` 后重新发出此命令，以获取对该脚本所做的任何更改。请注意，将 `source` 替换为 `.`（一个点）可以省去一些字符，这种简写形式在文档中更为常用。

`envsetup.sh` 脚本会导入若干命令，让您能够使用 Android 源代码，其中包括本练习中使用的命令。

如需查看可用命令的完整列表，请运行以下命令：

```
hmm


```

选择目标
----

### lunch

使用 `lunch` 选择要构建的目标。`lunch product_name-build_variant` 会选择 product_name 作为需要构建的产品，并选择 build_variant 作为需要构建的变体，然后将这些选择存储在环境中，以便供后续对 `m` 和其他类似命令的调用读取。

确切的配置可作为参数进行传递。例如，以下命令表示针对模拟器进行完整构建，并且启用所有调试功能。

```
lunch aosp_arm-eng


```

如果在没有参数的情况下运行，则 `lunch` 会提示您从菜单中选择目标。如需了解所有现有设备的构建配置，请参阅[选择设备构建系统](/setup/build/running#selecting-device-build)。

所有构建目标都采用 `BUILD-BUILDTYPE` 形式，其中 `BUILD` 是表示特定功能组合的代号。`BUILDTYPE` 是以下类型之一：

<table><thead><tr><th>构建类型</th><th>使用情况</th></tr></thead><tbody><tr><td>user</td><td>权限受限；适用于生产环境</td></tr><tr><td>userdebug</td><td>与 “user” 类似，但具有 root 权限和调试功能；是进行调试时的首选编译类型</td></tr><tr><td>eng</td><td>具有额外调试工具的开发配置</td></tr></tbody></table>

userdebug 版本的运行方式应该与 user 版本一样，且能够启用通常不符合平台安全模型的额外调试功能。这就使得 userdebug 版本具有更强大的诊断功能，因此是进行 user 测试的最佳选择。使用 userdebug 编译系统进行开发时，请遵循 [userdebug 指南](/setup/develop/new-device#userdebug-guidelines)。

eng 编译系统会优先考虑在平台上工作的工程师的工程生产率。eng 编译系统会关闭用于提供良好用户体验的各种优化。除此之外，eng build 的运行方式类似于 user 和 userdebug build，以便设备开发者能够看到代码在这些环境下的运行方式。

如需详细了解如何在实际硬件上进行构建和运行，请参阅[刷写设备](/setup/build/running)。

### tapas

`tapas` 命令用于配置未捆绑应用的构建流程。它会选择要由 Android 构建系统构建的各个应用。与 `lunch` 不同，`tapas` 不会请求为设备构建映像。

如需详细了解该命令，请运行 `tapas help`。

构建代码
----

本部分是一个简短摘要，用于确保设置已完成。

使用 `m` 构建所有内容。`m` 可以使用 `-jN` 参数处理并行任务。如果您没有提供 `-j` 参数，构建系统会自动选择您认为最适合您系统的并行任务计数。

```
m


```

如上所述，您可以通过在 `m` 命令行中列出相应名称来构建特定模块，而不是构建完整的设备映像。此外，`m` 还针对各种特殊目的提供了一些伪目标。以下是一些示例：

*   **`droid`** - `m droid` 是正常 build。此目标在此处，因为默认目标需要名称。
*   **`all`** - `m all` 会构建 `m droid` 构建的所有内容，加上不包含 `droid` 标记的所有内容。构建服务器会运行此命令，以确保包含在树中且包含 `Android.mk` 文件的所有元素都会构建。
*   **`m`** - 从树的顶部运行构建系统。这很有用，因为您可以在子目录中运行 `make`。如果您设置了 `TOP` 环境变量，它便会使用此变量。如果您未设置此变量，它便会从当前目录中查找相应的树，以尝试找到树的顶层。您可以通过运行不包含参数的 `m` 来构建整个源代码树，也可以通过指定相应名称来构建特定目标。
*   **`mma`** - 构建当前目录中的所有模块及其依赖项。
*   **`mmma`** - 构建提供的目录中的所有模块及其依赖项。
*   **`croot`** - `cd` 到树顶部。
*   **`clean`** - `m clean` 会删除此配置的所有输出和中间文件。此内容与 `rm -rf out/` 相同。

运行 `m help` 即可查看 `m` 提供的其他伪目标。

运行构建系统
------

您可以在模拟器上运行构建系统，也可以将其刷写到设备上。由于您已经使用 `lunch` 选择了构建目标，因此它不可能在构建目标以外的目标上运行。

**注意**：请注意获取专有二进制文件，否则 build 将无法在目标硬件上成功启动。有时，源代码可能会对不同 build 和分支提供不同的二进制文件。如果此时获取二进制 Blob，则需要解压缩这些 Blob 并执行 `m installclean`，然后重新构建。如需详细了解此过程，请参阅[获取专有二进制文件](/setup/build/downloading#obtaining-proprietary-binaries)。

### 使用 fastboot 刷机

如需刷写设备，请使用 `fastboot`；在构建成功后，它应该会包含在您的路径中。如需相关说明，请参阅[刷写设备](/setup/build/running#flashing-a-device)。

### 模拟 Android 设备

编译流程会自动将模拟器添加到您的路径中。如需运行模拟器，请输入以下命令：

```
emulator


```

了解 build 指纹
-----------

如需跟踪和报告与特定 Android build 相关的问题，请一定要了解 build 指纹。build 指纹是能让人看懂的唯一字符串，其中包含向每个 build 发出的制造商信息。有关详情，请参阅 Android 兼容性定义文档 (CDD) 中 [Build 参数](/compatibility/android-cdd#3_2_2_build_parameters)部分内的 “FINGERPRINT” 说明。

build 指纹表示特定的 Android 实现和修订版本。此唯一键可让应用开发者和其他人报告与特定固件版本相关的问题。如需了解 Android 问题报告流程，请参阅[报告 bug](/setup/contribute/report-bugs)。

build 指纹封装了所有 Android 实现详情：

*   API：Android 和原生，以及软 API 行为
*   核心 API 和部分系统界面行为
*   CDD 中定义的兼容性和安全性要求
*   应用所采用的产品规范和 [uses-feature](https://developer.android.com/guide/topics/manifest/uses-feature-element.html) 设置，用于定位符合预期要求的设备
*   硬件和软件组件的实现

如需查看完整详情，请参阅 [CDD](/compatibility/android-cdd)。如需有关打造全新 Android 设备的说明，请参阅[添加新设备](/setup/develop/new-device)。

排查常见构建错误
--------

### Java 版本不正确

如果您尝试构建的 Android 版本与您的 Java 版本不一致，则 `make` 会中止，并显示如下消息：

```
************************************************************
You are attempting to build with the incorrect version
of java.

Your version is: WRONG_VERSION.
The correct version is: RIGHT_VERSION.

Please follow the machine setup instructions at
    https://source.android.com/source/initializing.html
************************************************************


```

以下是可能的原因和解决方案：

*   无法按照 [JDK 要求](/setup/build/requirements#jdk)中指定的方式安装正确的 JDK。请确保您已按照[设置环境](/setup/build/building#initialize)和[选择目标](/setup/build/building#choose-a-target)中的步骤操作。
*   之前安装的另一个 JDK 出现在您的路径中。将正确的 JDK 附加到路径开头，或者移除有问题的 JDK。

### 不区分大小写的文件系统

在 macOS 中的 HFS 文件系统上进行编译时，可能会遇到如下错误：

```
************************************************************
You are building on a case-insensitive filesystem.
Please move your source tree to a case-sensitive filesystem.
************************************************************


```

请按照[创建区分大小写的磁盘映像](/setup/build/initializing#creating-a-case-sensitive-disk-image)中的说明操作。

### 没有 USB 权限

默认情况下，在大多数 Linux 系统中，无特权的用户无法使用 USB 端口。如果您看到权限遭拒错误，请按照[配置 USB 使用权限](/setup/build/initializing#configuring-usb-access)中的说明操作。

如果上述规则设置完成后，[ADB](/setup/build/adb) 已在运行且无法连接到设备，您可以使用 `adb kill-server` 将其终止。此命令将使 ADB 重启并采用新配置。

该内容对您有帮助吗？