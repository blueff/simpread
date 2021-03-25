> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [towardsdatascience.com](https://towardsdatascience.com/read-text-from-image-with-one-line-of-python-code-c22ede074cac)

[

![](https://miro.medium.com/fit/c/56/56/2*VmdbajrpX9nwOc9UtkV3Yg.png)

](https://medium.com/@radecicdario?source=post_page-----c22ede074cac--------------------------------)

Dealing with images is not a trivial task. To you, as a human, it’s easy to look at something and immediately know what is it you’re looking at. But computers don’t work that way.

Tasks that are too hard for you, like complex arithmetics, and math in general, is something that a computer chews without breaking a sweat. But here the **exact opposite applies** — tasks that are trivial to you, like recognizing is it cat or dog in an image are really hard for a computer. In a way, we are a perfect match. For now at least.

While image classification and tasks that involve some level of computer vision might require a good bit of code and a solid understanding, reading text from a somewhat well-formatted image turns out to be a one-liner in Python —and can be applied to so many real-life problems.

And in today’s post, I want to prove that claim. There will be some installation to go though, but it shouldn’t take much time. These are the libraries you’ll need:

*   OpenCV
*   PyTesseract

I don’t want to prolonge this intro part anymore, so why don’t we jump into the good stuff now.

Now, this library will only be used to load the images(s), you don’t actually need to have a solid understanding of it beforehand (although it might be helpful, you’ll see why).

According to the official documentation:

> OpenCV (Open Source Computer Vision Library) is an open source computer vision and machine learning software library. OpenCV was built to provide a common infrastructure for computer vision applications and to accelerate the use of machine perception in the commercial products. Being a BSD-licensed product, OpenCV makes it easy for businesses to utilize and modify the code.[1]

In a nutshell, you can use OpenCV to do any kind of **image transformations**, it’s fairly straightforward library.

If you don’t already have it installed, it’ll be just a single line in terminal:

```
pip install opencv-python
```

And that’s pretty much it. It was easy up until this point, but that’s about to change.

**_What the heck is this library?_** Well, according to Wikipedia:

> Tesseract is an optical character recognition engine for various operating systems. It is free software, released under the Apache License, Version 2.0, and development has been sponsored by Google since 2006.[2]

I’m sure there are more sophisticated libraries available now, but I’ve found this one working out pretty well. Based on my own experience, this library should be able to read text from any image, provided that the font isn’t some bulls*** that even you aren’t able to read.

If it can’t read from your image, spend more time playing around with OpenCV, applying various filters to make the text stand out.

Now the installation is a bit of a pain in the bottom. If you are on Linux it all boils down to a couple of **_sudo-apt get_** commands:

```
sudo apt-get update
sudo apt-get install tesseract-ocr
sudo apt-get install libtesseract-dev
```

I’m on Windows, so the process is a bit more tedious.

First, open up [THIS URL](https://github.com/UB-Mannheim/tesseract/wiki), and download 32bit or 64bit installer:

![](https://miro.medium.com/max/60/1*8DCCEQBjhSifztQA2QmvoA.png?q=20)

The installation by itself is straightforward, boils down to clicking **_Next_** a couple of times. And yeah, you also need to do a **pip installation**:

```
pip install pytesseract
```

**_Is that all?_** Well, no. You still need to tell Python where Tesseract is installed. On Linux machines, I didn’t have to do so, but it’s required on Windows. By default, it’s installed in **_Program Files_**.

If you did everything correctly, executing this cell should not yield any error:

![](https://miro.medium.com/max/60/1*FpkRjLbfJcnNH5ucdqfvew.png?q=20)

**_Is everything good?_** You may proceed.

Reading text from an image is a pretty difficult task for a computer to perform. Think about it, the computer doesn’t know what a letter is, it only works only with numbers. What happens behind the hood might seem like a black box at first, but I encourage you to investigate further if this is your area of interest.

I’m not saying that PyTesseract will work perfectly every time, but I’ve found it good enough even on some trickier images. But not straight out of the box. Some image manipulation is required to make the text stand out.

It’s a complex topic, I know. Take it one day at a time. One day it will be second nature to you.