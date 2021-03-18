> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [realpython.com](https://realpython.com/python-scipy-fft/)

The **Fourier transform** is a powerful tool for analyzing **signals** and is used in everything from audio processing to image compression. SciPy provides a mature 成熟implementation in its `scipy.fft` module, and in this tutorial, you’ll learn how to use it.

The `scipy.fft` module may look intimidating 令人望而生畏at first since there are many functions, often with similar names, and the [documentation](https://docs.scipy.org/doc/scipy/reference/tutorial/fft.html) uses a lot of technical terms without explanation. The good news is that you only need to understand a few core concepts to start using the module.

Don’t worry if you’re not comfortable with math! You’ll get a feel for the algorithm through concrete 实在examples, and there will be links to further resources if you want to dive into the equations方程式. For a visual introduction to how the Fourier transform works, you might like [3Blue1Brown’s video](https://www.youtube.com/watch?v=spUNpyF58BY).

**In this tutorial, you’ll learn:**

*   How and when to use the **Fourier transform**
*   How to select the correct function from **`scipy.fft`** for your use case
*   How to view and modify the **frequency spectrum** of a signal
*   Which different **transforms** are available in `scipy.fft`

If you’d like a summary of this tutorial to keep after you finish reading, then download the cheat sheet below. It has explanations of all the functions in the `scipy.fft` module as well as a breakdown of the different types of transform that are available:

The `scipy.fft` Module[](#the-scipyfft-module "Permanent link")
---------------------------------------------------------------

The Fourier transform is a crucial 重要的tool in many applications, especially in scientific computing and data science. As such, SciPy has long provided an implementation of it and its related transforms. Initially, SciPy provided the `scipy.fftpack` module, but they have since updated their implementation and moved it to the `scipy.fft` module.

SciPy is packed full of functionality. For a more general introduction to the library, check out [Scientific Python: Using SciPy for Optimization](https://realpython.com/python-scipy-cluster-optimize/).

### Install SciPy and Matplotlib[](#install-scipy-and-matplotlib "Permanent link")

Before you can get started, you’ll need to install SciPy and [Matplotlib](https://realpython.com/python-matplotlib-guide/). You can do this one of two ways:

1.  **Install with Anaconda:** Download and install the [Anaconda Individual Edition](https://www.anaconda.com/products/individual). It comes with SciPy and Matplotlib, so once you follow the steps in the installer, you’re done!
    
2.  **Install with `pip`:** If you already have [`pip`](https://realpython.com/what-is-pip/) installed, then you can install the libraries with the following command:
    
    ```
    $ python -m pip install -U scipy matplotlib
    ```
    

You can verify the installation worked by [typing `python` in your terminal](https://realpython.com/interacting-with-python/#using-the-python-interpreter-interactively) and running the following code:

>>>

```
>>> import scipy, matplotlib
>>> print(scipy.__file__)
/usr/local/lib/python3.6/dist-packages/scipy/__init__.py
>>> print(matplotlib.__file__)
/usr/local/lib/python3.6/dist-packages/matplotlib/__init__.py
```

This code imports SciPy and Matplotlib and prints the location of the modules. Your computer will probably show different paths, but as long as it prints a path, the installation worked.

SciPy is now installed! Now it’s time to take a look at the differences between `scipy.fft` and `scipy.fftpack`.

### `scipy.fft` vs `scipy.fftpack`[](#scipyfft-vs-scipyfftpack "Permanent link")

When looking at the SciPy documentation, you may come across two modules that look very similar:

1.  `scipy.fft`
2.  `scipy.fftpack`

The `scipy.fft` module is newer and should be preferred over `scipy.fftpack`. You can read more about the change in the [release notes for SciPy 1.4.0](https://docs.scipy.org/doc/scipy/reference/release.1.4.0.html#scipy-fft-added), but here’s a quick summary:

*   `scipy.fft` has an improved API.
*   `scipy.fft` enables using [multiple workers](https://realpython.com/intro-to-python-threading/), which can provide a speed boost in some situations.
*   `scipy.fftpack` is considered legacy, and SciPy recommends using `scipy.fft` instead.

Unless you have a good reason to use `scipy.fftpack`, you should stick with `scipy.fft`.

### `scipy.fft` vs `numpy.fft`[](#scipyfft-vs-numpyfft "Permanent link")

SciPy’s [fast Fourier transform (FFT)](https://en.wikipedia.org/wiki/Fast_Fourier_transform) implementation contains more features and is more likely to get bug fixes than NumPy’s implementation. If given a choice, you should use the SciPy implementation.

NumPy maintains an FFT implementation for backward compatibility even though the authors believe that functionality like Fourier transforms is best placed in SciPy. See the [SciPy FAQ](https://www.scipy.org/scipylib/faq.html#what-is-the-difference-between-numpy-and-scipy) for more details.

The Fourier Transform[](#the-fourier-transform "Permanent link")
----------------------------------------------------------------

[Fourier analysis](https://en.wikipedia.org/wiki/Fourier_analysis) is a field that studies how a **mathematical function** can be decomposed into a series of simpler **trigonometric functions**. The Fourier transform is a tool from this field for decomposing a function into its component frequencies.

Okay, that definition is pretty dense. For the purposes of this tutorial, the Fourier transform is a tool that allows you to take a signal and see the power of each frequency in it. Take a look at the important terms in that sentence:

*   A **signal** is information that changes over time. For example, audio, video, and voltage traces are all examples of signals.
*   A **frequency** is the speed at which something repeats. For example, clocks tick at a frequency of one hertz (Hz), or one repetition per second.
*   **Power**, in this case, just means the strength of each frequency.

The following image is a visual demonstration of frequency and power on some [sine waves](https://en.wikipedia.org/wiki/Sine_wave):

[![](https://files.realpython.com/media/freqpower2.f3dbf5bddc29.png)](https://files.realpython.com/media/freqpower2.f3dbf5bddc29.png)

The peaks of the **high-frequency** sine wave are closer together than those of the **low-frequency** sine wave since they repeat more frequently. The **low-power** sine wave has smaller peaks than the other two sine waves.

To make this more concrete, imagine you used the Fourier transform on a recording of someone playing three notes on the piano at the same time. The resulting **frequency spectrum** would show three peaks, one for each of the notes. If the person played one note more softly than the others, then the power of that note’s frequency would be lower than the other two.

Here’s what that piano example would look like visually:

[![](https://files.realpython.com/media/pianofreqblue.ff266a14503f.png)](https://files.realpython.com/media/pianofreqblue.ff266a14503f.png)

The highest note on the piano was played quieter than the other two notes, so the resulting frequency spectrum for that note has a lower peak.

### Why Would You Need the Fourier Transform?[](#why-would-you-need-the-fourier-transform "Permanent link")

The Fourier transform is useful in many applications. For example, [Shazam](https://www.shazam.com/company) and other music identification services use the Fourier transform to identify songs. JPEG compression uses a variant of the Fourier transform to remove the high-frequency components of images. [Speech recognition](https://realpython.com/python-speech-recognition/) uses the Fourier transform and related transforms to recover the spoken words from raw audio.

In general, you need the Fourier transform if you need to look at the frequencies in a signal. If working with a signal in the time domain is difficult, then using the Fourier transform to move it into the frequency domain is worth trying. In the next section, you’ll look at the differences between the time and frequency domains.

### Time Domain vs Frequency Domain[](#time-domain-vs-frequency-domain "Permanent link")

Throughout the rest of the tutorial, you’ll see the terms **time domain** and **frequency domain**. These two terms refer to two different ways of looking at a signal, either as its component frequencies or as information that varies over time.

In the time domain, a signal is a wave that varies in amplitude (y-axis) over time (x-axis). You’re most likely used to seeing graphs in the time domain, such as this one:

[![](https://files.realpython.com/media/timedomain.cc67471385a2.png)](https://files.realpython.com/media/timedomain.cc67471385a2.png)

This is an image of some audio, which is a **time-domain** signal. The horizontal axis represents time, and the vertical axis represents amplitude.

In the frequency domain, a signal is represented as a series of frequencies (x-axis) that each have an associated power (y-axis). The following image is the above audio signal after being Fourier transformed:

[![](https://files.realpython.com/media/freqdomain.fdeba267dfda.png)](https://files.realpython.com/media/freqdomain.fdeba267dfda.png)

Here, the audio signal from before is represented by its constituent frequencies. Each frequency along the bottom has an associated power, producing the spectrum that you see.

For more information on the frequency domain, check out the [DeepAI glossary entry](https://deepai.org/machine-learning-glossary-and-terms/frequency-domain).

### Types of Fourier Transforms[](#types-of-fourier-transforms "Permanent link")

The Fourier transform can be subdivided into different types of transform. The most basic subdivision is based on the kind of data the transform operates on: continuous functions or discrete functions. This tutorial will deal with only the **discrete Fourier transform (DFT)**.

You’ll often see the terms DFT and FFT used interchangeably, even in this tutorial. However, they aren’t quite the same thing. The **fast Fourier transform (FFT)** is an algorithm for computing the discrete Fourier transform (DFT), whereas the DFT is the transform itself.

Another distinction that you’ll see made in the `scipy.fft` library is between different types of input. `fft()` accepts complex-valued input, and `rfft()` accepts real-valued input. Skip ahead to the section [Using the Fast Fourier Transform (FFT)](#using-the-fast-fourier-transform-fft) for an explanation of complex and real numbers.

Two other transforms are closely related to the DFT: the **discrete cosine transform (DCT)** and the **discrete sine transform (DST)**. You’ll learn about those in the section [The Discrete Cosine and Sine Transforms](#the-discrete-cosine-and-sine-transforms).

Practical Example: Remove Unwanted Noise From Audio[](#practical-example-remove-unwanted-noise-from-audio "Permanent link")
---------------------------------------------------------------------------------------------------------------------------

To help build your understanding of the Fourier transform and what you can do with it, you’re going to filter some audio. First, you’ll create an audio signal with a high pitched buzz in it, and then you’ll remove the buzz using the Fourier transform.

### Creating a Signal[](#creating-a-signal "Permanent link")

Sine waves are sometimes called **pure tones** because they represent a single frequency. You’ll use sine waves to generate the audio since they will form distinct peaks in the resulting frequency spectrum.

Another great thing about sine waves is that they’re straightforward to generate using NumPy. If you haven’t used NumPy before, then you can check out [What Is NumPy?](https://realpython.com/tutorials/numpy/)

Here’s some code that generates a sine wave:

```
import numpy as np
from matplotlib import pyplot as plt

SAMPLE_RATE = 44100  # Hertz
DURATION = 5  # Seconds

def generate_sine_wave(freq, sample_rate, duration):
    x = np.linspace(0, duration, sample_rate * duration, endpoint=False)
    frequencies = x * freq
    # 2pi because np.sin takes radians
    y = np.sin((2 * np.pi) * frequencies)
    return x, y

# Generate a 2 hertz sine wave that lasts for 5 seconds
x, y = generate_sine_wave(2, SAMPLE_RATE, DURATION)
plt.plot(x, y)
plt.show()
```

After you [import](https://realpython.com/python-import/) NumPy and Matplotlib, you define two constants:

1.  **`SAMPLE_RATE`** determines how many data points the signal uses to represent the sine wave per second. So if the signal had a sample rate of 10 Hz and was a five-second sine wave, then it would have `10 * 5 = 50` data points.
2.  **`DURATION`** is the length of the generated sample.

Next, you define a function to generate a sine wave since you’ll use it multiple times later on. The function takes a frequency, `freq`, and then [returns](https://realpython.com/python-return-statement/) the `x` and `y` values that you’ll use to plot the wave.

The x-coordinates of the sine wave are evenly spaced between `0` and `DURATION`, so the code uses NumPy’s [`linspace()`](https://numpy.org/doc/stable/reference/generated/numpy.linspace.html) to generate them. It takes a start value, an end value, and the number of samples to generate. Setting `endpoint=False` is important for the Fourier transform to work properly because it assumes a signal is [periodic](http://msp.ucsd.edu/techniques/latest/book-html/node171.html).

[`np.sin()`](https://numpy.org/doc/stable/reference/generated/numpy.sin.html) calculates the values of the sine function at each of the x-coordinates. The result is multiplied by the frequency to make the sine wave oscillate at that frequency, and the product is multiplied by 2π to convert the input values to [radians](https://en.wikipedia.org/wiki/Radian).

After you define the function, you use it to generate a two-hertz sine wave that lasts five seconds and plot it using Matplotlib. Your sine wave plot should look something like this:

[![](https://files.realpython.com/media/firstsin.edca17cbaa5b.png)](https://files.realpython.com/media/firstsin.edca17cbaa5b.png)

The x-axis represents time in seconds, and since there are two peaks for each second of time, you can see that the sine wave oscillates twice per second. This sine wave is too low a frequency to be audible, so in the next section, you’ll generate some higher-frequency sine waves, and you’ll see how to mix them.

### Mixing Audio Signals[](#mixing-audio-signals "Permanent link")

The good news is that mixing audio signals consists of just two steps:

1.  Adding the signals together
2.  **Normalizing** the result

Before you can mix the signals together, you need to generate them:

```
_, nice_tone = generate_sine_wave(400, SAMPLE_RATE, DURATION)
_, noise_tone = generate_sine_wave(4000, SAMPLE_RATE, DURATION)
noise_tone = noise_tone * 0.3

mixed_tone = nice_tone + noise_tone
```

There’s nothing new in this code example. It generates a medium-pitch tone and a high-pitch tone assigned to the [variables](https://realpython.com/python-variables/) `nice_tone` and `noise_tone`, respectively. You’ll use the high-pitch tone as your unwanted noise, so it gets multiplied by `0.3` to reduce its power. The code then adds these tones together. Note that you use the underscore (`_`) to discard the `x` values returned by `generate_sine_wave()`.

The next step is **normalization**, or scaling the signal to fit into the target format. Due to how you’ll store the audio later, your target format is a 16-bit integer, which has a range from -32768 to 32767:

```
normalized_tone = np.int16((mixed_tone / mixed_tone.max()) * 32767)

plt.plot(normalized_tone[:1000])
plt.show()
```

Here, the code scales `mixed_tone` to make it fit snugly into a 16-bit integer and then cast it to that data type using NumPy’s `np.int16`. Dividing `mixed_tone` by its maximum value scales it to between `-1` and `1`. When this signal is multiplied by `32767`, it is scaled between `-32767` and `32767`, which is roughly the range of `np.int16`. The code plots only the first `1000` samples so you can see the structure of the signal more clearly.

Your plot should look something like this:

[![](https://files.realpython.com/media/mixedsignal.ff1130345c3a.png)](https://files.realpython.com/media/mixedsignal.ff1130345c3a.png)

The signal looks like a distorted sine wave. The sine wave you see is the 400 Hz tone you generated, and the distortion is the 4000 Hz tone. If you look closely, then you can see the distortion has the shape of a sine wave.

To listen to the audio, you need to store it in a format that an audio player can read. The easiest way to do that is to use SciPy’s [wavfile.write](https://docs.scipy.org/doc/scipy/reference/io.html) method to store it in a WAV file. 16-bit integers are a standard data type for WAV files, so you’ll normalize your signal to 16-bit integers:

```
from scipy.io.wavfile import write

# Remember SAMPLE_RATE = 44100 Hz is our playback rate
write("mysinewave.wav", SAMPLE_RATE, normalized_tone)
```

This code will write to a file `mysinewave.wav` in the directory where you run your Python script. You can then listen to this file using any audio player or even [with Python](https://realpython.com/playing-and-recording-sound-python/). You’ll hear a lower tone and a higher-pitch tone. These are the 400 Hz and 4000 Hz sine waves that you mixed.

Once you’ve completed this step, you have your audio sample ready. The next step is removing the high-pitch tone using the Fourier transform!

### Using the Fast Fourier Transform (FFT)[](#using-the-fast-fourier-transform-fft "Permanent link")

It’s time to use the FFT on your generated audio. The FFT is an algorithm that implements the Fourier transform and can calculate a frequency spectrum for a signal in the time domain, like your audio:

```
from scipy.fft import fft, fftfreq

# Number of samples in normalized_tone
N = SAMPLE_RATE * DURATION

yf = fft(normalized_tone)
xf = fftfreq(N, 1 / SAMPLE_RATE)

plt.plot(xf, np.abs(yf))
plt.show()
```

This code will calculate the Fourier transform of your generated audio and plot it. Before breaking it down, take a look at the plot that it produces:

[![](https://files.realpython.com/media/firstfft.b1983096fafd.png)](https://files.realpython.com/media/firstfft.b1983096fafd.png)

You can see two peaks in the positive frequencies and mirrors of those peaks in the negative frequencies. The positive-frequency peaks are at 400 Hz and 4000 Hz, which corresponds to the frequencies that you put into the audio.

The Fourier transform has taken your complicated, wibbly signal and turned it into just the frequencies it contains. Since you put in only two frequencies, only two frequencies have come out. The negative-positive symmetry is a side effect of putting **real-valued input** into the Fourier transform, but you’ll hear more about that later.

In the first couple of lines, you import the functions from `scipy.fft` that you’ll use later and define a variable, `N`, that stores the total number of samples in the signal.

After this comes the most important section, calculating the Fourier transform:

```
yf = fft(normalized_tone)
xf = fftfreq(N, 1 / SAMPLE_RATE)
```

The code calls two very important functions:

1.  [**`fft()`**](https://docs.scipy.org/doc/scipy/reference/generated/scipy.fft.fft.html) calculates the transform itself.
    
2.  [**`fftfreq()`**](https://docs.scipy.org/doc/scipy/reference/generated/scipy.fft.fftfreq.html) calculates the frequencies in the center of each **bin** in the output of `fft()`. Without this, there would be no way to plot the x-axis on your frequency spectrum.
    

A **bin** is a range of values that have been grouped, like in a [histogram](https://realpython.com/python-histograms/). For more information on **bins**, see this [Signal Processing Stack Exchange question](https://dsp.stackexchange.com/questions/26927/what-is-a-frequency-bin). For the purposes of this tutorial, you can think of them as just single values.

Once you have the resulting values from the Fourier transform and their corresponding frequencies, you can plot them:

```
plt.plot(xf, np.abs(yf))
plt.show()
```

The interesting part of this code is the processing you do to `yf` before plotting it. You call [`np.abs()`](https://numpy.org/doc/stable/reference/generated/numpy.absolute.html) on `yf` because its values are **complex**.

A **complex number** is a [number](https://realpython.com/python-numbers/) that has two parts, a **real** part and an **imaginary** part. There are many reasons why it’s useful to define numbers like this, but all you need to know right now is that they exist.

Mathematicians generally write complex numbers in the form _a_ + _bi_, where _a_ is the real part and _b_ is the imaginary part. The _i_ after _b_ means that _b_ is an imaginary number.

**Note:** Sometimes you’ll see complex numbers written using _i_, and sometimes you’ll see them written using _j_, such as 2 + 3_i_ and 2 + 3_j_. The two are the same, but _i_ is used more by mathematicians, and _j_ more by engineers.

For more on complex numbers, take a look at [Khan Academy’s course](https://www.khanacademy.org/math/algebra2/x2ec2f6f830c9fb89:complex) or the [Maths is Fun page](https://www.mathsisfun.com/numbers/complex-numbers.html).

Since complex numbers have two parts, graphing them against frequency on a two-dimensional axis requires you to calculate a single value from them. This is where `np.abs()` comes in. It calculates √(a² + b²) for complex numbers, which is an overall magnitude for the two numbers together and importantly a single value.

**Note:** As an aside, you may have noticed that `fft()` returns a maximum frequency of just over 20 thousand Hertz, 22050Hz, to be exact. This value is exactly half of our sampling rate and is called the [Nyquist frequency](https://en.wikipedia.org/wiki/Nyquist_frequency).

It’s a fundamental concept in signal processing and means that your sampling rate has to be at least twice the highest frequency in your signal.

### Making It Faster With `rfft()`[](#making-it-faster-with-rfft "Permanent link")

The frequency spectrum that `fft()` outputted was reflected about the y-axis so that the negative half was a mirror of the positive half. This symmetry was caused by inputting **real numbers** (not complex numbers) to the transform.

You can use this symmetry to make your Fourier transform faster by computing only half of it. `scipy.fft` implements this speed hack in the form of [`rfft()`](https://numpy.org/doc/stable/reference/generated/numpy.fft.rfft.html).

The great thing about `rfft()` is that it’s a drop-in replacement for `fft()`. Remember the FFT code from before:

```
yf = fft(normalized_tone)
xf = fftfreq(N, 1 / SAMPLE_RATE)
```

Swapping in `rfft()`, the code remains mostly the same, just with a couple of key changes:

```
from scipy.fft import rfft, rfftfreq

# Note the extra 'r' at the front
yf = rfft(normalized_tone)
xf = rfftfreq(N, 1 / SAMPLE_RATE)

plt.plot(xf, np.abs(yf))
plt.show()
```

Since `rfft()` returns only half the output that `fft()` does, it uses a different function to get the frequency mapping, `rfftfreq()` instead of `fftfreq()`.

`rfft()` still produces complex output, so the code to plot its result remains the same. The plot, however, should look like the following since the negative frequencies will have disappeared:

[![](https://files.realpython.com/media/rfftplot.8e16850b2dde.png)](https://files.realpython.com/media/rfftplot.8e16850b2dde.png)

You can see that the image above is just the positive side of the frequency spectrum that `fft()` produces. `rfft()` never calculates the negative half of the frequency spectrum, which makes it faster than using `fft()`.

Using `rfft()` can be up to twice as fast as using `fft()`, but some input lengths are faster than others. If you know you’ll be working only with real numbers, then it’s a speed hack worth knowing.

Now that you have the frequency spectrum of the signal, you can move on to filtering it.

### Filtering the Signal[](#filtering-the-signal "Permanent link")

One great thing about the Fourier transform is that it’s reversible, so any changes you make to the signal in the frequency domain will apply when you transform it back to the time domain. You’ll take advantage of this to filter your audio and get rid of the high-pitched frequency.

**Warning:** The filtering technique demonstrated in this section isn’t suitable for real-world signals. See the section [Avoiding Filtering Pitfalls](#avoiding-filtering-pitfalls) for an explanation of why.

The values returned by `rfft()` represent the power of each frequency bin. If you set the power of a given bin to zero, then the frequencies in that bin will no longer be present in the resulting time-domain signal.

Using the length of `xf`, the maximum frequency, and the fact that the frequency bins are evenly spaced, you can work out the target frequency’s index:

```
# The maximum frequency is half the sample rate
points_per_freq = len(xf) / (SAMPLE_RATE / 2)

# Our target frequency is 4000 Hz
target_idx = int(points_per_freq * 4000)
```

You can then set `yf` to `0` at indices around the target frequency to get rid of it:

```
yf[target_idx - 1 : target_idx + 2] = 0

plt.plot(xf, np.abs(yf))
plt.show()
```

Your code should produce the following plot:

[![](https://files.realpython.com/media/freqremoved.51f079cdfcf8.png)](https://files.realpython.com/media/freqremoved.51f079cdfcf8.png)

Since there’s only one peak, it looks like it worked! Next, you’ll apply the inverse Fourier transform to get back to the time domain.

### Applying the Inverse FFT[](#applying-the-inverse-fft "Permanent link")

Applying the inverse FFT is similar to applying the FFT:

```
from scipy.fft import irfft

new_sig = irfft(yf)

plt.plot(new_sig[:1000])
plt.show()
```

Since you are using `rfft()`, you need to use `irfft()` to apply the inverse. However, if you had used `fft()`, then the inverse function would have been `ifft()`. Your plot should now look like this:

[![](https://files.realpython.com/media/cleansig.a30a1b9f6591.png)](https://files.realpython.com/media/cleansig.a30a1b9f6591.png)

As you can see, you now have a single sine wave oscillating at 400 Hz, and you’ve successfully removed the 4000 Hz noise.

Once again, you need to normalize the signal before writing it to a file. You can do it the same way as last time:

```
norm_new_sig = np.int16(new_sig * (32767 / new_sig.max()))

write("clean.wav", SAMPLE_RATE, norm_new_sig)
```

When you listen to this file, you’ll hear that the annoying noise has gone away!

### Avoiding Filtering Pitfalls[](#avoiding-filtering-pitfalls "Permanent link")

The above example is more for educational purposes than real-world use. Replicating the process on a real-world signal, such as a piece of music, [could introduce more buzz than it removes](https://dsp.stackexchange.com/questions/6220/why-is-it-a-bad-idea-to-filter-by-zeroing-out-fft-bins).

In the real world, you should filter signals using the filter design functions in the [`scipy.signal`](https://docs.scipy.org/doc/scipy/reference/signal.html) package. Filtering is a complex topic that involves a lot of math. For a good introduction, take a look at [The Scientist and Engineer’s Guide to Digital Signal Processing](https://www.dspguide.com/ch14.htm).

The Discrete Cosine and Sine Transforms[](#the-discrete-cosine-and-sine-transforms "Permanent link")
----------------------------------------------------------------------------------------------------

A tutorial on the `scipy.fft` module wouldn’t be complete without looking at the [discrete cosine transform (DCT)](https://en.wikipedia.org/wiki/Discrete_cosine_transform) and the [discrete sine transform (DST)](https://en.wikipedia.org/wiki/Discrete_sine_transform). These two transforms are closely related to the Fourier transform but operate entirely on real numbers. That means they take a real-valued function as an input and produce another real-valued function as an output.

SciPy implements these transforms as [`dct()`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.fft.dct.html#scipy.fft.dct) and [`dst()`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.fft.dst.html#scipy.fft.dst). The `i*` and `*n` variants are the inverse and _n_-dimensional versions of the functions, respectively.

The DCT and DST are a bit like two halves that together make up the Fourier transform. This isn’t quite true since the math is a lot more complicated, but it’s a useful mental model.

So if the DCT and DST are like halves of a Fourier transform, then why are they useful?

For one thing, they’re faster than a full Fourier transform since they effectively do half the work. They can be even faster than `rfft()`. On top of this, they work entirely in real numbers, so you never have to worry about complex numbers.

Before you can learn how to choose between them, you need to understand even and odd functions. **Even functions** are symmetrical about the y-axis, whereas **odd functions** are symmetrical about the origin. To imagine this visually, take a look at the following diagrams:

[![](https://files.realpython.com/media/evenandodd.8410a9717f96.png)](https://files.realpython.com/media/evenandodd.8410a9717f96.png)

You can see that the even function is symmetrical about the y-axis. The odd function is symmetrical about _y_ = -_x_, which is described as being **symmetrical about the origin**.

When you calculate a Fourier transform, you pretend that the function you’re calculating it on is infinite. The full Fourier transform (DFT) assumes the input function repeats itself infinitely. However, the DCT and DST assume the function is extended through symmetry. The DCT assumes the function is extended with even symmetry, and the DST assumes it’s extended with odd symmetry.

The following image illustrates how each transform imagines the function extends to infinity:

[![](https://files.realpython.com/media/funcextension.8794e9dc3154.png)](https://files.realpython.com/media/funcextension.8794e9dc3154.png)

In the above image, the DFT repeats the function as is. The DCT mirrors the function vertically to extend it, and the DST mirrors it horizontally.

Note that the symmetry implied by the DST leads to big jumps in the function. These are called **discontinuities** and [produce more high-frequency components](https://dsp.stackexchange.com/questions/46754/dct-vs-dst-for-image-compression) in the resulting frequency spectrum. So unless you know your data has odd symmetry, you should use the DCT instead of the DST.

The DCT is very commonly used. There are many [more examples](https://en.wikipedia.org/wiki/Discrete_cosine_transform), but the JPEG, MP3, and WebM standards all use the DCT.

Conclusion[](#conclusion "Permanent link")
------------------------------------------

The Fourier transform is a powerful concept that’s used in a variety of fields, from pure math to audio engineering and even finance. You’re now familiar with the **discrete Fourier transform** and are well equipped to apply it to filtering problems using the `scipy.fft` module.

**In this tutorial, you learned:**

*   How and when to use the **Fourier transform**
*   How to select the correct function from **`scipy.fft`** for your use case
*   What the differences are between the **time domain** and the **frequency domain**
*   How to view and modify the **frequency spectrum** of a signal
*   How using **`rfft()`** can speed up your code

In the last section, you also learned about the **discrete cosine transform** and the **discrete sine transform**. You saw what functions to call to use them, and you learned when to use one over the other.

If you’d like a summary of this tutorial, then you can download the cheat sheet below. It has explanations of all the functions in the `scipy.fft` module as well as a breakdown of the different types of transform that are available:

Keep exploring this fascinating topic and experimenting with transforms, and be sure to share your discoveries in the comments below!