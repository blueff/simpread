> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/@iamryanodonnell/guide-upgrading-to-ios-13-from-windows-10-9fa6f87c8f7a)

[![](https://miro.medium.com/fit/c/56/56/0*F4JLQL0uHXnD5ncn)](https://medium.com/@iamryanodonnell?source=post_page-----9fa6f87c8f7a--------------------------------)

![](https://miro.medium.com/max/60/1*WLgAnQX1mGmyKdKdnwRxgA.png?q=20)

![](https://miro.medium.com/max/1920/1*WLgAnQX1mGmyKdKdnwRxgA.png)

Installation of iOS 13 from Windows

**THIS IS IMPORTANT — I am not responsible for loss or damage to your device or data. You are also installing beta software which is guaranteed to contain many bugs. Also, please back up your phone before continuing.**

The inspiration for this guide was due to my own stupid mistake — installing the watchOS 6 beta on an iOS 12 device. It installed successfully, however, it unpaired from the iPhone, and could not be re-paired due to software incompatibility with iOS 12.3.1, as shown from the image below:

![](https://miro.medium.com/max/28/1*oDbQuVtsQm8nBq9XiHlOWw.jpeg?q=20)

![](https://miro.medium.com/max/355/1*oDbQuVtsQm8nBq9XiHlOWw.jpeg)

You’ll need these before you continue:  
- iTunes (installed from Apple’s website, NOT Microsoft Store)  
- iOS 13 IPSW file (available at betaprofiles.com)  
- libimobile2019 ([GitHub](https://github.com/Devjam81/libimobile2019))

**Step 1: Install iTunes  
**Download iTunes from Apple’s website, and do not run or install the version from the Microsoft Store.  
Remember to archive your backup by going to where your “Backup” folder is located.

**Step 2: Download and extract libimobile2019  
**Place this in the root of your drive (for example, C:\)

**Step 3: Copy & paste the iOS 13 IPSW file, then rename it  
**To make things easier, copy and paste the IPSW file into the libimobile2019-master folder, and rename it to ios13.ipsw.

**Step 4: Load Command Prompt as an administrator  
**Search for Command Prompt in your Start menu, and then right click and then click on “Run as administrator”, and navigate to where the libimobile2019-master folder is. An example is as follows:

“cd C:\libimobile2019-master\”

**Step 5: If not already connected, connect your iPhone  
**Connect your iPhone to your PC using the Lightning cable.

**Step 6: TIME TO FLASH  
**It’s time to flash your iPhone, and there’s no going back from this point forwards.

In Command Prompt, type the following:  
idevicerestore.exe -d ios13.ipsw

Your command prompt should now show a load of code (don’t be worried!), and begins the process of putting your iPhone into recovery mode and upgrading you to iOS 13.

**Step 7: If you get IBEC errors  
**The first thing to make sure you do is to have the version of iTunes installed from the Apple website, and not the Microsoft Store.

Secondly, if you have the Apple website version, go to Device Manager, and then “Universal Serial Bus devices”.  
Find “Apple Recovery (iBoot)” and then Uninstall device, and tick “Uninstall driver software”.

Unplug your iPhone and then reconnect it, and after a few seconds, try the above command again.

**Step 8: Success  
**The command prompt window should display “DONE” when the process is complete.

Your iPhone will take longer than usual to boot, and you may be prompted to “Swipe up for recovery”. This will keep your data in place, and attempt recovery for your iOS 13 installation.

When successful, your iPhone will reboot again, and you’ll be able to explore iOS 13!