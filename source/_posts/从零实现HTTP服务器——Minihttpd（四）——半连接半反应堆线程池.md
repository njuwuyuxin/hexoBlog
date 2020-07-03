---
title: 从零实现HTTP服务器——Minihttpd（四）——半连接半反应堆线程池
date: 2020-07-03 12:31:42
categories: 从零开始
tags:
- 计算机网络
- http
- 后端
---
----
在我们使用了epoll实现了上万并发请求的处理后，我们开始考虑程序中存在的另一瓶颈，即多线程处理请求时存在的问题。
在之前的代码中，当收到了客户端的一条请求后，我们是这样做的
```
 //处理客户连接上接收到的数据
else if (events[i].events & EPOLLIN){
  thread accept_thread(accept_request,sockfd,this);
  accept_thread.detach();
}
```
每次收到一条请求时，我们都**创建了一个新的线程**，去执行这条请求的处理。
- 对于低并发量的情况，同时I/O操作密集型的线程函数，这样的方式还基本可以接受，因为此时程序运行效率的瓶颈主要在I/O相关操作上，此时为每个请求创建一个新的线程是可以接受的。
- 但是当遇到高并发请求，同时每个线程函数执行的内容相对简单，为CPU密集型函数时，每次创建新的线程就会产生非常明显的效率问题——CPU大量时间用于线程创建和线程切换上。

为了解决这个问题，一个经典的方案是使用“线程池”进行多线程操作，我们设定好线程池中初始线程数量，在初始化阶段让所有线程运行起来，避免反复创建线程、销毁线程造成的额外开销

### 半连接半反应堆线程池
本项目中实现了一个半连接半反应堆线程池，其主要工作原理如下图
![半连接半反应堆线程池](https://upload-images.jianshu.io/upload_images/16734657-09d5ea66198dac77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
实际上就是主线程用于接受客户端请求，将所有请求放入请求队列中（该队列对所有工作线程可见），各个工作线程以竞争方式读取工作队列中的请求进行处理。
这里主要涉及到线程同步的问题，由于主线程和各个工作线程都需要对工作队列进行操作，因此需要保证同一时间只有一个线程对工作队列进行操作

#### 互斥锁
保证线程同步的一个基本方法是使用互斥锁，通过对关键区代码进行“加锁”，“解锁”操作，保证同一时间只有一个线程访问到关键区代码。当一个线程想要对关键区代码进行“加锁”操作，但该段代码已处于“锁定”状态，则该线程会被阻塞住，等待这段代码被释放后再获取操作权。

具体代码实现如下，这里为了便于线程管理，抽象出了一个线程池对象
```
//ThreadPool.h
class ThreadPool{
public:
    ThreadPool(HttpServer* server, int workthread = 8);
    void append(int sockfd);                //把事件加入请求队列
    void init();                            //创建N个工作线程并运行
    void init(int count);                   //手动指定创建N个工作线程并运行
    static void work(ThreadPool* pool);     //运行工作线程
private:
    HttpServer* http_server;                //与之绑定的HttpServer对象
    int thread_count;                       //线程池线程数
    queue<int> request_list;                //请求队列
    Sem request_list_sem;                   //请求队列信号量
    mutex request_list_mutex;               //请求队列互斥锁

    void run();                             //每个工作线程执行函数
};

//ThreadPool.cpp
void ThreadPool::init(int count){
    thread_count = count;
    for(int i=0;i<thread_count;i++){
        thread work_thread(ThreadPool::work,this);
        work_thread.detach();
    }
}

void ThreadPool::run(){
    while(1){
        request_list_mutex.lock();
        if(request_list.empty()){
            request_list_mutex.unlock();
            continue;
        }
        int sockfd = request_list.front();
        request_list.pop();
        request_list_mutex.unlock();
        HttpServer::accept_request(sockfd,http_server);    //请求处理
    }
}
```
在init函数初始化阶段创建N个线程并设为分离态，使各工作线程开始运作。
每个工作线程循环读取请求队列，同时进行加解锁操作保证线程同步，之后进行相应的请求处理。
至此我们实现了基本的，多个工作线程以竞争方式处理请求的线程池。


### 存在问题
使用线程池代替了每次创建线程的操作后，使用压力测试进行性能检验，却发现在面对高并发请求时，使用这样的线程池，**反而大大的降低了程序的吞吐量，造成了严重的性能问题**

在每次创建线程池，面对上万并发请求时，其吞吐量大约为80w QPS左右，但使用线程池后，吞吐量骤降为8w QPS左右，降低了整整一个数量级。

重新审视我们实现线程池的代码，发现了一个非常明显的问题：
**当请求队列为空时，各个工作线程持续不断的对请求队列进行加锁、解锁操作，同时与主线程发生竞争，导致工作队列长时间被工作线程抢夺，却并未执行有意义的操作。而主线程却请求队列被阻塞而无法把新的请求添加入队列。**
为了解决这个问题，这里使用了信号量机制

#### 信号量
使用信号量机制可以实现一个简单的“生产者——消费者”模型，其工作流程主要是：
- 当主线程向请求队列中加入请求时，使信号量+1
- 工作线程每次循环首先使用sem_wait 等待信号量，若信号量=0，则代表队列中无请求需要处理，此时线程睡眠。若信号量>0，则线程被唤醒，同时信号量-1，代表消耗掉一个请求。

使用这样一个“生产者——消费者”模型，可以实现在请求队列为空时，各工作线程处于休眠态，避免不必要的竞争和阻塞。而当有请求需要处理时，又可以将工作线程唤醒进行工作。

具体代码实现也非常简单，其中Sem为本文对c原生semaphore操作进行的封装类
```
//主线程调用，把新的请求加入请求队列
void ThreadPool::append(int sockfd){
    request_list_mutex.lock();
    request_list.push(sockfd);
    request_list_mutex.unlock();
    request_list_sem.post();    //信号量+1
}

void ThreadPool::run(){
    while(1){
        request_list_sem.wait();    //等待信号量>0，并消耗
        request_list_mutex.lock();
        if(request_list.empty()){
            request_list_mutex.unlock();
            continue;
        }
        int sockfd = request_list.front();
        request_list.pop();
        request_list_mutex.unlock();
        HttpServer::accept_request(sockfd,http_server);
    }
}
```
此时使用Webbench进行压力测试，测试10000并发请求时，测试结果显示，此时吞吐量约为250w QPS，其效率相比单纯用互斥锁进行同步有了极大提升，相比每次创建线程也有了明显提升。
![压力测试结果](https://upload-images.jianshu.io/upload_images/16734657-6e536bfc401efe83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 运行测试时，服务器启用32个工作线程，具体线程数需要针对服务器cpu核心数，I/O操作和CPU操作的时间占比等进行制定。

### Github
[https://github.com/njuwuyuxin/MiniHttpd](https://github.com/njuwuyuxin/MiniHttpd)