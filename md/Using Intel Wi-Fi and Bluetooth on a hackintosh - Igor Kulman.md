> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.kulman.sk](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/)

If you use a hackintosh you have to choose your hardware carefully to make sure it is supported by macOS. You can get [Wi-Fi + Bluetooth card used by Apple as I did in my desktop](https://blog.kulman.sk/adding-wifi-and-bluetooth-for-apple-features-to-hackintosh/), but sometimes you do not have much choice.

When I turned by old `Thinkpad T440s` into a hackintosh I bought a Wi-Fi dongle because the `Intel AC7260` Wi-Fi + Bluetooth card is not supported by macOS, no Intel cards are.

Later I discovered and [open-source project that aims to make Intel Wi-Fi and Bluetooth work on macOS](https://github.com/OpenIntelWireless) and I was able to make the `Intel AC7260` card work, no dongles needed.

### Bluetooth driver

To get Intel Bluetooth working you need [IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware). It is a macOS kernel extension that that uses firmware binaries from Linux to make Bluetooth work.

Make sure your specific Intel card is supported, download the [latest release](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases) and use the two kexts; `IntelBluetoothFirmware.kext` and `IntelBluetoothInjector.kext`. If you use `Clover` just copy them to `EFI/Clover/Kexts/Other`.

Make sure you do not use any of `AirportBrcmFixup`, `BT4LEContinuityFixup`, `BrcmBluetoothInjector`, `BrcmPatchRAM3` so you do not create a conflict.

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/BT_hu1da6870ea44fa9cf3d692b0394a9a223_72141_48x0_resize_q20_box_2.png)

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/BT.png)

After reboot Bluetooth will appear in System Preferences and you will be able to find and pair your Bluetooth devices.

### Wi-Fi driver

To get Intel Wi-Fi working you need [itlwm](https://github.com/OpenIntelWireless/itlwm). Similar to `IntelBluetoothFirmware` it is a macOS kernel extension using firmware from Linux.

Make sure your specific Intel card is supported and download the [latest release](https://github.com/OpenIntelWireless/itlwm/releases). The release includes two kexts; `itlwm.kext` and `itlwmx.kext`. The `itlwmx.kext` is for use with the Intel X cards, like `Intel X200`, the `itlwm.kext` is for all the older cards like mine.

#### Networks management

When loaded, `itlwm.kext` makes your Intel Wi-Fi card available as an Ethernet card, not as a Wi-Fi card. This means you will not get the classic macOS user interface for connecting to Wi-Fi networks.

You need to either configure your Wi-Fi networks either manually or use a custom Wi-Fi management app.

To configure the Wi-Fi networks manually open `itlwm.kext` and find `Info.plist`. If you open `Info.plist` you will see a section called `IOKitPersonalities:itlwm:WiFiConfig` with 4 Wi-Fi networks configured. Just change it to your networks configuration, providing your networks names and passwords and save the changes.

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/itlwm_manual_hua66e5ea35551dc3598e306f61ce228ec_98938_48x0_resize_q20_box_2.png)

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/itlwm_manual.png)

You can load `itlwm.kext` immediately

```
sudo cp -R itlwm.kext /tmp
sudo chown -R root:wheel /tmp/itlwm.kext
sudo kextload /tmp/itlwm.kext
```

Or on every boot, if you use `Clover` just copy the correct kext to `EFI/Clover/Kexts/Other`.

Make sure you do not use any of `AirportBrcmFixup`, `BT4LEContinuityFixup`, `BrcmBluetoothInjector`, `BrcmPatchRAM3` so you do not create a conflict.

### Wi-Fi management app

If you do not want to manage your Wi-Fi networks editing the `Info.plist` file in `itlwm.kext` or `itlwmx.kext` there is a project called [HeliPort](https://github.com/OpenIntelWireless/HeliPort).

This project aims to replicate the standard macOS Wi-Fi management UI.

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/HeliPort_hua3d3c1e27fae3f99d52035a02c110e8e_72886_48x0_resize_q20_box_2.png)

![](https://blog.kulman.sk/using-intel-wifi-bt-on-macos/HeliPort.png)

Just download the [latest release](https://github.com/OpenIntelWireless/HeliPort/releases), copy it to Applications and launch it.

#### See also

*   [Using different Git config for personal and work projects](https://blog.kulman.sk/different-git-config-for-work-projects/)
*   [Workaround for Swift scripts crashing after update to Xcode 11.4](https://blog.kulman.sk/workaround-for-swift-scripts-crash/)
*   [Adding Wi-Fi and Bluetooth to hackintosh to enable Apple-specific features](https://blog.kulman.sk/adding-wifi-and-bluetooth-for-apple-features-to-hackintosh/)
*   [Editing macOS app About dialog](https://blog.kulman.sk/editing-macos-app-about-dialog/)
*   [Detecting click on a specific NSTableViewCell](https://blog.kulman.sk/detecting-click-on-a-nstableviewcell/)