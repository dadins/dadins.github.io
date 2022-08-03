socket.md

```mermaid
sequenceDiagram
    participant __sys_listen()
    participant sockfd_lookup_light()
    participant inet_listen()

    autonumber
    __sys_listen()->>sockfd_lookup_light(): 根据fd查找socket内核对象
    sockfd_lookup_light()-->>__sys_listen(): 返回一个struct socket结构体
    note over __sys_listen(): 1. 获取backlog值 backlog=min(backlog, sysctl_somaxconn)<br> 2. 其中默认的somaxconn值为4096
    __sys_listen()->>inet_listen():执行真正的监听过程（sock->ops->listen(sock, backlog)）
    note over inet_listen(): 设置全连接队列长度(sk->sk_max_ack_backlog)
    inet_listen()->> inet_csk_listen_start(): 执行真正的监听过程
    inet_csk_listen_start() ->> reqsk_queue_alloc(): 分配接收队列(全连接队列: icsk->icsk_accept_queue)
    note over inet_csk_listen_start(): sk->sk_ack_backlog初始化(全连接队列的长度)
    inet_csk_listen_start()-->> inet_listen(): 监听过程返回
    inet_listen()-->>__sys_listen():返回socket的listen结果
```




```mermaid
sequenceDiagram
    participant __sys_accept4()
    participant __sys_accept4_file()
    participant inet_accept()
    participant inet_csk_accept()

    autonumber
    __sys_accept4()->>__sys_accept4_file(): 调用真正的accept流程
    note over __sys_accept4_file(): 1. 分配新的socket对象(newsock) <br> 2. 获取一个新的fd(newfd)
    __sys_accept4_file()->>inet_accept():从三次握手完成后的全连接队列里面取一个连接出来
    inet_accept() --> inet_csk_accept(): 从sk1(accept对应的socket)的全连接队列里面取一个连接出来
    note over inet_csk_accept(): 1. 这里涉及到了socket的阻塞、非阻塞判断<br> 2. 在inet_csk_wait_for_connect()中涉及到了进程调度，wait exclusive的处理(accept的惊群处理)
    note over inet_csk_accept(): 从全连接队列取一个连接下来，并将sk->sk_ack_backlog减1
```


##TODO:
1. socket的半连接队列
2. epoll惊群现象
