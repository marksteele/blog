---
layout: blog
title: 'Pro-tip: grep through terminal history'
date: '2019-10-23T09:39:24-04:00'
cover: /images/bash.png
categories:
  - pro-tips
---
Might seem like a stupid little trick, but...

On a mac, if you want to `grep` your terminal scrollback (as opposed to using search).

command-a (select all), command-c (copy), then:

```bash
pbpaste | grep <WHAT YOU ARE LOOKING FOR>
```

