> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_39516859/article/details/79834039)

1、Wave 文件
---------

  WAV 是 Microsoft 开发的一种声音文件格式，虽然它支持多种压缩格式，不过它通常被用来保存未压缩的声音数据（PCM 脉冲编码调制)。WAV 有三个重要的参数：声道数、取样频率和量化位数。

*   声道数：可以是单声道或者是双声道
*   采样频率：一秒内对声音信号的采集次数，常用的有 8kHz, 16kHz, 32kHz, 48kHz, 11.025kHz,  
    22.05kHz, 44.1kHz
*   量化位数：用多少 bit 表达一次采样所采集的数据，通常有 8bit、16bit、24bit 和 32bit 等几种

  常见的声音文件主要有两种，分别对应于单声道（11.025KHz 采样率、8Bit 的采样值）和双声道（44.1KHz 采样率、16Bit 的采样值）。  
  对于单声道声音文件，采样数据为八位的短整数（short int 00H-FFH）；而对于双声道立体声声音文件，每次采样数据为一个 16 位的整数（int），高八位和低八位分别代表左右两个声道。WAVE 文件数据块包含以脉冲编码调制（PCM）格式表示的样本。WAVE 文件是由样本组织而成的。在单声道 WAVE 文件中，声道 0 代表左声道，声道 1 代表右声道。在多声道 WAVE 文件中，样本是交替出现的。

2、写 Wave 文件
-----------

写 WAV 文件的方法和读类似：

```
# -*- coding: utf-8 -*-
import wave
import numpy as np
import scipy.signal as signal

framerate = 44100
time = 10

# 产生10秒44.1kHz的100Hz - 1kHz的频率扫描波
t = np.arange(0, time, 1.0/framerate)
wave_data = signal.chirp(t, 100, time, 1000, method='linear') * 10000
wave_data = wave_data.astype(np.short)

# 打开WAV文档
f = wave.open(r"E:\练习\音频信号处理\Spectrum_analysis\Intro.wav", "wb")

# 配置声道数、量化位数和取样频率
f.setnchannels(1)
f.setsampwidth(2)
f.setframerate(framerate)
# 将wav_data转换为二进制数据写入文件
f.writeframes(wave_data.tostring())
f.close()
```

10-12 行通过调用 scipy.signal 库中的 chrip 函数，产生长度为 10 秒、取样频率为 44.1kHz、100Hz 到 1kHz 的频率扫描波。由于 chrip 函数返回的数组为 float64 型，需要调用数组的 astype 方法将其转换为 short 型。

18-20 行分别设置输出 WAV 文件的声道数、量化位数和取样频率，当然也可以调用文件对象的 setparams 方法一次性配置所有的参数。最后 22 行调用文件的 writeframes 方法，将数组的内部的二进制数据写入文件。writeframes 方法会自动的更新 WAV 文件头中的长度信息 (nframes)，保证其和真正的数据数量一致。