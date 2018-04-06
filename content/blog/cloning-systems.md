+++
title = "cloning systems"
date = "2011-04-17"
slug = "2011/04/17/cloning-systems"
Categories = []
+++

Grab [partclone](http://partclone.org/) 

On both the server you want to image, and the server you want to restore to, boot a live usb that has netcat and partclone. I create 
my own using the gentoo LiveUSB install http://www.gentoo.org/doc/en/liveusb.xml, and copy the tools onto the usb stick.

On target node:

```bash
nc -l -p 999 | partclone.ext4 -r -o /dev/sda2
```

On source node:

```bash
partclone.ext4 -c -s /dev/sda2 | nc 1.2.3.4 999.http://partclone.nchc.org.tw/trac/wiki/Download
```

It will send the image over the network, and will do so efficiently without wasting bandwidth on sending empty disk sectors, as it's a logical partition dump.
