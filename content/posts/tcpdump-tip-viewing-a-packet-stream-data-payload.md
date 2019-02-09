---
title: "tcpdump tip viewing a packet stream data payload"
date: "2013-10-24"
slug: "2013/10/24/tcpdump-tip-viewing-a-packet-stream-data-payload"
categories:
  - networking
cover: "/images/tcpip.png"
---

Here is an alias that I've used often to view packet payloads using tcpdump which filters out all the overhead packets (just contains payloads).

I usually stick the following lines into my .bashrc on all the servers I install.

```bash
alias tcpdump_http="tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' -A -s0" 
alias tcpdump_http_inbound="tcpdump 'tcp dst port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' -A -s0" 
alias tcpdump_http_outbound="tcpdump 'tcp src port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' -A -s0" 
```

You can pass as argument the interface you want to listen on (defaults to eth0) via a '-i eth0:1' for example. It snarfs in the payload, so it's easy to follow what's going on.

An equally viable alternative is to install tcpflow.
