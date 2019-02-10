---
layout: blog
title: Building an image gallery for Hugo
date: '2019-02-09T21:40:11-05:00'
cover: /images/hugo.png
categories:
  - hugo shortcodes
---
# A quick image gallery for Hugo

<!--more-->
Here's a quick image gallery that can read a folder and generate a quick gallery.

It will try to load all JPG files in the provided path, and assumes that every image has a thumbnail file that has the `-thumb.EXT` extension. Ex: for a file 'blah.jpg', it would assume there would also be a 'blah-thumb.jpg'.

Drop this into `layouts/shortcodes/foldergallery.html`:

{{< gist 38aa7ce30608da6c5182527105e79b0e >}}

Once that's done add this javascript to your installation somewhere: 

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/fancybox/3.4.0/jquery.fancybox.min.js"></script>
```

The shortcode assumes that galleries are stored in folders under the /static path.

Here's an example of how to use it:

```md
{{< foldergallery src="images/galleries/2015/" >}}
```

# Generating thumbnails

Here's how I do it in Linux (might also work on Mac with homebrew). First, install imagemagick. I store my galleries under `static/images/galleries/<GALLERYNAME>`. To create thumbnails for all my galleries:

```bash
for i in `find static/images/galleries -type f ! -name "*-thumb.jpg" -name "*.jpg"`; do echo $i; if [ -f ${i%.*}-thumb.jpg ]; then continue; fi; convert $i -thumbnail 100x100 ${i%.*}-thumb.jpg; done
```

Repeat this for each file format you want thumbnails for (this will only convert jpegs).
