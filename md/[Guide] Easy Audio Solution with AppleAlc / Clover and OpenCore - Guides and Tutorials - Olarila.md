> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.olarila.com](https://www.olarila.com/topic/6074-guide-easy-audio-solution-with-applealc-clover-and-opencore/)

*   **Administrators**

 [![](https://www.olarila.com/uploads/monthly_2020_02/SystemLogo.thumb.png.8835e028ff1c8e4dac5ad11c7b352cbe.png)](https://www.olarila.com/profile/2-mald0n/ "Go to MaLd0n's profile") 

### [MaLd0n](https://www.olarila.com/profile/2-mald0n/ "Go to MaLd0n's profile") 1,001,160

[Posted April 18, 2019](https://www.olarila.com/topic/6074-guide-easy-audio-solution-with-applealc-clover-and-opencore/?do=findComment&comment=73820)

### **[MaLd0n](https://www.olarila.com/profile/2-mald0n/ "Go to MaLd0n's profile")**

*   ACPI Eater
*    [![](https://www.olarila.com/uploads/monthly_2020_02/SystemLogo.thumb.png.8835e028ff1c8e4dac5ad11c7b352cbe.png)](https://www.olarila.com/profile/2-mald0n/ "Go to MaLd0n's profile") 
*   Administrators
*   ![](https://www.olarila.com/uploads/monthly_2020_02/team_Administrator.png.bd0cd8947f97b1e6e15925f4feeef633.png)
*   1,001,160
*   1,010,760 posts

*   **Administrators**

*   [](#elControls_73820_menu "More options...")
    *   [Share](https://www.olarila.com/topic/6074-guide-easy-audio-solution-with-applealc-clover-and-opencore/ "Share this post")

[Posted April 18, 2019](https://www.olarila.com/topic/6074-guide-easy-audio-solution-with-applealc-clover-and-opencore/?do=findComment&comment=73820)

**Donation with Paypal** **[https://tinyurl.com/r7fchn9](https://tinyurl.com/r7fchn9)**

**Donation with BTC** **33HeGCuCSh4tUBqdYkQqKpSDa1E7WeAJQ3**

**Donation with PicPay** **@danielnmaldonado**

**Donation with PiX** **@danielnmaldonado@gmail.com**

**Donation with Mercado Livre** **[https://tinyurl.com/y9evkj5u](https://tinyurl.com/y9evkj5u)**

****1-Use AppleAlc.kext and Lilu.kext in Clover/kexts/other(preferred) or Library/Extensions****

**Download AppleAlc**

[https://tinyurl.com/y6vh5aqy](https://tinyurl.com/y6vh5aqy)

**Download Lilu**

[https://tinyurl.com/ya657fxl](https://tinyurl.com/ya657fxl)

****2-You need check what your audio codec****

You can use a **Hackintool**, ****Manufacturer Page**** or ****Run_me.app**** for check it(stay with AppleAlc load)

****-Hackintool****

**Download Hackintool**

[https://tinyurl.com/yamzbgd6](https://tinyurl.com/yamzbgd6)

![](https://i.imgur.com/3vBNaq9.png)

****-Manufacturer Page****

![](https://i.imgur.com/DFRAkC7.png)

****-Run_Me.app****

**Download Run_Me.app**

[https://tinyurl.com/y9562hz7](https://tinyurl.com/y9562hz7)

![](https://i.imgur.com/OfyUZPd.png)

![](https://i.imgur.com/tak3655.png)

****Let's Start**** ![](https://www.olarila.com/uploads/emoticons/icon_mrgreen.gif.6f9694644c180bbd7e68748ae419c70a.gif)

****1-Go to** ******AppleAlc GitHub****** **page****

[https://tinyurl.com/yath7r42](https://tinyurl.com/yath7r42)

****2-Go to "********Clone or download********/********Download ZIP********"****

![](https://i.imgur.com/zhVOr1p.png)

****3-Extract file and go to** ******Resources****** **folder****

![](https://i.imgur.com/gzSMpvn.png)

****4-Check what** ******layout-ids****** **you can use for your audio codec(in my case is ALC892)****

![](https://i.imgur.com/8uOzTYM.png)

****I have many options****

![](https://i.imgur.com/3VHEOi5.png)

****5-Clover Solution****

****Open config.plist with one good plist editor, you can use a Clover.app, go to Boot/Arguments and add alcid=xx (xx is layout-id for your audio codec)****

![](https://i.imgur.com/l0Nqyhs.png)

**6-OpenCore Solution**

****Open config.plist with one good plist editor, go to NVRAM/Add/boot-args and add alcid=xx (xx is layout-id for your audio codec)****

****![](https://i.imgur.com/u7YPqVY.png)****

****-Credits and thanks to the old and new people in the community who developed patches, kexts and bootloaders!****

**Slice, Kabyl, usr-sse2, jadran, Blackosx, dmazar, STLVNUB, pcj, apianti, JrCs, pene, FrodoKenny, skoczy, ycr.ru, Oscar09, xsmile, SoThOr, RehabMan, Download-Fritz, Zenit432, cecekpawon, Intel, Apple, Oracle, Chameleon Team, crazybirdy, Mieze, Mirone, Oldnapalm, netkas, Elconiglio, artut-pt, ErmaC, Pavo, Toleda, Master Chief and family, bcc9, The King, PMheart, Sherlocks, Micky1979, vit9696, vandroiy2013, Voodoo Team, Pike R. Alpha, lvs1974, Austere.J, CVad, headkaze, onemanosx, erroruser, Jenny David, Sampath007, Facebook Hackintosh community, Facebook Olarila community and many, many, many others!**

**We're all here to have fun and learn from each other!**

*   ![](https://www.olarila.com/uploads/reactions/react_like.png) 1
*   ![](https://www.olarila.com/uploads/reactions/react_thanks.png) 1

##### Link to post

##### Share on other sites