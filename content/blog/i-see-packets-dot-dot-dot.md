+++
title = "i see packets dot dot dot"
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

Also had a handy hex lookup table (originally found at http://ascii.cl):

<TABLE ALIGN="CENTER" WIDTH="730" CELLSPACING="0" CELLPADDING="5" border=8 BORDERCOLOR=#333366 BGCOLOR="#FFFFFF">
<TR>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
0<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>
8<br>9<br>10<br>11<br>12<br>13<br>14<br>15<br><br>
</TD>
<TD align=center>
0<br>1<br>2<br>3<br>4<br>5<br>6<br>7<br>
8<br>9<br>A<br>B<br>C<br>D<br>E<br>F<br><br>
</TD>
<TD align=center>
000<br>001<br>002<br>003<br>004<br>005<br>006<br>007<br>
010<br>011<br>012<br>013<br>014<br>015<br>016<br>017<br><br>
</TD>
<TD align=center>
00000000<br>00000001<br>00000010<br>00000011<br>
00000100<br>00000101<br>00000110<br>00000111<br>
00001000<br>00001001<br>00001010<br>00001011<br>
00001100<br>00001101<br>00001110<br>00001111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
16<br>17<br>18<br>19<br>20<br>21<br>22<br>23<br>
24<br>25<br>26<br>27<br>28<br>29<br>30<br>31<br><br>
</TD>
<TD align=center>
10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>
18<br>19<br>1A<br>1B<br>1C<br>1D<br>1E<br>1F<br><br>
</TD>
<TD align=center>
020<br>021<br>022<br>023<br>024<br>025<br>026<br>027<br>
030<br>031<br>032<br>033<br>034<br>035<br>036<br>037<br><br>
</TD>
<TD align=center>
00010000<br>00010001<br>00010010<br>00010011<br>
00010100<br>00010101<br>00010110<br>00010111<br>
00011000<br>00011001<br>00011010<br>00011011<br>
00011100<br>00011101<br>00011110<br>00011111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
32<br>33<br>34<br>35<br>36<br>37<br>38<br>39<br>
40<br>41<br>42<br>43<br>44<br>45<br>46<br>47<br><br>
</TD>
<TD align=center>
20<br>21<br>22<br>23<br>24<br>25<br>26<br>27<br>
28<br>29<br>2A<br>2B<br>2C<br>2D<br>2E<br>2F<br><br>
</TD>
<TD align=center>
040<br>041<br>042<br>043<br>044<br>045<br>046<br>047<br>
050<br>051<br>052<br>053<br>054<br>055<br>056<br>057<br><br>
</TD>
<TD align=center>
00100000<br>00100001<br>00100010<br>00100011<br>
00100100<br>00100101<br>00100110<br>00100111<br>
00101000<br>00101001<br>00101010<br>00101011<br>
00101100<br>00101101<br>00101110<br>00101111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
48<br>49<br>50<br>51<br>52<br>53<br>54<br>55<br>
56<br>57<br>58<br>59<br>60<br>61<br>62<br>63<br><br>
</TD>
<TD align=center>
30<br>31<br>32<br>33<br>34<br>35<br>36<br>37<br>
38<br>39<br>3A<br>3B<br>3C<br>3D<br>3E<br>3F<br><br>
</TD>
<TD align=center>
060<br>061<br>062<br>063<br>064<br>065<br>066<br>067<br>
070<br>071<br>072<br>073<br>074<br>075<br>076<br>077<br><br>
</TD>
<TD align=center>
00110000<br>00110001<br>00110010<br>00110011<br>
00110100<br>00110101<br>00110110<br>00110111<br>
00111000<br>00111001<br>00111010<br>00111011<br>
00111100<br>00111101<br>00111110<br>00111111<br><br>
</TD>
</TR>
</TABLE>
</TD>
</TR>
<TR>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
64<br>65<br>66<br>67<br>68<br>69<br>70<br>71<br>
72<br>73<br>74<br>75<br>76<br>77<br>78<br>79<br><br>
</TD>
<TD align=center>
40<br>41<br>42<br>43<br>44<br>45<br>46<br>47<br>
48<br>49<br>4A<br>4B<br>4C<br>4D<br>4E<br>4F<br><br>
</TD>
<TD align=center>
100<br>101<br>102<br>103<br>104<br>105<br>106<br>107<br>
110<br>111<br>112<br>113<br>114<br>115<br>116<br>117<br><br>
</TD>
<TD align=center>
01000000<br>01000001<br>01000010<br>01000011<br>
01000100<br>01000101<br>01000110<br>01000111<br>
01001000<br>01001001<br>01001010<br>01001011<br>
01001100<br>01001101<br>01001110<br>01001111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
80<br>81<br>82<br>83<br>84<br>85<br>86<br>87<br>
88<br>89<br>90<br>91<br>92<br>93<br>94<br>95<br><br>
</TD>
<TD align=center>
50<br>51<br>52<br>53<br>54<br>55<br>56<br>57<br>
58<br>59<br>5A<br>5B<br>5C<br>5D<br>5E<br>5F<br><br>
</TD>
<TD align=center>
120<br>121<br>122<br>123<br>124<br>125<br>126<br>127<br>
130<br>131<br>132<br>133<br>134<br>135<br>136<br>137<br><br>
</TD>
<TD align=center>
01010000<br>01010001<br>01010010<br>01010011<br>
01010100<br>01010101<br>01010110<br>01010111<br>
01011000<br>01011001<br>01011010<br>01011011<br>
01011100<br>01011101<br>01011110<br>01011111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
96<br>97<br>98<br>99<br>100<br>101<br>102<br>103<br>
104<br>105<br>106<br>107<br>108<br>109<br>110<br>111<br><br>
</TD>
<TD align=center>
60<br>61<br>62<br>63<br>64<br>65<br>66<br>67<br>
68<br>69<br>6A<br>6B<br>6C<br>6D<br>6E<br>6F<br><br>
</TD>
<TD align=center>
140<br>141<br>142<br>143<br>144<br>145<br>146<br>147<br>
150<br>151<br>152<br>153<br>154<br>155<br>156<br>157<br><br>
</TD>
<TD align=center>
01100000<br>01100001<br>01100010<br>01100011<br>
01100100<br>01100101<br>01100110<br>01100111<br>
01101000<br>01101001<br>01101010<br>01101011<br>
01101100<br>01101101<br>01101110<br>01101111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
112<br>113<br>114<br>115<br>116<br>117<br>118<br>119<br>
120<br>121<br>122<br>123<br>124<br>125<br>126<br>127<br><br>
</TD>
<TD align=center>
70<br>71<br>72<br>73<br>74<br>75<br>76<br>77<br>
78<br>79<br>7A<br>7B<br>7C<br>7D<br>7E<br>7F<br><br>
</TD>
<TD align=center>
160<br>161<br>162<br>163<br>164<br>165<br>166<br>167<br>
170<br>171<br>172<br>173<br>174<br>175<br>176<br>177<br><br>
</TD>
<TD align=center>
01110000<br>01110001<br>01110010<br>01110011<br>
01110100<br>01110101<br>01110110<br>01110111<br>
01111000<br>01111001<br>01111010<br>01111011<br>
01111100<br>01111101<br>01111110<br>01111111<br><br>
</TD>
</TR>
</TABLE>
</TD>
</TR>
<TR>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
128<br>129<br>130<br>131<br>132<br>133<br>134<br>135<br>
136<br>137<br>138<br>139<br>140<br>141<br>142<br>143<br><br>
</TD>
<TD align=center>
80<br>81<br>82<br>83<br>84<br>85<br>86<br>87<br>
88<br>89<br>8A<br>8B<br>8C<br>8D<br>8E<br>8F<br><br>
</TD>
<TD align=center>
200<br>201<br>202<br>203<br>204<br>205<br>206<br>207<br>
210<br>211<br>212<br>213<br>214<br>215<br>216<br>217<br><br>
</TD>
<TD align=center>
10000000<br>10000001<br>10000010<br>10000011<br>
10000100<br>10000101<br>10000110<br>10000111<br>
10001000<br>10001001<br>10001010<br>10001011<br>
10001100<br>10001101<br>10001110<br>10001111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
144<br>145<br>146<br>147<br>148<br>149<br>150<br>151<br>
152<br>153<br>154<br>155<br>156<br>157<br>158<br>159<br><br>
</TD>
<TD align=center>
90<br>91<br>92<br>93<br>94<br>95<br>96<br>97<br>
98<br>99<br>9A<br>9B<br>9C<br>9D<br>9E<br>9F<br><br>
</TD>
<TD align=center>
220<br>221<br>222<br>223<br>224<br>225<br>226<br>227<br>
230<br>231<br>232<br>233<br>234<br>235<br>236<br>237<br><br>
</TD>
<TD align=center>
10010000<br>10010001<br>10010010<br>10010011<br>
10010100<br>10010101<br>10010110<br>10010111<br>
10011000<br>10011001<br>10011010<br>10011011<br>
10011100<br>10011101<br>10011110<br>10011111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
160<br>161<br>162<br>163<br>164<br>165<br>166<br>167<br>
168<br>169<br>170<br>171<br>172<br>173<br>174<br>175<br><br>
</TD>
<TD align=center>
A0<br>A1<br>A2<br>A3<br>A4<br>A5<br>A6<br>A7<br>
A8<br>A9<br>AA<br>AB<br>AC<br>AD<br>AE<br>AF<br><br>
</TD>
<TD align=center>
240<br>241<br>242<br>243<br>244<br>245<br>246<br>247<br>
250<br>251<br>252<br>253<br>254<br>255<br>256<br>257<br><br>
</TD>
<TD align=center>
10100000<br>10100001<br>10100010<br>10100011<br>
10100100<br>10100101<br>10100110<br>10100111<br>
10101000<br>10101001<br>10101010<br>10101011<br>
10101100<br>10101101<br>10101110<br>10101111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD WIDTH="180">
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
176<br>177<br>178<br>179<br>180<br>181<br>182<br>183<br>
184<br>185<br>186<br>187<br>188<br>189<br>190<br>191<br><br>
</TD>
<TD align=center>
B0<br>B1<br>B2<br>B3<br>B4<br>B5<br>B6<br>B7<br>
B8<br>B9<br>BA<br>BB<br>BC<br>BD<br>BE<br>BF<br><br>
</TD>
<TD align=center>
260<br>261<br>262<br>263<br>264<br>265<br>266<br>267<br>
270<br>271<br>272<br>273<br>274<br>275<br>276<br>277<br><br>
</TD>
<TD align=center>
10110000<br>10110001<br>10110010<br>10110011<br>
10110100<br>10110101<br>10110110<br>10110111<br>
10111000<br>10111001<br>10111010<br>10111011<br>
10111100<br>10111101<br>10111110<br>10111111<br><br>
</TD>
</TR>
</TABLE>
</TD>
</TR>
<TR>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
192<br>193<br>194<br>195<br>196<br>197<br>198<br>199<br>
200<br>201<br>202<br>203<br>204<br>205<br>206<br>207<br><br>
</TD>
<TD align=center>
C0<br>C1<br>C2<br>C3<br>C4<br>C5<br>C6<br>C7<br>
C8<br>C9<br>CA<br>CB<br>CC<br>CD<br>CE<br>CF<br><br>
</TD>
<TD align=center>
300<br>301<br>302<br>303<br>304<br>305<br>306<br>307<br>
310<br>311<br>312<br>313<br>314<br>315<br>316<br>317<br><br>
</TD>
<TD align=center>
11000000<br>11000001<br>11000010<br>11000011<br>
11000100<br>11000101<br>11000110<br>11000111<br>
11001000<br>11001001<br>11001010<br>11001011<br>
11001100<br>11001101<br>11001110<br>11001111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
208<br>209<br>210<br>211<br>212<br>213<br>214<br>215<br>
216<br>217<br>218<br>219<br>220<br>221<br>222<br>223<br><br>
</TD>
<TD align=center>
D0<br>D1<br>D2<br>D3<br>D4<br>D5<br>D6<br>D7<br>
D8<br>D9<br>DA<br>DB<br>DC<br>DD<br>DE<br>DF<br><br>
</TD>
<TD align=center>
320<br>321<br>322<br>323<br>324<br>325<br>326<br>327<br>
330<br>331<br>332<br>333<br>334<br>335<br>336<br>337<br><br>
</TD>
<TD align=center>
11010000<br>11010001<br>11010010<br>11010011<br>
11010100<br>11010101<br>11010110<br>11010111<br>
11011000<br>11011001<br>11011010<br>11011011<br>
11011100<br>11011101<br>11011110<br>11011111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
224<br>225<br>226<br>227<br>228<br>229<br>230<br>231<br>
232<br>233<br>234<br>235<br>236<br>237<br>238<br>239<br><br>
</TD>
<TD align=center>
E0<br>E1<br>E2<br>E3<br>E4<br>E5<br>E6<br>E7<br>
E8<br>E9<br>EA<br>EB<br>EC<br>ED<br>EE<br>EF<br><br>
</TD>
<TD align=center>
340<br>341<br>342<br>343<br>344<br>345<br>346<br>347<br>
350<br>351<br>352<br>353<br>354<br>355<br>356<br>357<br><br>
</TD>
<TD align=center>
11100000<br>11100001<br>11100010<br>11100011<br>
11100100<br>11100101<br>11100110<br>11100111<br>
11101000<br>11101001<br>11101010<br>11101011<br>
11101100<br>11101101<br>11101110<br>11101111<br><br>
</TD>
</TR>
</TABLE>
</TD>
<TD>
<TABLE ALIGN="CENTER" CELLSPACING="0" CELLPADDING="0" border=0>
<TR>
<TD WIDTH="40" align=center><br><b>Dec</b></TD>
<TD WIDTH="40" align=center><br><b>Hex</b></TD>
<TD WIDTH="40" align=center><br><b>Oct</b></TD>
<TD WIDTH="60" align=center><br><b>Bin</b></TD>
</TR>
<TR>
<TD colspan=4><HR></TD>
</TR>
<TR>
<TD align=center>
240<br>241<br>242<br>243<br>244<br>245<br>246<br>247<br>
248<br>249<br>250<br>251<br>252<br>253<br>254<br>255<br><br>
</TD>
<TD align=center>
F0<br>F1<br>F2<br>F3<br>F4<br>F5<br>F6<br>F7<br>
F8<br>F9<br>FA<br>FB<br>FC<br>FD<br>FE<br>FF<br><br>
</TD>
<TD align=center>
360<br>361<br>362<br>363<br>364<br>365<br>366<br>367<br>
370<br>371<br>372<br>373<br>374<br>375<br>376<br>377<br><br>
</TD>
<TD align=center>
11110000<br>11110001<br>11110010<br>11110011<br>
11110100<br>11110101<br>11110110<br>11110111<br>
11111000<br>11111001<br>11111010<br>11111011<br>
11111100<br>11111101<br>11111110<br>11111111<br><br>
</TD>
</TR>
</TABLE>
</TD>
</TR>
</TABLE>

