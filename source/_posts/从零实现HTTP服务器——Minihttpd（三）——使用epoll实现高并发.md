---
title: 从零实现HTTP服务器——Minihttpd（三）——使用epoll实现高并发
date: 2020-06-29 23:17:08
categories: 从零开始
tags:
- 计算机网络
- http
- 后端
---
----
在实现了基本的接受请求，返回响应这一基本功能后，我们尝试提高该服务器能同时处理的并发请求数，实现面对海量请求时的高并发处理，主要使用了linux下的epoll机制。本文主要对epoll的基本原理进行讲解，同时展示epoll简单的使用方法。

## epoll
linux下的多路复用IO接口主要有select、poll和epoll，其中epoll是对select和poll的改进。所谓多路复用IO接口，就是当需要处理大批量文件描述符时使用的系统接口。文件、管道IO、socket等均使用了文件描述符，而epoll因为在处理大批量文件描述符时的高效性而收到了广泛的应用。

## epoll高效的原因
传统的select、poll方式，通过维护一个文件描述符数组，将所有的文件描述符统一管理，而在某一时刻，某一文件描述符可能有多种状态：缓冲区有内容可读、缓冲区有内容可写、空闲等等情况。而传统方式每次返回所有文件描述符，对所有描述符进行轮询处理，判断哪些描述符被“激活”需要进行处理。
显然，使用轮询的方式，其算法复杂度为O(n)级别，也就是说随着文件描述符的增长，其耗时也为线性增长，这就导致其难以处理高并发请求的情况（海量文件描述符），因此select方式限制了文件描述符的最大数量为1024.

而epoll最大的特点是通过epoll_wait函数，每次返回的是已就绪的文件描述符列表，而所有空闲的文件描述符并不进行返回。这首先避免了大量文件描述符从内核态拷贝到用户态内存的开销，同时避免了轮询请求大量无用的判断，其算法复杂度为O(1)级别。

### epoll的实现方式
- epoll能够“选择性”的返回就绪态的文件描述符，主要依赖于其底层的数据结构。
epoll使用了一个红黑树维护所有文件描述符的集合，这为查询，插入，删除某一描述符提供了很大便利。
- 另外epoll使用了一个双向链表用于维护当前所有就绪态的文件描述符，每次调用epoll_wait函数时，就是将该双向链表中的文件描述符返回。
- 同时epoll使用了回调机制，在把文件描述符加入到epoll红黑树中的同时，注册了相应的一些事件（如收到消息），当某一描述符的事件被触发，则将该描述符加入至双向链表中。
通过这样的数据结构，使得epoll每次不必返回所有的文件描述符，降低了算法复杂度的同时节约了大量系统资源，在并发请求数线性增长的情况下，其复杂度并不会线性增加，从而轻松实现百万级别的高并发处理。

epoll机制的详细解读可以参考这篇博客:
[https://blog.csdn.net/daaikuaichuan/article/details/83862311](https://blog.csdn.net/daaikuaichuan/article/details/83862311)

### 使用案例
在本文实现的http服务器中，我们尝试使用epoll修改底层处理逻辑，主要修改了 server类下的start_listen函数。

**epoll的核心api主要有三个**
- int epoll_create(int size)  
用于初始化epoll描述符，参数为epoll事件列表最大值（在linux内核2.6版本之后已弃用，可以忽略）
-  int epoll_ctl(int epfd， int op， int fd， struct epoll_event *event) 
用于将某一描述符注册到epoll内核事件表中
-  int epoll_wait(int epfd， struct epoll_event *events， int maxevents， int timeout) 
用于获取当前就绪的文件描述符

对于这三个api的详细内容可以参考下面这篇博客：
[https://www.jianshu.com/p/31cdfd6f5a48](https://www.jianshu.com/p/31cdfd6f5a48)

需要注意的是，epoll支持LT、ET两种模式
- LT 水平触发，即当某一描述符就绪时，每次epoll_wait均会将其返回
- ET 边缘触发，即当某一描述符由空闲转换为就绪时，epoll_wait将其返回一次，之后无视

这两种模式类似数字信号中电平的高低，LT模式类似高电平时持续触发，而ET模式则在低电平转换成高电平这一“边缘”时触发一次。
其区别主要是LT模式可以不一次性将描述符缓冲区读完，下次epoll_wait仍然会将其返回可以继续读取。
而MT模式则必须一次性将缓冲区内容读完，否则即使仍有内容未读，epoll_wait也不会返回该描述符，造成内容丢失。
**epoll默认使用LT模式**


### demo
```
void HttpServer::add_epoll_fd(int event_fd){
    epoll_event event;
    event.data.fd = event_fd;
    event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, event_fd, &event);
}

void HttpServer::start_listen(){
    stringstream ss;
    string s_port;
    ss<<port;
    ss>>s_port;
    Log::log("Minihttpd running on port "+s_port,INFO);

    epollfd = epoll_create(5);
    add_epoll_fd(server_sock);      //把监听socket加入内核事件表

    epoll_event events[MAX_EVENT_NUMBER];
    while(1){
        int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        stringstream s;
        string s_number;
        s<<number;
        s>>s_number;
        Log::log("current tcp links="+s_number,DEBUG);
        for (int i = 0; i < number; i++){
            int sockfd = events[i].data.fd;

            //处理新到的客户连接
            if (sockfd == server_sock){
                int client_sock = -1;
                struct sockaddr_in client_name;
                socklen_t  client_name_len = sizeof(client_name);
                client_sock = accept(server_sock,
                        (struct sockaddr *)&client_name,
                        &client_name_len);
                if (client_sock == -1){
                    Log::log("accept failed",ERROR);
                    continue;
                }
                add_epoll_fd(client_sock);      //把客户端socket加入内核事件表
            }
            else if (events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)){
                Log::log("epoll end",DEBUG);
                //服务器端关闭连接
            }
            //处理客户连接上接收到的数据
            else if (events[i].events & EPOLLIN){
                thread accept_thread(accept_request,sockfd,this);
                accept_thread.detach();
            }
            else if (events[i].events & EPOLLOUT){
                Log::log("epoll out",DEBUG);
            }
        }
    }

    close(server_sock);
}
```
其中epoll_event结构体结构为
```
typedef union epoll_data {
    void *ptr; /* 指向用户自定义数据 */
    int fd; /* 注册的文件描述符 */
    uint32_t u32; /* 32-bit integer */
    uint64_t u64; /* 64-bit integer */
} epoll_data_t;

struct epoll_event {
    uint32_t events; /* 描述epoll事件 */
    epoll_data_t data; /* 见上面的结构体 */
};
```

### Github
[https://github.com/njuwuyuxin/MiniHttpd](https://github.com/njuwuyuxin/MiniHttpd)