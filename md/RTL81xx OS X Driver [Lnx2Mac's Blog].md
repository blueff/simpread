> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [lnx2mac.blogspot.com](http://lnx2mac.blogspot.com/p/realtekrtl81xx-osx-driver.html)

![](https://lh5.googleusercontent.com/-HYwG0oBavZk/ToIDGYEdYfI/AAAAAAAAAPc/mtgAtF9ClQ4/RTL81xx-ChangesNetworking.png)

Overview:
---------

Due to the shortcomings of the current Realtek 81xx Hackintosh drivers (such as lack of or limited support of 8111E, 32/64-bit, sleep issues), I endeavored to port the Linux RTL81xx driver to Mac OS X.

Porting a driver is never a trivial task, but this one brought two interesting challenges:

1.  Lack of documentation (Realtek provides it only under NDA)
2.  and... I didn't have access to a board with the 8111E

I started this port based on Chuck Fry's _"Chucko R1000SL"_, and merging in support for new chips, from the Realtek Official Linux drivers (released August 31, 2010).

RealtekRTL81xx.kext is the result of this work, and it fully supports the RTL8168/RTL8111 family of NICs.

![](http://lh3.ggpht.com/_9g2xyiIBsNU/THhoql4Qm2I/AAAAAAAAALg/64IlyYnNd-0/RTL81xx.png)

Features:
---------

Some highlights of the issues it fixes, and features provided are:

*   Support for the following Realtek chips:
    *   RTL8168 / RTL8168B / RTL8168C
    *   RTL8111 / RTL8111B / RTL8111C / RTL8111CP / RTL8111D / RTL8111D(L) / RTL8111DP / RTL8111E
    *   Newer (SandyBridge) RTL8111E, RTL8111E-VL, and RTL8111F _**(Since v0.0.90)**_
*   Full 32-bit and 64-bit support
*   Fixed Sleep/Wake issues
*   Full support of WakeOnLAN _**(Since v0.0.90)**_
*   Full support of WakeOnDemand _**(Since v0.0.90)**_
*   Improved Auto-Negotiation
*   Implemented negotiated-media reporting
*   Improved stability
*   Improved performance (see image below)
*   Numerous unreported/minor bug fixes
*   Mac OS X Snow Leopard and Lion support
*   Convenient Installer Package

[![](http://lh5.ggpht.com/_9g2xyiIBsNU/TJeoZnWvbzI/AAAAAAAAAMA/ouaT96DNfto/s400/RTL81xx-v0_0_67-performance.png)](http://lh5.ggpht.com/_9g2xyiIBsNU/TJeoZnWvbzI/AAAAAAAAAMA/ouaT96DNfto/s1600/RTL81xx-v0_0_67-performance.png)

Over _80 members_ of the [tonymacx86 Forum](http://tonymacx86.blogspot.com/) participated in 6 restricted beta cycles, testing it on several platforms, P55, H55, X58.  
I'd like to specially thank [MacMan](http://macmanx86.blogspot.com/) for his cooperation and invaluable help in alpha testing this (for several days, I would email him the driver, and he would reply with logs) !

Support my work:
----------------

As you probably understand, a lot of time and effort was (and is) spent on developing and maintaining this project.

I plan to keep this Driver free and public for anyone in the Mac OS X community.

If you like it, please consider helping.  
Making a donation is _not required_, but will be **greatly appreciated !**

Download Links:
---------------

Please Do not redistribute this, but instead **link to this page**. Current Stable Release: **N/A** Current BETA Release:

*   [RealtekRTL81xx-0_0_90.pkg (Google Docs)](http://bit.ly/pLlD13)

Previous BETA Releases:

*   [RealtekRTL81xx-0_0_67+.pkg (Google Docs)](https://docs.google.com/uc?id=0B2RpaAfDQsmeMjBlNjAwMGQtY2I0OS00OGIzLTk0M2ItNDhiOGE1MmQwZWU4&export=download&authkey=CIaL858N&hl=en)
*   [RealtekRTL81xx-0_0_67+.pkg (GitHub Mirror)](https://github.com/downloads/Lnx2Mac/RTL81xx/RealtekRTL81xx-0_0_67+.pkg)
*   [RealtekRTL81xx-0_0_67.zip](https://docs.google.com/uc?id=0B2RpaAfDQsmeNjk0YjMwYzMtMjBiNy00ODgxLWIzODEtYzE3NDVmOWZmOTNj&export=download&authkey=CJWUw_UC&hl=en)

Installation instructions:
--------------------------

*   Download and execute the Installer .pkg file
*   Follow the instructions on screen

_Note that if you have installed conflicting kexts, these will be moved to a folder on your desktop._

Troubleshooting:
----------------

Based on the number of problem reports I get, I created the following page: [Reviving Dead Ethernet Ports](http://lnx2mac.blogspot.com/2011/03/reviving-dead-ethernet-ports.html)

Reporting Problems:
-------------------

The guys at the [tonymacx86 Forum](http://www.tonymacx86.com/) created a [subforum](http://www.tonymacx86.com/viewforum.php?f=64) just for this! **Thanks guys, you rock !** Report all issues there !  
Other comments and feedback, can be posted here.

Enjoy,  
Lnx2Mac

![](http://www.google.com/help/hc/images/logos/blogger_logo.gif)There's currently an [issue with Blogger](http://www.google.com/support/forum/p/blogger/thread?tid=56e0f315d3a8eab6&hl=en) not showing/accepting comments on Static Pages.

In the meantime, please post your comments in the [Lnx2Mac Releases new RTL81xx Beta](http://lnx2mac.blogspot.com/2011/09/lnx2mac-releases-new-rtl81xx-beta.html) blog post.

Update: As of 19-Oct-2011 4:00AM GMT, it seems this blogger issue has been fixed