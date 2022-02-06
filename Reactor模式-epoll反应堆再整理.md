# Reactor模式-epoll反应堆再整理

## epoll的ET非阻塞模式

思考下面一种场景：

假设使用readn()函数去读取socket上的500个字节的数据，但是此时只读取到了498个字节，此时如果是默认的阻塞IO的socket，则会一直阻塞在下面的readn()函数上，导致一直无法返回，即使后面客户端再发两个字节的数据过来，由于readn()函数一直未返回，所以epoll_wait()函数也不会去通知readn()继续进行读，这样就会出现问题。所以ET只会支持非阻塞的方式。

![Screen Shot 2022-02-06 at 21.37.06](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202202062137374.png)

ET只支持非阻塞模式，利用fcntl来设置，在读数据的时候进行设置：

```c
flag = fcntl(connfd, F_GETFL);          /* 修改connfd为非阻塞读 */  
flag |= O_NONBLOCK;  
fcntl(connfd, F_SETFL, flag);  
```

## 参考文献

1.   游双《Linux高性能服务器编程》
2.   [如何深刻理解Reactor和Proactor？](https://www.zhihu.com/question/26943938)