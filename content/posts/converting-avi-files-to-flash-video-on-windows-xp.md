---
title: "converting avi files to flash video on windows xp"
date: "2011-01-06"
slug: "2011/01/06/converting-avi-files-to-flash-video-on-windows-xp"
---

Step 1. Install ffmpeg.

I believe I downloaded it from [here](http://ffmpeg.arrozcru.org/autobuilds/)

There was issue with the installation. After installing it (in c:\windows I believe), I noticed that none of the presets worked. This is natively a linux app, and after a bit of poking around I saw that it was looking for the presets in /usr/local/share/ffmpeg.

To fix this, I created that folder in the root of the disk I was using for the transcoding (X in my case), and put the preset files in there. After that, it worked like a charm.

Step 2. The handy dandy ffmpeg command with some batch wizardry:

```bash
for /f %G IN ('dir /b raw\*.AVI') do ffmpeg -i raw\%G -b 2000k -s 640x480 %~nG.flv
```

Enjoy.


Update

The windows Imagemagick packages has a ffmpeg binary bundled in. As a bonus, it appears to work better in my limited testing. Download it [here](http://www.imagemagick.org/download/binaries/ImageMagick-6.6.9-5-Q16-windows-dll.exe)

Update 2

To rotate 90 degrees.
```bash
for /f %G IN ('dir /b raw\*.AVI') do ffmpeg -i raw\%G -b 2000k -s 640x480 -vf "transpose=1" %~nG.flv
```
