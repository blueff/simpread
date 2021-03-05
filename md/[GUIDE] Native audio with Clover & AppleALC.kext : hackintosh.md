> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.reddit.com](https://www.reddit.com/r/hackintosh/comments/4e23w6/guide_native_audio_with_clover_applealckext/)

This guide will show you how to natively enable audio with [AppleALC.kext](https://github.com/vit9696/AppleALC) for patching & injection. With this new kext by [/u/vit9696](https://www.reddit.com/u/vit9696/), audio won't be affected by software updates and SIP/kext injection can remain enabled, as the kext is injectable from the EFI partition.

**NOTE:** This guide requires a bootloader capable of kext injection. **You NEED an original, unpatched AppleHDA.kext**. This guide is for OS X 10.9 and above; older versions require different patches and are not fully supported by AppleALC.kext. **Note that Skylake systems may need the HDA ACPI device to be renamed to HDEF**; this can be done with a Clover ACPI patch in config.plist:

```
<key>DSDT</key>
<dict>
	<key>Patches</key>
	<array>
		<dict>
			<key>Comment</key>
			<string>Rename HDAS to HDEF</string>
			<key>Find</key>
			<data>SERBUw==</data>
			<key>Replace</key>
			<data>SERFRg==</data>
		</dict>
	</array>
</dict>
```

Special thanks to [/u/marotte](https://www.reddit.com/u/marotte/) for catching my mistake with the patch (it wasn't originally for HDAS->HDEF).

Step 1: Layout ID Injection

This can accomplished in several different ways. You can edit Clover's config.plist to do so, or do edits to your DSDT. Make sure you pick the right layout ID for your setup.

Supported layout IDs

Look at the [AppleALC wiki](https://github.com/vit9696/AppleALC/wiki/Supported-codecs) for the supported layout IDs for each codec.

Clover config.plist editing

Edit the Inject key under Devices/Audio in config.plist. If it doesn't exist, paste it in. It is recommended that you use a propertly list editor such as Xcode & PlistEdit Pro. I would advise against using Clover Configurator.

```
<key>Devices</key>
<dict>
	<key>Audio</key>
	<dict>
		<key>Inject</key>
		<integer>1</integer>
	</dict>
</dict>
```

Replace 1 with the layout ID you wish to use.

Step 2: Download and Install AppleALC.kext

Download the latest version of AppleALC from [the GitHub Releases page](https://github.com/vit9696/AppleALC/releases). Mount your EFI partition, and copy it to Clover/kexts/$OS_VERSION or Clover/kexts/Other.

If you followed this guide correctly, you should now have working audio after a reboot. You can safely enable System Integrity Protection (SIP) as long as you are injecting your kexts from the EFI partition.