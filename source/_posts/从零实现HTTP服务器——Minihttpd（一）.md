---
title: 从零实现HTTP服务器——Minihttpd（一）
date: 2020-06-23 23:15:28
categories: 从零开始
tags:
- 计算机网络
- http
- 后端
---
----
## 前言
最近学习了一下Tinyhttpd的源码，对http服务器的基本工作原理有了简单的理解，而Tinyhttpd一方面年头较为久远（上个世纪的代码），另一方面基本全部由C语言实现，因此便萌生了用C++从头造轮子的想法，同时加深对TCP、HTTP等协议，socket编程等理解。

## HTTP服务器基本工作流程
一个最简单的HTTP服务器，其基本功能主要是接受来自浏览器的http请求，之后根据请求内容返回相应的http response。因此对于我们要实现的最基本的http服务器，首先要完成的就是接受请求和发送响应。

## HTTP报文格式
### HTTP请求
HTTP请求主要由请求头（header）和请求体（body）构成，中间使用了空行（\r\n）进行分隔，具体结构如图所示
![HTTP请求格式](https://s1.ax1x.com/2020/06/23/NUzklD.png)
以浏览器访问某一网站为例，在除去最开始的 “请求方法 URL 协议版本” 这一行后，剩余部分均为请求头部字段，没有列出的首行格式形如 `GET /index.html http/1.1`

![浏览器请求](https://s1.ax1x.com/2020/06/23/NUxQsJ.png)

### HTTP响应
HTTP响应结构与请求类似，分为响应头和和响应体，中间以空行分隔。
响应头首行为 “协议版本 HTTP状态码“（OK可省略），剩余均为头部字段，按需求可自行添加。
![HTTP响应格式](https://s1.ax1x.com/2020/06/23/NapnRf.jpg)

## 基本的HTTP服务器实现
在理解了http服务器的简单工作流程和http请求相关格式后，我们便可以动手编写最基础的http服务器。为了方便各模块抽象，目前主要使用三个类进行基础维护，分别为：HttpServer、HttpRequest、HttpResponse

### HttpRequest和HttpResponse
这两个类主要是便于进行http请求的解析与响应报文的构造，也可以方便的看出http请求和响应的简单结构。
HttpRequest结构如下，分别对应了请求报文格式，可以方便的读取头部各字段信息
```
class HttpRequest{
public:
    HttpRequest(string raw_data);
    inline const string get_method(){ return method; };
    inline const string get_url(){ return url; };
    inline const map<string,string>& get_header(){ return header; };
private:
    string method;  //该http请求方法
    string url;     //请求URL
    string version; //http版本
    map<string,string> header;
};
```
HttpResponse结构如下，对于通用字段单独列出方便快速设置，同时提供自定义字段设置方法，而load_from_file则提供了文件读取相关功能，主要对应于解析请求中的url字段，找到服务器上相应文件进行返回。
```
class HttpResponse{
public:
    HttpResponse(int st);
    void set_header(string key, string val);    //设置头部自定义字段
    void load_from_file(string url);
    string get_response();

    /* 基础头部字段，供快速填充，自定义字段需手动设置 */
    string Allow;
    string Content_Encoding;
    string Content_Length;
    string Content_Type;
    string Expires;
    string Last_Modified;
    string Location;
    string Refresh;
    string Set_Cookie;
    string WWW_Authenticate;
    
private:
    string version;                     //http版本
    string status;                      //http状态码
    string date;                        //response生成时间
    string server;                      //http服务器名称
    map<string,string> custom_header;   //自定义头部字段
    string generate_header();           //使用全部信息组装HTTP Response头部
    string response_body;               //返回内容体
};
```

### HttpServer
HttpServer类主要用于维护单个服务器实例，也是服务器的最核心功能。目前的基本功能便是建立套接字，接受请求并返回，其类结构为
```
class HttpServer{
public:
    HttpServer();
    HttpServer(u_short p);
    inline int get_sock_id(){ return server_sock; };
    inline u_short get_port(){ return port; };
    void start_listen();
    static void accept_request(int client_sock,HttpServer* t);
private:
    int server_sock;
    u_short port;
    string baseURL;
    void startup();
};
```
其中startup函数用于创建套接字用于之后监听请求，HTTP协议基于的是TCP协议，因此套接字需正确设置，配置端口号、本地网卡IP等信息，这里为了便于使用，如果不指定端口号会随机使用某一可用端口。
```
 int on = 1;
    struct sockaddr_in name;

    server_sock = socket(PF_INET, SOCK_STREAM, 0);    //使用TCP协议
    if (server_sock == -1)
        cerr<<"[ERROR]: create socket failed"<<endl;
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;
    name.sin_port = htons(port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);
    if ((setsockopt(server_sock, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))) < 0)  
    {  
        cerr<< "[ERROR]: setsockopt failed"<<endl;
    }
    if (bind(server_sock, (struct sockaddr *)&name, sizeof(name)) < 0)
        cerr<<"[ERROR]: bind failed"<<endl;

    if (port == 0)  /* if dynamically allocating a port */
    {
        socklen_t namelen = sizeof(name);
        if (getsockname(server_sock, (struct sockaddr *)&name, &namelen) == -1)
            cerr<<"[ERROR]: getsockname failed"<<endl;
        port = ntohs(name.sin_port);
    }
    //listen第二个参数为连接请求队列长度，5代表最多同时接受5个连接请求
    if (listen(server_sock, 5) < 0)   
        cerr<<"[ERROR]: listen failed"<<endl;
```
在创建了socket后，我们便可使用该socket监听发来的tcp数据包，从中识别出http请求，这部分工作交由start_listen()函数完成
```
cout<<"httpd running on port "<<port<<endl;
    int client_sock = -1;
    struct sockaddr_in client_name;
    socklen_t  client_name_len = sizeof(client_name);
    pthread_t newthread;

    while (1)
    {
        //accept函数用来保存请求客户端的地址相关信息
        client_sock = accept(server_sock,
                (struct sockaddr *)&client_name,
                &client_name_len);
        if (client_sock == -1)
            cerr<<"[ERROR]: accept failed"<<endl;

        thread accept_thread(accept_request,client_sock,this);
        accept_thread.join();
    }

    close(server_sock);
```
这里主要使用accept函数保存客户端socket相关信息，在接收到客户端发送的一条请求后，创建一个新的线程用于该请求的处理，具体处理部分如下，主要通过read读取原始数据存入buffer，将该信息交给HttpRequest类进行简单解析，同时利用HttpResponse类构造响应报文，使用send将该响应发送回客户端，之后关闭该套接字。
```
 int client = client_sock;
    char buf[1024];
    read(client_sock,(void*)buf,1024);
    string req(buf);
    HttpRequest request(req);
    cout<<"url: "<<request.get_url()<<endl;
    string req_url = t->baseURL + request.get_url();
    
    auto header = request.get_header();
    cout<<"[GET REQUEST]: Host = "<<header.find("Host")->second<<endl;

    HttpResponse response(200);
    response.load_from_file(req_url);
    response.Content_Type = "text/html";
    string res_string = response.get_response();
    // cout<<res_string<<endl;
    send(client,res_string.c_str(),strlen(res_string.c_str()),0);
    close(client);
```
至此一条http请求便可以被正确解析并返回，总体流程为：
- 创建server_socket
- 监听某一端口请求
- 接收数据解析请求
- 构造响应报文
- 发送响应，关闭客户端套接字

到这里一个具备基础功能的http服务器已经初具雏形，可以解析简单的http请求，同时根据请求路径读取本地的html文档进行返回，交由浏览器展示。之后我们会不断完善该服务器，实现更复杂的一些功能。

Github地址：[https://github.com/njuwuyuxin/MiniHttpd](https://github.com/njuwuyuxin/MiniHttpd)
