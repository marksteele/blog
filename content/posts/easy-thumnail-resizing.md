---
title: "easy thumnail resizing"
date: "2011-05-29"
slug: "2011/05/29/easy-thumnail-resizing"
---

Download and install [ImageMagick](http://www.imagemagick.org/)

```bash
cd path\to\images
mkdir thumbs
mkdir resized
mogrify -path thumbs -thumbnail 100x100 *.JPG 
mogrify -path resized -resize 1024x768 *.JPG
```


Done!
