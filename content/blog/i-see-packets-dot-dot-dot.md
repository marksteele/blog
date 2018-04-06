+++
title = "i see packets..."
date = "2013-10-23"
slug = "2013/10/23/i-see-packets-dot-dot-dot"
Categories = []
+++

While studying for the GCIA certification, I put together the following reference to be able to eyeball packets and see at a glance what's inside a hex packet dump.
<!--more-->
```
IP HEADER DISSECTION
====================
                                          3 bits from high order nibble
          IP-header-len                    for flags
               |        Total-length         | rest is frag offset
               |            |                |  | 
        ver    |    ToS     |       ID       |  |    TTL  PROTO checksum    src ip
0x0000: [4]   [5]   [00]  [0046]   [0000]  [[40]00]  [40] [11]   [a314]   [c0a8 0b01]
                                               |
                                               |
                  -----------------------------+
                  |
             [0]    [1]   [0] [0 0000 0000 0000] --> frag offset
             r/evil df     mf

          dst ip
             |
0x0010: [c0a8 0b02]  <options or payload>


IP header length: mutliplied by 4. Value of 5 means no options. Max length is 60 bytes.

-----------------------------------------------------------------------------------------


UDP packet dissection (no IP options)
=====================================

0x0000: [4]   [5]   [00]  [0046]   [0000]  [[4]000]  [40] [11]   [a314]   [c0a8 0b01]
 
               start of udp header 
                     |
          dst ip     | srcport  dstport  length   chekcsum   payload
0x0010: [c0a8 0b02] << [0035]   [ce9e]   [0032]   [15b9]    <2a9c 8180
0x0020: .... payload ...


UDP DNS packet dissection


0x0000: [4]   [5]   [00]  [0046]   [0000]  [[4]000]  [40] [11]   [a314]   [c0a8 0b01]
                                  
                                                 start of DNS       
                                                       |        
                                                       |   ID
0x0010: [c0a8 0b02] [0035]   [ce9e]   [0032]   [15b9] << [2a9c] [81] [80]
                                                                  |    |
                                 ---------------------------------+    |
                                 |                                     |
                          [1] [0000] [0] [0] [1]                [1]  [000] [0000]
                          QR  opcode AA  TC   RD                 RA    Z    RCODE

        qdcount ancount     nscount     arcount    
0x0020: [0001]  [0001]      [0000]      [0000]  
<snip>

The rest of the sections (question section, answer section, authority section, additional info) 
are variable length based on the first 4 counters.

In this case, we're seeing a one answer. Variable length fields are formatted as 
1 byte length -> first part (eg: www)	length 2 -> second part (eg:google) -> 
length N part N (eg:com) followed by a null (00)


TCP packet dissection (no ip options)
======================================
 
0x0000: [4][5] [00] [0045] [3621] [[4]000] [40] [06] [d2c4] [c0a8 582e] 
                 Start TCP header
                     |  
                     |  sport dport  seq num     ack num
0x0010: [c0a8 584e] << [9792] [1a0b] [9230 e09d] [23eb 4ae7]

     header len
         |      flags win size  checksum           options
         |       |       |        |                  |
0x0020: [80]    [18]  [029a]    [4d20]    [0101 080a 0097 2480 003d
0x0030: 2eef] <<PAYLOAD>>


Options: NOPNOP timestamp+10 bytes len, value

```
