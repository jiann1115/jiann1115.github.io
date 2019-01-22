---
11layout: post
title:  "Linux Traffic Control - Summarize"
date:   2019-01-10 00:51:40 -0700
categories: networking
---



# Linux Traffic Control - Summarize

[Linux Traffic Control - Introduction](https://jiann1115.github.io/networking/2018/08/15/tc-introction.html)

[Linux Traffic Control - qdisc](https://jiann1115.github.io/networking/2018/08/16/tc-qdisc.html)

[Linux Traffic Control - htb](https://jiann1115.github.io/networking/2018/08/20/tc-htb.html)

[Linux Traffic Control - cmd](https://jiann1115.github.io/networking/2018/12/13/tc-cmd.html)

閱讀上面四篇文章，對tc有一些認識，接下來簡單整理一下，再次使用之前的範例(http 80 port的流量限制)，來觀察dev tap823dcd7d-bc上會發生甚麼事情。

再下達命令前，dev tap823dcd7d-bc是採用pfifo

```
#tc qdisc show dev tap823dcd7d-bc
    qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```


 我們即將下達的命令是會改變egress的qdisc，會再netfilter下面的位置

![egress_Netfilter-packet-flow](/images/tc/egress_Netfilter-packet-flow.png)

Kernel的流程如下圖，Data Link layer 會做packet scheduling，dev_queue_xmit()會對dev的qdisc做 enqueue()，pfifo的enqueue()指向pfifo_fast_enqueue()，然後dequeue()依照pfifo的policy進行

![pfifo](/images/tc/pfifo.png)

現在，我們下達TC命令:

針對DEV tap823dcd7d-bc後的vm做http 80 port的流量限制為1Mbit，其餘留量可保持在4Mbit :

```
tc qdisc add dev tap823dcd7d-bc root handle 1: htb default 10
# 設定1:10流量4Mbit, 以下再分1:100, 1:200 
tc class add dev tap823dcd7d-bc parent 1: classid 1:10 htb rate 4Mbit ceil 4Mbit

# 讓sport 80導入到1:100, 流量1Mbit, 
tc class add dev tap823dcd7d-bc parent 1:10 classid 1:100 htb rate 1Mbit ceil 1Mbit prio 1
tc qdisc add dev tap823dcd7d-bc parent 1:100 handle 101: pfifo
tc filter add dev tap823dcd7d-bc parent 1: protocol ip prio 1 u32 match ip sport 80 0xffff flowid 1:100

# 其他導入到1:200, 流量3Mbit, 
tc class add dev tap823dcd7d-bc parent 1:10 classid 1:200 htb rate 3Mbit ceil 4Mbit prio 1
tc qdisc add dev tap823dcd7d-bc parent 1:200 handle 102: pfifo
tc filter add dev tap823dcd7d-bc parent 1: protocol ip prio 1 u32 match u8 0 0 flowid 1:200
```

**TC 命令到Kernel，會去修改qdisc為HTB並按照命令設定其policy**

![tc_qdisc_cmd](/images/tc/tc_qdisc_cmd.png)

我們從命令查詢，目前dev tap823dcd7d-bc的狀態如下

```
#tc -d qdisc show dev tap823dcd7d-bc
qdisc htb 1: root refcnt 2 r2q 10 default 10 direct_packets_stat 33 ver 3.17 direct_qlen 1000
qdisc pfifo 101: parent 1:100 limit 1000p
qdisc pfifo 102: parent 1:200 limit 1000p
```

```
#tc -s class show dev tap823dcd7d-bc
class htb 1:100 parent 1:10 leaf 101: prio 1 rate 1Mbit ceil 1Mbit burst 1600b cburst 1600b
    ...
class htb 1:10 root rate 4Mbit ceil 4Mbit burst 1600b cburst 1600b
    ...
class htb 1:200 parent 1:10 leaf 102: prio 1 rate 3Mbit ceil 4Mbit burst 1599b cburst 1600b
```

```
#tc filter show dev tap823dcd7d-bc
filter parent 1: protocol ip pref 1 u32 
filter parent 1: protocol ip pref 1 u32 fh 800: ht divisor 1 
filter parent 1: protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:100 
  match 00500000/ffff0000 at 20
filter parent 1: protocol ip pref 1 u32 fh 800::801 order 2049 key ht 800 bkt 0 flowid 1:200 
  match 00000000/00000000 at 0
```


圖形化來看如下:

![htb](/images/tc/htb.png)





我們再加上prio的分類，然後把VoIP做QoS，qdisc prio特性很簡單，會讓priority高的qdisc先送出，確定queue內沒有之後才會送其他次高priority 的qdisc。利用這個特性，讓VoIP的封包能優先被送出，我們可以把設計改成下面方式，把DSCP中VoIP的traffic設定為高優先級

> SIP Traffic
>
> SIP (signalling) – Class 3, AF31, DSCP: 0x1A, TOS: 0x68
>
> RTP (media) – Expedited Forwarding, EF, DSCP: 0x2E, TOS: 0xB8



```
tc qdisc add dev tap823dcd7d-bc root handle 1: prio bands 2 priomap 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1

tc qdisc add dev tap823dcd7d-bc parent 1:1 handle 11: pfifo
tc qdisc add dev tap823dcd7d-bc parent 1:2 handle 12: htb default 10

# SIP TOS: 0x68 導入到1:1
tc filter add dev tap823dcd7d-bc parent 1: prio 1 protocol ip u32 \
    match ip tos 0x68 0xff \
    match ip protocol 0x11 0xff \
    flowid 1:1

# RTP TOS: 0xB8 導入到1:1
tc filter add dev tap823dcd7d-bc parent 1: prio 1 protocol ip u32 \
    match ip tos 0xb8 0xff \
    match ip protocol 0x11 0xff \
    flowid 1:1

# 其他導入到1:2
tc filter add dev tap823dcd7d-bc parent 1: protocol ip prio 2 u32 \
    match u8 0 0 \
    flowid 1:2

## 設定12:10流量4Mbit, 以下再分12:100, 12:200
tc class add dev tap823dcd7d-bc parent 12: classid 12:10 htb rate 4Mbit ceil 4Mbit

## 讓sport 80導入到1:100, 流量1Mbit
tc class add dev tap823dcd7d-bc parent 12:10 classid 12:100 htb rate 1Mbit ceil 1Mbit prio 2
tc qdisc add dev tap823dcd7d-bc parent 12:100 handle 101: pfifo
tc filter add dev tap823dcd7d-bc parent 12: protocol ip prio 2 u32 match ip sport 80 0xffff flowid 12:100

## 其他導入到1:200, 流量3Mbit
tc class add dev tap823dcd7d-bc parent 12:10 classid 12:200 htb rate 3Mbit ceil 4Mbit prio 2
tc qdisc add dev tap823dcd7d-bc parent 12:200 handle 102: pfifo
tc filter add dev tap823dcd7d-bc parent 12: protocol ip prio 2 u32 match u8 0 0 flowid 12:200
```

命令查詢，更新dev tap823dcd7d-bc的狀態如下

```
#tc -d qdisc show dev tap823dcd7d-bc
qdisc prio 1: root refcnt 2 bands 2 priomap  1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
qdisc pfifo 11: parent 1:1 limit 1000p
qdisc htb 12: parent 1:2 r2q 10 default 10 direct_packets_stat 21736 ver 3.17 direct_qlen 1000
qdisc pfifo 101: parent 12:100 limit 1000p
qdisc pfifo 102: parent 12:200 limit 1000p
```

```
#tc class show dev tap823dcd7d-bc
class prio 1:1 parent 1: leaf 11: 
class prio 1:2 parent 1: leaf 12: 
class htb 12:100 parent 12:10 leaf 101: prio 2 rate 1Mbit ceil 1Mbit burst 1600b cburst 1600b 
class htb 12:10 root rate 4Mbit ceil 4Mbit burst 1600b cburst 1600b 
class htb 12:200 parent 12:10 leaf 102: prio 2 rate 3Mbit ceil 4Mbit burst 1599b cburst 1600b
```

```
#tc filter show dev tap823dcd7d-bc
filter parent 1: protocol ip pref 1 u32 
filter parent 1: protocol ip pref 1 u32 fh 800: ht divisor 1 
filter parent 1: protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:1 
  match 00680000/00ff0000 at 0
  match 00110000/00ff0000 at 8
filter parent 1: protocol ip pref 1 u32 fh 800::801 order 2049 key ht 800 bkt 0 flowid 1:1 
  match 00b80000/00ff0000 at 0
  match 00110000/00ff0000 at 8
filter parent 1: protocol ip pref 3 u32 
filter parent 1: protocol ip pref 3 u32 fh 801: ht divisor 1 
filter parent 1: protocol ip pref 3 u32 fh 801::800 order 2048 key ht 801 bkt 0 flowid 1:2 
  match 00000000/00000000 at 0

```

圖形化來看，就會變成這樣子

![htb](/images/tc/prio_voip.png)

