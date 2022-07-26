# TCP 三次握手分析
---
todo:
* TCP Socket
* 三次握手阶段涉及到的内核参数介绍
---

```sequence
title: TCP 标准三次握手
participant client
participant server
Note left of client: CLOSE -> SYN-SENT
client->server: 1. SYN (seq = x, ack = 0)
Note right of server: LISTEN -> SYN-RCVD
server-->client: 2. SYN + ACK (seq = y, ack = x+1)
Note left of client: SYN-SENT -> ESTABLISHED
client->server: 3. ACK (seq = x+1, ack = y+1)
Note right of server: SYN-RCVD -> ESTABLISHED
```
* TCP的option一般是在握手阶段协商的，如mss等。
* 通常情况下，握手阶段是不传输数据的，除非两端开启了TFO（一个选项），TFO是什么后面会分析。
---

```sequence
title: 三次握手有丢包
participant client
participant midbox
participant server

client-> midbox: 1. SYN (seq = x, ack = 0)
midbox->midbox: 对SYN包进行处理:\n 1. 路由器检测路由；\n 2. 防护墙创建连接表，做NAT等
Note right of midbox: SYN包被midbox丢弃: \n 1. 找不到下一跳路由；\n 2. 网络拥塞;\n 3. 防火墙需要检验等 
client->client: 1. 等待对端响应；\n 2. 开始定时器进行超时检测
Note right of client: 1. 在三次握手阶段，定时器的超时间隔在RFC中指定了: \n 3s(RFC 1122)\n 1s(RFC 6298 2.1)\n linux kernel: tcp.h - TCP_TIMEOUT_INIT \n 2. 超时时间内没有收到SYN+ACK，由定时器重传SYN包；   
client-> midbox: 1'. SYN (seq = x, ack = 0)
midbox->midbox: 对SYN包进行处理 
midbox->server: 1'. SYN (seq = x, ack = 0)
server-->client: 2. SYN + ACK (seq = y, ack = x+1)
client->server: 3. ACK (seq = x+1, ack = y+1)
```
* 在三次握手阶段，SYN包被丢弃是很正常的现象；
* RFC规定了定时器的间隔（最早的时候是3s，在2009年左右的时候改成了1s，具体参考RFC。有兴趣的可以在不同版本的操作系统上抓包看一下）
* 客户端重传间隔遵循指数退让的原则：每次间隔较上一次翻倍，1s, 2s, 4s, 8s, 16s...
* 正常的客户端（一般是在操作系统中实现的，也有用户态协议栈，如lwip等）的实现都有重传次数限制，如在linux操作系统上：/proc/sys/net/ipv4/tcp_syn_retries (在内核中默认值是定义在tcp.h中的：TCP_SYN_RETRIES)
* **以上描述的是RFC标准规范，但具体实现还要依赖操作系统，有的操作系统出于某些目的（提高连接建立成功率等）会在RFC的基础上进行调整，比方说水果（前面几次重传间隔都是1s，没有翻倍）**
---




```sequence
title: SYN-Flood 1.0
participant attacker
participant server

attacker->server: 1. SYN(seq = x, ack = 0)
attacker->server: ...(大量的SYN请求)
attacker->server: N. SYN(seq = x', ack = 0)

server-->attacker: 2. SYN + ACK (seq = y, ack = x+1)
server-->attacker: ...(大量的SYN+ACK响应)
Note over server: 1. 系统上有大量SYN_RECV状态的连接\n 2. 占用大量的资源（内存、带宽等）
server-->attacker: N. SYN + ACK (seq = y', ack = x'+1)
```
* 攻击者不断的向服务器发送握手请求，但不响应服务器的SYN+ACK，耗费服务器的资源：
1. 内存：记录连接；
2. CPU：维持连接的定时器；
3. 带宽：定时器超时之后会重传SYN+ACK；
* 不足：通过单一的源IP发送攻击，很容易被识别出来，通过限速和源IP过滤可以轻松处理掉攻击。
---

```sequence
title: SYN-Flood 2.0
participant attacker
participant server
participant cloud hosts

attacker->server: 1. random(IP), SYN(seq = x, ack = 0)
attacker->server: ...(大量的SYN请求)
attacker->server: N. random(IP), SYN(seq = x', ack = 0)

server-->cloud hosts: 2. SYN + ACK (seq = y, ack = x+1)
server-->cloud hosts: ...(大量的SYN+ACK响应)
Note over server: 1. 系统上有大量SYN_RECV状态的连接\n 2. 占用大量的资源（内存、带宽等）
server-->cloud hosts: N. SYN + ACK (seq = y', ack = x'+1)
```
* 通过伪造源IP的方式，让服务器更加难以溯源。
* 该方式还可以用来做基于SYN-Flood的反射攻击（见下文）。
---

```sequence
title: SYN+ACK-Flood
participant attacker
participant reflector
participant victim

attacker->reflector: 1. sip = victim, SYN(seq = x, ack = 0)
attacker->reflector: ...(大量的SYN请求)
attacker->reflector: N. sip = victim, SYN(seq = x', ack = 0)

reflector-->victim: 2. SYN + ACK (seq = y, ack = x+1)
reflector-->victim:  ...(大量的SYN+ACK响应)
Note over reflector: 1. 系统上有大量SYN_RECV状态的连接\n 2. 占用大量的资源（内存、带宽等）
reflector-->victim:  N. SYN + ACK (seq = y', ack = x'+1)
```
* 攻击者采集网上的各种开源服务（如CDN等）,作为反射器。
* 攻击者伪造源IP为victim的源IP，向反射器发送大量的的SYN-Flood攻击包，reflector会产生大量的SYN+ACK包反射到victim上。
* 通常情况下，reflector反射的SYN+ACK包都是固定的源端口，而且还是知名的源端口，victim可以通过过滤源端口的方式解决。
* **慎用限速，如果攻击者伪造源IP，向两个微服务之间，互相发送SYN-Flood，服务是否就不可用了？**
* **如果伪造源IP为内网IP，会怎样，反射包是否会打到内网去？**
* ****
---