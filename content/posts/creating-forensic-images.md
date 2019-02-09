---
title: "creating forensic images"
date: "2013-04-04"
slug: "2013/04/04/creating-forensic-images"
categories:
  - security
  - linux
  - devops
cover: "/images/linux-security.jpg"
---

Often reading big disks is a time consuming endeavor. To minimize the number of times you need to read the data, here's a tip for reading the image using dd, compressing it, and checksumming it.

```bash
dd if=/dev/sda | pv | tee >( md5sum > box.dd.md5 ) | \
tee >( sha1 > box.dd.sha1  ) | tee box.dd | gzip | \
tee box.dd.gz | tee >( md5sum >box.dd.gz.md5 ) | \
sha1 >box.dd.gz.sha1
```

This is going to be a pretty CPU hungry process. If you.ve got lots of cores, you can further speed things up by using 'pigz' (parallel gzip) instead of gzip.

As a side note, this is a generic approach when you need to pipe the output of one program to many others simultaneously.
