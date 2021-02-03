> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [superpowered.com](https://superpowered.com/androidaudiopathlatency)

[For practical tips on dealing with latency, Android developers please also see [Android Audio Low Latency Primer.](https://superpowered.com/android-audio-low-latency-primer)]

[UPDATED AS OF JUNE 2016: Please see [Android's 10 ms Problem? SOLVED.](https://superpowered.com/superpowered-android-media-server)]

[UPDATED AS OF JANUARY 2016: [Rebooting Android's 10 Millisecond Problem: Audio Latency Improvements in Android 6.0 Marshmallow](https://superpowered.com/android-marshmallow-latency)]

Many mobile apps that are critically dependent on low latency audio functionality such as some games, synthesizers, DAWs (Digital Audio Workstations), interactive audio apps and virtual instrument apps, and the coming wave of virtual reality apps, all of which thrive on Apple's platform (App Store + iOS devices) --- and generate big revenues for App Store and iOS developers are **largely non-existent** on Android.

**Android Audio's 10 Millisecond Problem**, a little understood yet extremely difficult technical challenge with enormous ramifications, **prevents these sorts of revenue producing apps from performing in an acceptable manner** and even being published (!) on Android at this point in time.

Startups and developers are unwilling to port and publish otherwise successful iOS apps (with ~10 ms audio latency needs) on Android for fear of degraded audio performance resulting in negative word-of-mouth and a hit to their professional reputation and brand.

Consumers lose because have a strong desire to buy such apps on Android, as shown by revenue data on iOS, and currently, are unable to do so. One can appreciate the scale of this problem/opportunity when one takes into account the so-called 'next billion' consumers who will be 'mobile-only'.

We want to solve this. This explainer provides an easily understood overview of the Android 10 Millisecond Problem with actual latency data from the Google Nexus 9.

Contents
--------

How Android's 10 Millisecond Problem and Android Audio Path Latency Impacts App Developers and Android OEMs
-----------------------------------------------------------------------------------------------------------

Even though music apps make up only 3% of all downloads in the iOS App Store, the Music app category is the 3rd highest revenue generating app category after Games and Social Networking. Which suggests that music apps monetize disproportionately well on platforms that offer low latency performance such as the App Store/iOS devices.

![](https://superpowered.com/images/AppAnnieDownloadsandRevenue.jpg)

On Android, it is a different story. In the Google Play store, the Music category is not even a top five revenue producing app category.

The overwhelming majority of Android devices suffer from too high audio latency, preventing developers from building apps that would satisfy consumer demand on Android.

As such, Google and Android app developers are leaving billions of dollars on the table for Apple and iOS developers because of Android's 10 Millisecond Problem.

For the purposes of this explainer, roundtrip audio latency is simply the difference in time between when an audio input is introduced into a mobile device, undergo some sort of needed processing, and exits the same device. As any musician will tell you, we as humans are most comfortable with latencies of ~10 milliseconds. Anything significantly higher tends to disturb us.

Most Android apps have more than 100 ms of audio output latency, and more than 200 ms of round-trip (audio input to audio output) latency. To give you a quick example from the Oscar winning film Whiplash, it’s like the drummer is dragging by a half beat behind the band!

Some specific examples on how audio related applications suffer from roundtrip audio latency greater than ~10 ms:

*   Music instruments apps, audio effect apps: musicians cannot play together on stage, as the performer using an Android device will be half beat behind the others. It’s not even usable for practicing.
*   DJs can not perform beat-matching, as their pre-listening signal in their headphones is far behind the master signal playing for the audience. Applying effects like a loop roll or echo is very hard too.
*   Games: sound effects, such as explosions or gun sounds lag behind by a few frames. Game audio is then “detached” from visuals, making for poor user-experience, preventing immersive gaming experiences.
*   VOIP apps, such as Skype: if both users are using a high latency Android phone, the overall audio latency is higher than the network latency. In other words, it takes more time for audio to “flow” through Android, than data packets to be transferred between continents.
*   Virtual reality (VR): when the viewer turns his head, the audio “follows” too late, destroying the 3D audio experience. Check the Paul McCartney Google Cardboard app for an example. Google is on the verge of leaving billions in revenue in VR opportunities for Apple.

In order to educate and inform tech industry leaders, app developers, technologists, product managers, executives, journalists, entrepreneurs, musicians, gamers and investors about the scope and ramifications of Android's 10 Millisecond Problem, one whose existence that no one benefits from, we at Superpowered have developed the explainer you are reading right now to provide an easily digestible overview of the entire Android audio chain and potential bottlenecks.

Our goal is that we rally and to unite around this challenge of 10 ms roundtrip audio latency on Android, and moreover, transform it into an opportunity that fosters innovation, better user-experiences and benefits Google Play customers, Android developers, Android OEMs and the entire Android ecosystem.

Notes about Audio Latency
-------------------------

Digital audio latency measurement has two useful measurement units:

*   Millisecond (ms): the 1/1000 of one second. Most latency is referred to with this measurement unit and happens in this time scale.
*   Sample (or frame): represents one discrete digital point (a number) in the audio stream. Sampling is how software converts a continuous signal like a sound wave to a sequence of samples. Samples are independent of the number of audio channels. For a one channel signal, one sample means one number. One sample for a two channel signal means two numbers and so on.

We calculate the audio signal flow’s overall latency using the very best case scenario:

*   Audio in Android’s native layer (Android NDK) is set up using Google’s low latency recommendations. Unfortunately, most Android applications do not follow Google’s low latency recommendations.
*   On-device Android is configured appropriately and can make use of the “Fast Mixer” path for both audio input and output. Apart from the most recent Nexus models, most other manufacturers do not configure Android to support Fast Mixer, hence round-trip latency is significantly higher on those devices. Please see [Superpowered’s Mobile Audio Census and Latency Test App](https://superpowered.com/latency) for latency measurement data on many popular Android devices.

Android 5.0 Lollipop Audio Path Latency Explanation in Plain English
--------------------------------------------------------------------

**Analog audio input** _↓_

**Analog audio input**

There may be several different analog components, such as a pre-amplifier for the built-in microphone. These analog components can be considered as "zero latency" in this case, because their true latency is typically magnitudes below 1 ms.

_Latency: 0_

**ADC** _↓_1 ms

**Analog to digital conversion (ADC)**

The audio chip measures the incoming audio stream in predefined intervals and converts every measurement to a number. This predefined interval is called the sampling rate, measured in Hz. Our [Mobile Audio Census and Latency Test App](https://superpowered.com/latency) shows that the 48000 Hz is the native sample rate for most audio chips on Android and iOS devices, meaning that the audio stream is sampled 48000 times in every second.

Because ADC implementations often contain an oversampling filter inside, a rule of thumb is to attribute 1 ms latency for the ADC step.

Now that the audio stream has been digitized, from this point forward the audio stream is now digital audio. Digital audio almost never travels one-by-one, but rather, in chunks, called "buffers" or "periods".

_Latency: 1 ms_

**Bus** _↓_1-6 ms

**Bus transfer from the audio chip to the audio driver**

The audio chip has several tasks. It handles ADC and DAC, switches between or mixes several inputs and outputs, applies volume, etc. It also "groups" the discrete digital audio samples into buffers and handles the transfer of these buffers to the operating system.

The audio chip is connected to the CPU with a bus, such as USB, PCI, Firewire, etc. Every bus has its own transfer latency depending on its internal buffer sizes and buffer counts. The latency here ranges from 1 ms (audio chip on an internal system bus) to 6 ms (USB sound card with conservative USB bus settings) typically.

_Latency: 1-6 ms_

**Audio driver** _↓_1+ period

**Audio driver (ALSA, OSS, etc.)**

The audio driver receives the incoming audio into a ring buffer in "bus buffer size" steps using the audio chip's native sample rate, 48000 Hz in most cases.

This ring buffer plays an essential part in smoothing bus transfer jitter ("roughness"), and "connects" the bus transfer buffer size to the operating system audio stack's buffer size. Consuming data from the ring buffer happens in the operating system audio stack's buffer size, so it naturally adds some latency.

Android runs "on top" of Linux, and most Android devices use the most popular Linux audio driver system, ALSA (Advanced Linux Sound Architecture). ALSA handles the ring buffer like this:

*   Audio is consumed from the ring buffer in "period size" steps.
*   The ring buffer's size is a multiple of the"period size".

For example:

*   Period size = 480 samples.
*   Period count = 2.
*   The ring buffer's size is 480x2 = 960 samples.
*   Audio input is received into one period (480 samples), while the audio stack reads/processes the other period (480 samples).
*   Latency = 1 period, 480 samples. It equals to 10 ms at 48000 Hz.

<table><tbody><tr><td colspan="2">ring buffer (960 samples)</td></tr><tr><td>period (480 samples)</td><td>period (480 samples)</td></tr></tbody></table>

A common period count is 2, but some systems may go higher..

_Latency: one or more periods_

**HAL** _↓_0+ samples

**Android audio Hardware Abstraction Layer (HAL)**

The HAL acts as a middleman between the Android media server and the Linux audio driver. HAL implementations are provided by the mobile device's manufacturer upon"porting" Android onto the device.

Implementations are open, vendors are free to create any kind of HAL code. Communication with the media server happens using predefined structures. The media server loads the HAL and asks to create input or output streams with optional preferred parameters such as the sample rate, buffer size or audio effects.

Note: The HAL may or may not perform according to the parameters and the media server must "adapt" to the HAL.

The typical HAL implementation is tinyALSA, which is used to communicate with the ALSA audio driver. Some vendors put closed source code here to implement audio features they feel important.

After analyzing the code of a number of open source HAL implementations in the Android source repository, we found a few quirks adding significant amount of latency and CPU load unnecessarily to the audio path due strange configurations and poor coding.

A good HAL implementation should not add any latency.

_Latency: 0 or more samples_

**Audio Flinger** _↓_1+ period

**Audio Flinger**

The Android media server consists of two services:

*   The AudioPolicy service handles audio session and permission handling, such as enabling access to the microphone or interrupts on calls. It's very similar to iOS’ audio session handling.
*   The AudioFlinger service handles the digital audio streams.

Audio Flinger creates a RecordThread, which acts as a middleman between an application and the audio driver. Its basic job is:

*   Obtaining the next input audio buffer from the driver's ring buffer using the Android HAL.
*   Resampling the buffer if the application requests different sample rate than the native sample rate.
*   Performing additional buffering if the application requests different buffer size than the native period size.

Audio Flinger has a "fast mixer" path, if Android is configured that way. If a user application is using native (Android NDK) code and sets up an audio buffer queue with the native hardware sample rate and period size, no resampling, additional buffering or mixing ("MixerThread") will happen in this step.

The RecordThread works with a "push" method, without any strict synchronization to the audio driver. It tries to make an "educated guess" when to wake up and run, but the "push" method is way more sensitive to dropouts. Low latency systems always use the "pull" method, where the audio driver "dictates" audio i/o through the entire audio chain. It's clear that when Android OS was initially conceived, designed and developed, low latency audio was not a priority.

_Latency: 1 period (best case scenario)_

**Binder** _↓_

**Binder**

Shared memory in Android's main inter-process communication system is used to transfer the audio buffers between Audio Flinger and the user application. It's the heart of Android, used everywhere internally in Android.

_Latency: 0_

**AudioRecord** _↓_0+ samples

**AudioRecord**

We are in the user application's process now. AudioRecord implements the application side of the audio input. This is a client library feature accessible via OpenSL ES for example.

AudioRecord runs a thread to periodically acquire a new buffer from Audio Flinger, with the "push" philosophy described at Audio Flinger. It doesn't add latency to the audio path if the developer sets it to work with only one buffer.

_Latency: 0+ samples_

**User Application** _↓_>1 period

**User Application**

Finally, the audio input reaches its destination, the user application.

Because the input and output threads are not the same, a user application must implement a ring buffer between the threads. Its size is 2 periods minimum (1 for audio input and 1 for audio output), but poorly written applications often use brute force and use more periods to solve CPU bottlenecks.

From this point, we are we start traveling back out with some audio output.

_Latency: more than 1 period, near 2 typically (best case scenario)_

**AudioTrack** _↓_0+ samples

**AudioTrack**

AudioTrack Implements the user application's side of the audio output. This is a client library feature accessible via OpenSL ES for example. It runs a thread to periodically send the next audio buffer to Audio Flinger. After Android 4.4.4, AudioTrack doesn't add latency to the audio path as it can be set up to use one buffer only.

_Latency: 0+ samples_

**Binder** _↓_

**Binder**

Same as for audio input.

_Latency: 0_

**Audio Flinger** _↓_1+ period

**Audio Flinger**

Creates a PlaybackThread, which works as the inverse of the RecordThread described at audio input.

_Latency: 1 period (best case scenario)_

**HAL** _↓_0+ samples

**Android audio HAL**

Same as for audio input.

_Latency: 0 or more samples_

**Audio driver** _↓_1+ period

**Audio driver (ALSA, OSS, etc.)**

Audio output in the audio driver works identically to the audio input and uses a ring buffer too.

_Latency: one or more periods_

**Bus** _↓_1-6 ms

**Bus transfer from the audio driver to the audio chip**

Similar to the audio input's bus transfer, the latency here ranges from 1 ms to 6 ms typically.

_Latency: 1-6 ms_

**DAC** _↓_1 ms

**Digital to analog conversion (DAC)**

The inverse of ADC, digital audio is "converted" back to analog in this point. For the same reasons at ADC, a rule of thumb is to assume 1 ms of latency for DAC.

_Latency: 1 ms_

**Analog audio output**

**Analog audio output**

The DAC's output signal is analog audio, but it needs additional components to drive connected devices, such as headphones. Similar to the analog audio input, the analog components can be considered to be"zero latency".

_Latency: 0_

Android Audio Path Latency Animation
------------------------------------

![](https://superpowered.com/images/Android-Audio-Path-Latency-Superpowered-Audio700px.gif)

Want to embed this image? Copy the following code:

`<a href="https://superpowered.com/androidaudiopathlatency"><img src="http://bit.ly/1I9MKxo" alt="Android 5.0 Lollipop Audio Path Latency" title="Android 5.0 Lollipop Audio Path Latency" /></a><br />Learn more about <a href="https://superpowered.com/androidaudiopathlatency">Android's 10 Millisecond Problem</a>.`

Android Audio Path Latency Case study: Google Nexus 9
-----------------------------------------------------

To date, the Google Nexus 9 performs best in Android round-trip audio latency measurement tests.

The best result is 35 ms using a USB sound card or a special audio dongle directly connecting the headphone connector's mic input and output, to disable the built-in microphone array's noise canceling/feedback destroying feature which adds about ~13 ms of additional latency.

So, using the same model as above, let’s decompose the 35 ms best-case round-trip audio latency of Google Nexus 9:

<table><caption>How the 35 ms round-trip latency of Google Nexus 9 comes up?</caption><tbody><tr><th>Component</th><th>Samples</th><th>Ms</th></tr><tr><td>ADC</td><td></td><td>1</td></tr><tr><td>Bus</td><td></td><td>1</td></tr><tr><td>ALSA audio driver</td><td>256</td><td>5.3</td></tr><tr><td>Audio Flinger</td><td>256</td><td>5.3</td></tr><tr><td>User Application's Ring Buffer</td><td>512</td><td>10.6</td></tr><tr><td>Audio Flinger</td><td>256</td><td>5.3</td></tr><tr><td>ALSA audio driver</td><td>256</td><td>5.3</td></tr><tr><td>Bus</td><td></td><td>1</td></tr><tr><td>DAC</td><td></td><td>1</td></tr><tr><td colspan="2">Result:&nbsp;</td><td>35.8</td></tr></tbody></table>

About Superpowered
------------------

Our mission is to extend the makers’ creative and productive capabilities – allowing them to create and make things real – profoundly shaping them, the builders, to build things that weren’t possible without Superpowered audio technology.

To that end, we are building technology, traversing the audio stack, that will solve Android's 10 Millisecond Problem. In the meantime, the Superpowered Audio SDK for Android and iOS is:

*   Cross-platform: Developers can use and re-use the same code on Android, iOS and OSX.
*   Super fast: it has the highest performance audio DSP on mobile devices, providing desktop-grade processing and pro audio quality. The reduced CPU load improves battery life and smoothness of all applications.
*   Works great for both "push" and "pull" audio stacks, and even offline processing.
*   0 latency: Superpowered features and processing don't add any latency. User applications built with the Superpowered Audio SDK easily run with the lowest audio latency possible on any iOS or Android device.

We'd love to hear from you. Please email us with your suggestions, comments and questions. [Hello@Superpowered.com](mailto:hello@superpowered.com)

**Thanks for reading.**

_Gabor (@szantog) and Patrick (@Pv), founders of Superpowered_

PS Please join the great conversations about Android's 10 ms problem at [Reddit](http://www.reddit.com/submit?url=http://superpowered.com/androidaudiopathlatency/&title=Android%E2%80%99s%2010%20Millisecond%20Problem:%20The%20Android%20Audio%20Path%20Latency%20Explainer "Reddit Android") and [Hacker News](https://news.ycombinator.com/item?id=9386994 "Hacker News").

*   Android audio latency
*   Android latency
*   AudioFlinger
*   Android Media Server
*   Low latency audio