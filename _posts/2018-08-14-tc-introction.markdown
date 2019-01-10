---
layout: post
title:  "Linux Traffic Control - Introduction"
date:   2018-08-15 00:51:40 -0700
categories: networking
---



# Linux Traffic Control

接收包從輸入接口（Input Interface）進來後，經過流量限制（Ingress Policing）丟棄不符合規定的數據包，由輸入多路分配器（Input De-Multiplexing）進行判斷選擇：如果接收包的目的是本主機，那麼將該包送給上層處理；否則需要進行轉發，將接收包交到轉發塊（Forwarding Block）處理。轉發塊同時也接收本主機上層（TCP、UDP等）產生的包。轉發塊通過查看路由表，決定所處理包的下一跳。然後，對包進行排列以便將它們傳送到輸出接口（Output Interface）。一般我們只能限制網卡發送的數據包，不能限制網卡接收的數據包，所以我們可以通過改變發送次序來控制傳輸速率。**Linux 流量控制主要是在輸出接口(egress)排列(qdisc)時進行處理和實現的**

![Netfilter-packet-flow](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)

![ingress_Netfilter-packet-flow.png](/images/tc/ingress_Netfilter-packet-flow.png)![egress_Netfilter-packet-flow.png](/images/tc/egress_Netfilter-packet-flow.png)

一般說到qdisc都是指egress qdisc。每塊網卡實際上還可以添加一個ingress qdisc，不過它有諸多的限制

- ingress qdisc不能包含子類，而只能作過濾
- ingress qdisc只能用於簡單的整形

如果相對ingress方向作流量控制的話，可以藉助ifb（ [Intermediate Functional Block](https://wiki.linuxfoundation.org/networking/ifb)）內核模塊。因為流入網絡接口的流量是無法直接控制的，那麼就需要把流入的包導入（通過tc action）到一個中間的隊列，該隊列在ifb 設備上，然後讓這些包重走tc 層，最後流入的包再重新入棧，流出的包重新出棧



## Components of Linux Traffic Control 

[Traffic Control HOWTO]: http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html	"Linux TC components"

| Traditional element                                          | Linux component                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [shaping](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-shaping) | The [`class`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-class) offers shaping capabilities. |
| [scheduling](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-scheduling) | A [`qdisc`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-qdisc) is a scheduler. Schedulers can be simple such as the FIFO or complex, containing classes and other qdiscs, such as HTB. |
| [classifying](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-classifying) | The [`filter`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-filter) object performs the classification through the agency of a [`classifier`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-classifier) object. Strictly speaking, Linux classifiers cannot exist outside of a filter. |
| [policing](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-policing) | A [`policer`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-police) exists in the Linux traffic control implementation only as part of a [`filter`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-filter). |
| [dropping](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-dropping) | To [`drop`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-drop) traffic requires a [`filter`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-filter) with a [`policer`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-police) which uses "drop" as an action. |
| [marking](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-marking) | The `dsmark` [`qdisc`](http://www.tldp.org/HOWTO/Traffic-Control-HOWTO/components.html#c-qdisc) is used for marking.<br /> install a DSCP on the packet itself (for DiffServ) |

查詢[linux tc command](https://linux.die.net/man/8/tc)，可以看到以下控制方法:

**SHAPING**(限制/整形)

When traffic is shaped, its rate of transmission is under control. Shaping may be more than lowering the available bandwidth - it is also used to smooth out bursts in traffic for better network behaviour. Shaping occurs on egress.

當流量被限制，它的傳輸速率就被控制在某個值以下。限制值可以大大小於有效帶寬，這樣可以平滑突發數據流量，使網絡更爲穩定。shaping（限制）只適用於向外的流量。 

**SCHEDULING**(調度)

By scheduling the transmission of packets it is possible to improve interactivity for traffic that needs it while still guaranteeing bandwidth to bulk transfers. Reordering is also called prioritizing, and happens only on egress.

通過調度數據包的傳輸，可以在帶寬範圍內，按照優先級分配帶寬。SCHEDULING (調度) 也只適於向外的流量。 

**POLICING**(策略)

Where shaping deals with transmission of traffic, policing pertains to traffic arriving. Policing thus occurs on ingress.

SHAPING用於處理向外的流量，而POLICIING(策略)用於處理接收到的數據。 

**DROPPING**(丟棄)

Traffic exceeding a set bandwidth may also be dropped forthwith, both on ingress and on egress.

Processing of traffic is controlled by three kinds of objects: qdiscs, classes and filters.

如果流量超過某個設定的帶寬，就丟棄數據包，不管是向內還是向外。 



## 流量控制處理對象(objects)： qdisc (排隊規則)、class (類別) 和 filter (過濾器) 

### 簡要描述

- Qdisc: how to queue the packets

- Class: tied with qdiscs to form a hierarchy

- Filter: how to classify or filter the packets

- Action: how to deal with the matched packets

**處理流程**

```
for_each_packet(pkt, Qdisc):
  for_each_filter(filter, Qdisc):
    if filter(pkt):
      classify(pkt)
      for_each_action(act, filter):
        act(pkt)
```

**代碼源**

- Kernel source code

  net/sched/sch\_\*.c

  net/sched/cls\_\*.c

  net/sched/act_*.c

- iproute2 source code

  tc/q\_\*.c

  tc/f\_\*.c

  tc/m\_\*.c


### QDISCS (how to queue the packets)

- Attached to a network interface
- Can be organized hierarchically with classes
- Has a unique handle on each interface
- Almost all qdiscs are for egress
- Ingress is a special case

qdisc is short for 'queueing discipline' and it is elementary to understanding traffic control. Whenever the kernel needs to send a packet to an interface, it is enqueued to the qdisc configured for that interface. Immediately afterwards, the kernel tries to get as many packets as possible from the qdisc, for giving them to the network adaptor driver.
A simple QDISC is the 'pfifo' one, which does no processing at all and is a pure First In, First Out queue. It does however store traffic when the network interface can't handle it momentarily.

QDisc (排隊規則) 是queueing discipline 的簡寫，它是理解流量控制(traffic control)的基礎。無論何時，內核如果需要通過某個網絡接口發送數據包，它都需要按照爲這個接口配置的qdisc(排隊規則)把數據包加入隊列。然後，內核會儘可能多地從qdisc裏面取出數據包，把它們交給網絡適配器驅動模塊。
最簡單的QDisc是pfifo它不對進入的數據包做任何的處理，數據包採用先入先出的方式通過隊列。不過，它會保存網絡接口一時無法處理的數據包。 

**Classless Qdiscs (無階)**

- FIFO (First In, First Out) [bfifo, pfifo, pfifo_head_drop]
  - 使用最簡單的qdisc，純粹的先進先出。只有一個參數：limit，用來設置隊列的長度,pfifo是以數據包的個數為單位；bfifo是以字節數為單位
- Priority queueing [pfifo_fast, prio]
  - 在編譯內核時，如果打開了高級路由器(Advanced Router)編譯選項，pfifo_fast就是系統的標準QDISC。它的隊列包括三個波段(band)。在每個波段裡面，使用先進先出規則。而三個波段(band)的優先級也不相同，band 0的優先級最高，band 2的最低。如果band裡面有數據包，系統就不會處理band 1裡面的數據包，band 1和band 2之間也是一樣。數據包是按照服務類型(Type of Service,TOS)被分配多三個波段(band)裡面的
- Multiqueue [mq, multiq]
  - For multiple hardware TX queues
  - Queue mapping with hash, priority or by classifier
  - Combine with priority: mq_prio
- red (Random Early Detection)
  - 當帶寬的佔用接近於規定的帶寬時，系統會隨機地丟棄一些數據包。適合高帶寬應用
- sfq (Stochastic Fairness Queueing)
  - 它按照會話(session–對應於每個TCP連接或者UDP流)為流量進行排序，然後循環發送每個會話的數據包
- tbf (Token Bucket Filter)
  - 適合於把流速降低到某個值

**Classful Qdiscs (階級)**

- CBQ (Class Based Queueing)
  - 實現了一個豐富的連接共享類別結構，既有限制(shaping)帶寬的能力，也具有帶寬優先級管理的能力
  - 帶寬限制是通過計算連接的空閒時間完成的
  - 空閒時間的計算標準是數據包離隊事件的頻率和下層連接(數據鏈路層)的帶寬
- HTB (Hierarchy Token Bucket)
  - 通過在實踐基礎上的改進，它實現了一個豐富的連接共享類別體系。使用HTB可以很容易地保證每個類別的帶寬，雖然它也允許特定的類可以突破帶寬上限，佔用別的類的帶寬
  - 實現帶寬限制，也能夠劃分類別的優先級
- PRIO
  - 不能限制帶寬，因為屬於不同類別的數據包是順序離隊的
  - 可以很容易對流量進行優先級管理，只有屬於高優先級類別的數據包全部發送完畢，才會發送屬於低優先級類別的數據包
  - 為了方便管理，需要使用iptables或者ipchains處理數據包的服務類型(Type Of Service,ToS)

### CLASSES  (tied with qdiscs to form a hierarchy)

Some qdiscs can contain classes, which contain further qdiscs - traffic may then be enqueued in any of the inner qdiscs, which are within the classes.  When the kernel tries to dequeue a packet from such a classful qdisc it can  come  from  any  of  the  classes. A qdisc may for example prioritize certain kinds of traffic by trying to dequeue from certain classes before others.

某些QDisc(排隊規則)可以包含一些類別，不同的類別中可以包含更深入的QDisc(排隊規則)，通過這些細分的QDisc還可以爲進入的隊列的數據包排隊。通過設置各種類別數據包的離隊次序，QDisc可以爲設置網絡數據流量的優先級。

### FILTERS (how to classify or filter the packets)

- As known as classifier
- Attached to a Qdisc
- The rule to match a packet
- Need qdisc support
- Protocol, priority, handle

A filter is used by a classful qdisc to determine in which class a packet will be enqueued. Whenever traffic arrives at a class with subclasses, it needs to be classified. Various methods may be employed to do so, one of these are the filters. All filters attached to the class are called, until one of them returns with a verdict. If no verdict was made, other criteria may be available. This differs per qdisc.

filter(過濾器)用於爲數據包分類，決定它們按照何種QDisc進入隊列。無論何時數據包進入一個劃分子類的類別中，都需要進行分類。分類的方法可以有多種，使用fileter(過濾器)就是其中之一。使用filter(過濾器)分類時，內核會調用附屬於這個類(class)的所有過濾器，直到返回一個判決。如果沒有判決返回，就作進一步的處理，而處理方式和QDISC有關。需要注意的是，filter(過濾器)是在QDisc內部，它們不能作爲主體。 

