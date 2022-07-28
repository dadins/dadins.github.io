# 空连接防御方案

* proxy作为反向代理，正常客户端工作流程如下：

```mermaid
sequenceDiagram
    participant clients
    participant Proxy
    participant server

    autonumber
    clients->Proxy: SYN (seq = X, ack = 0)
    Proxy-->clients: SYN+ACK (seq = Y, ack = X+1) 
    note left of Proxy: Y = hash(sip + sport + timestamp)
    clients->Proxy: ACK (seq = X + 1, ack = Y + 1)
    note over Proxy: 直接drop(不消耗任何资源)
    clients->Proxy: PSH+ACK(seq = X + 1, ack = Y + 1, payload=200)
    note left of Proxy: 通过syncookie算法直接校验ack是否为hash(sip+sport+timestamp) + 1
    note left of Proxy: 校验通过，创建连接表
    
    Proxy->server:  SYN (seq = X, ack = 0)
    server-->Proxy: SYN + ACK (seq = Z, ack = X+1) 
    note right of Proxy: ack映射(Y <--> Z)
    Proxy->server: PSH + ACK (seq = X + 1, ack = Z + 1, payload=200)

```
--- 
```mermaid
  sequenceDiagram
    participant bots
    participant Proxy
    participant server

    autonumber

    loop 1000 times
        bots->Proxy: SYN (seq = X, ack = 0)
        Proxy-->bots: SYN+ACK (seq=hash(sip + sport + timestamp),  ack = X+1) 
        bots->Proxy: ACK (seq = X+1, ack = Y+1)
        note left of Proxy: 直接drop(不消耗任何资源，无需记录连接表)
    end
```
```
核心原理:延迟syncookie的校验过程
三次握手最后一个ACK(seq=X+1、ack=Y+1)
三次握手完的第一个请求(seq=X+1、ack=Y+1)
如果客户端是正常的，我们校验完第一个payload包里面的ack号后再创建连接。
如果客户端是异常的，我们就省了最后的连接表了。
```
