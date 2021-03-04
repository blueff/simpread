> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.macbreaker.com](http://www.macbreaker.com/2012/01/how-to-manually-install-kexts.html)

[![](http://4.bp.blogspot.com/-mUOl2VF-cXY/Tx8-ysLdlFI/AAAAAAAAAS0/rRrkR654XYE/s1600/kextbeast.png)](http://4.bp.blogspot.com/-mUOl2VF-cXY/Tx8-ysLdlFI/AAAAAAAAAS0/rRrkR654XYE/s1600/kextbeast.png)

Every now and then, you'll find need to install a kext that isn't included in [Multibeast](http://multibeast/). Unfortunately, installing kexts manually isn't as simple as dragging the kext file into /System/Library/Extensions (where most kexts are installed to). You also need to repair system permissions every time you perform an installation; you can do this manually on through Terminal, but it's really not worth the effort when using KextBeast is a much easier solution. 

[KextBeast](http://tonymacx86.com/downloads) is a tool provided by tonymacx86 for installing kexts. It supports Lion and allows you to install kexts to any hard drive partition that you choose (in case you need to install kexts on a [cloned hard drive](http://www.macbreaker.com/2012/01/look-at-hard-drive-cloning.html)), unlike [Kext Helper](http://cheetha.net/), a formerly popular kext installer. To install any kext file with Kextbeast, place the desired kexts for installation on your desktop and run the Kextbeast installer. After choosing which drive to install to, the process is fully automated.

  

[![](http://2.bp.blogspot.com/-meFDBSPfkbs/Tx9AfmFBl9I/AAAAAAAAAS8/NBLS2VojSMQ/s1600/Region+capture+2.png)](http://2.bp.blogspot.com/-meFDBSPfkbs/Tx9AfmFBl9I/AAAAAAAAAS8/NBLS2VojSMQ/s1600/Region+capture+2.png)

  

If you insist on doing it the command-line way, here's how: First, copy the kext file to /System/Library/Extensions. Then, open the Terminal app (located in OS X under Applications->Utilities) and type:

**cd /System/Library/Extensions**  

For those unfamiliar with command line, this just changes the directory that Terminal is working on to /System/Library/Extensions. Then type:

**sudo chmod -R 755 name.kext**  
**sudo chown -R root:wheel name.kext**

Replace "name.kext" with the name of the kext you're installing. This fixes the kext's permissions (basically what System Utilities does in Multibeast). If you're running Mac OS X Lion, this is all you need to do. If you're still running Snow Leopard, you also need to clear the kernel cache. Type in this:

**sudo rm -R Extensions.kextcache**  
**sudo rm -R Extensions.mkext**

Then reboot. If everything went well, then hopefully the newly installed kext will do the job.