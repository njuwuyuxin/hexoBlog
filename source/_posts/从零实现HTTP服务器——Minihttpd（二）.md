---
title: 从零实现HTTP服务器——Minihttpd（二）
date: 2020-06-28 20:45:09
categories: 从零开始
tags:
- 计算机网络
- http
- 后端
---
----
上一篇中我们实现了接受浏览器的请求，并返回本地的网页给浏览器展示，接下来对该简单的功能进行下一步完善

### Content-Type
http响应头中非常重要的一个字段是Content-Type，它决定了浏览器如何解析返回的响应内容，如果该字段缺失则默认为text/html格式，因此我们上文返回的简单html网页并没有添加该字段，浏览器也能正常解析。
但是稍微复杂的前端页面都包含了css样式文件，JavaScript脚本文件等，如果不指定Content-Type字段，则浏览器无法正确解析这些文件（有些浏览器超强的兼容性可以一定程度上自动判断内容格式）
因此我们在返回本地的文件作为响应时，需要手动设置Content-Type字段，判断依据为文件的扩展名，这里使用了一个map维护扩展名与Content-Type映射关系。
```
void HttpResponse::init_content_type_map(){
    content_type_map.insert(pair<string,string>("html","text/html"));
    content_type_map.insert(pair<string,string>("htm","text/html"));
    content_type_map.insert(pair<string,string>("shtml","text/html"));
    content_type_map.insert(pair<string,string>("css","text/css"));
    content_type_map.insert(pair<string,string>("js","text/javascript"));
    content_type_map.insert(pair<string,string>("txt","text/plain"));
    content_type_map.insert(pair<string,string>("js","text/javascript"));
    content_type_map.insert(pair<string,string>("xml","text/xml"));

    content_type_map.insert(pair<string,string>("ico","image/x-icon"));
    content_type_map.insert(pair<string,string>("jpg","image/jpeg"));
    content_type_map.insert(pair<string,string>("jpeg","image/jpeg"));
    content_type_map.insert(pair<string,string>("jpe","image/jpeg"));
    content_type_map.insert(pair<string,string>("gif","image/gif"));
    content_type_map.insert(pair<string,string>("png","image/png"));
    content_type_map.insert(pair<string,string>("tiff","image/tiff"));
    content_type_map.insert(pair<string,string>("tif","image/tiff"));
    content_type_map.insert(pair<string,string>("rgb","image/x-rgb"));

    content_type_map.insert(pair<string,string>("mpeg","video/mpeg"));
    content_type_map.insert(pair<string,string>("mpg","video/mpeg"));
    content_type_map.insert(pair<string,string>("mpe","video/mpeg"));
    content_type_map.insert(pair<string,string>("qt","video/quicktime"));
    content_type_map.insert(pair<string,string>("mov","video/quicktime"));
    content_type_map.insert(pair<string,string>("avi","video/x-msvideo"));
    content_type_map.insert(pair<string,string>("movie","video/x-sgi-movie"));

    content_type_map.insert(pair<string,string>("woff","application/font-woff"));
    content_type_map.insert(pair<string,string>("ttf","application/octet-stream"));
}
```
这里添加了一些常用格式的Content-Type类型，后续涉及到更复杂的文件类型时对其进一步扩展

### gzip压缩
在http响应的结构中，我们常常可以看到一个名为Content-Encoding的字段，其值大多为gzip,deflate等。该字段决定的是http响应体的编码格式。目前主流浏览器均支持gzip等格式的压缩格式。使用压缩格式的最大好处就是减少网络传输的信息量，提高网页加载速度，但由于服务端多了压缩的步骤，也一定程度增加了服务器的负担（客户端单次处理时，解压的影响可以忽略不计）。
而gzip格式又是应用最广泛的一种压缩格式，其对文本内容的压缩率常常可以达到40%以上，对于html,css,javascript文件均有着非常好的压缩效果。

本文实现的Minihttpd为了增加gzip格式的压缩功能使用了zlib库，其代码均为c编写，使用方法相对简单，这里列出部分供参考
```
//raw_data为原始数据，buffer为压缩后数据存储缓冲区，buffer_size为缓冲区大小，返回值为压缩后数据字节数
uLong gzip_compress(string raw_data,Bytef*& buffer,int buffer_size){
    size_t raw_data_size = raw_data.size();
    z_stream strm;
    z_stream d_stream;
    d_stream.zalloc = NULL;
    d_stream.zfree = NULL;
    d_stream.opaque = NULL;
    d_stream.next_in = (Bytef*)raw_data.c_str();
    d_stream.avail_in = raw_data_size;
    d_stream.next_out = buffer;
    d_stream.avail_out = buffer_size;

    int ret = deflateInit2(&d_stream, Z_DEFAULT_COMPRESSION, Z_DEFLATED,
						MAX_WBITS + 16, 8, Z_DEFAULT_STRATEGY);
    if (Z_OK != ret)
    {
        Log::log("init deflate error",ERROR);
        // cout<< ret <<endl;
    }

    int err = 0;
    int flag = 0;
    for(;;) {
        if((err = deflate(&d_stream, Z_FINISH)) == Z_STREAM_END) break;
        if(flag > 3){
            stringstream ss;
            ss<< "deflate failed,errNo = "<<err;
            Log::log(ss.str(),ERROR);
            return 0;
        }

        //输出缓冲区不足，尝试扩容，最多三次扩容失败则放弃压缩
        if(err == Z_BUF_ERROR){
            flag++;
            delete buffer;
            buffer_size = buffer_size*1.5;
            buffer = new Bytef[buffer_size];
            d_stream.next_out = buffer;
            d_stream.avail_out = buffer_size;
            stringstream ss;
            ss<< "deflate buffer error,try larger buffer :"<<flag;
            Log::log(ss.str(),WARN);
        }
    }
    if(deflateEnd(&d_stream) != Z_OK){
        Log::log("deflate failed when end",ERROR);
        return 0;
    }
    return d_stream.total_out;
}
```
主要工作流程为：
1. deflateInit2()  设置压缩格式等信息
2. deflate() 进行压缩
3. deflateEnd() 压缩完毕释放临时空间等收尾工作

#### 编码格式判别
接收到http请求后，首先判断请求头中是否包含Accept-Encoding字段，如果存在，检查其后面接受的压缩格式等，决定是否使用gzip压缩（注意响应头也需要添加Content-Encoding:gzip字段）

####Github
[https://github.com/njuwuyuxin/MiniHttpd](https://github.com/njuwuyuxin/MiniHttpd)
欢迎共同学习