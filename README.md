# socket编程（应用层）

注意点：在通信过程中，套接字一定是成对出现的

内部实现：一个文件描述符，两个缓冲区，即：

**在网络通信中，套接字一定是成对出现的。**一端的发送缓冲区对应对端的接收缓冲区。我们使用同一个文件描述符索发送缓冲区和接收缓冲区。

![image-20220105190343549](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151049191.png)

## 网络字节序

**重点复习：**

小端法（本机存储）：高位（23～31 bit）存高地址，低位（0～7bit）存低地址

例子：int a = 0x12345678

| 高地址：12 |
| :--------: |
|     34     |
|     56     |
| 低地址：78 |

在网络字节序中，采用大端法（网络存储）：高位存低地址，低位存高地址

转换函数：

```c
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong); // 本地转网络，主要针对的是ip地址，因为ip地址为32位
uint16_t htons(uint16_t hostshort); // 本地转网络，针对端口，16位
uint32_t ntohl(uint32_t netlong); // 网络转本地
uint16_t ntohs(uint16_t netshort);
```

h表示host，n表示network，l表示32位长整数，主要用来转换IP地址，s表示16位短整数，主要用来转换端口号。

比如第一个就是“host to network long”，第二个是“host to network short”。

如果主机是小端字节序，这些函数将参数做相应的大小端转换然后返回，如果主机是大端字节序，这些函数不做转换，将参数原封不动地返回。

## IP地址转换函数

更新的版本

```c
#include <arpa/inet.h>
// 该函数将本地（string IP）转换为网络（binary IP）
int inet_pton(int af, const char *src, void *dst);
// 网络-> 本地
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);

af: ip协议类型，AF_INET, AF_INET6
src: ip地址
dst: 传出参数：转换后的ip地址
```

成功时返回1，失败返回0并设置errno，说明src指向的不是一个有效的ip地址。

## sockaddr数据结构

strcut sockaddr 很多网络编程函数诞生早于IPv4协议，那时候都使用的是sockaddr结构体,为了向前兼容，现在sockaddr退化成了（void *）的作用，传递一个地址给函数，至于这个函数是sockaddr_in还是sockaddr_in6，由地址族确定，然后函数内部再强制类型转化为所需的地址类型。

![Screen Shot 2022-01-05 at 19.42.18](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151049162.png)

```c
// man 7 ip
struct sockaddr_in {
	__kernel_sa_family_t sin_family; 			/* Address family */  	地址结构类型
	__be16 sin_port;					 		/* Port number */		端口号
	struct in_addr sin_addr;					/* Internet address */	IP地址
	/* Pad to size of `struct sockaddr'. */
	unsigned char __pad[__SOCK_SIZE__ - sizeof(short int) -
	sizeof(unsigned short int) - sizeof(struct in_addr)];
};
```



使用方式整理：

## socket模型创建流程图

一共三个套接字，独立一个listen套接字



![image-20220105200402968](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151049000.png)

TCP通信流程分析:

server:

1. socket()  创建socket
2. bind() 绑定服务器地址结构
3. listen()  设置监听上限
4.  accept()  阻塞监听客户端连接，accept返回的socket才是真正用来读写的socket
5.  read(fd)  读socket获取客户端数据
6.  小--大写  toupper()
7. write(fd)
8. close();

client:

1. socket()  创建socket
2. connect(); 与服务器建立连接
3. write() 写数据到 socket
4. read() 读转换后的数据。
5. 显示读取结果
6. close()

 ## 主要函数

### socket

```c
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
domain:
	AF_INET 这是大多数用来产生socket的协议，使用TCP或UDP来传输，用IPv4的地址
	AF_INET6 与上面类似，不过是来用IPv6的地址
	AF_UNIX 本地协议，使用在Unix和Linux系统上，一般都是当客户端和服务器在同一台及其上的时候使用
type:
	SOCK_STREAM 这个协议是按照顺序的、可靠的、数据完整的基于字节流的连接。这是一个使用最多的socket类型，这个socket是使用TCP来进行传输。
	SOCK_DGRAM 这个协议是无连接的、固定长度的传输调用。该协议是不可靠的，使用UDP来进行它的连接。
	SOCK_SEQPACKET该协议是双线路的、可靠的连接，发送固定长度的数据包进行传输。必须把这个包完整的接受才能进行读取。
	SOCK_RAW socket类型提供单一的网络访问，这个socket类型使用ICMP公共协议。（ping、traceroute使用该协议）
	SOCK_RDM 这个类型是很少使用的，在大部分的操作系统上没有实现，它是提供给数据链路层使用，不保证数据包的顺序
protocol:
	传0 表示使用默认协议。
返回值：
  成功：返回指向新创建的socket的文件描述符，失败：返回-1，设置errno
```

### bind函数

给socket绑定一个地址结构（IP + port）

```c
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
sockfd：
	socket文件描述符
addr:
	（struct sockaddr*)&addr
	构造出IP地址加端口号
addrlen:
	sizeof(addr)长度
返回值：
	成功返回0，失败返回-1, 设置errno
```

使用步骤：

```c
struct sockaddr_in servaddr;
bzero(&servaddr, sizeof(servaddr));
servaddr.sin_family = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
// 或者 servaddr.sin_addr.s_addr = htonl(“192.168.1.1”)；
servaddr.sin_port = htons(6666);
```

### listen函数

注意不是用来阻塞监听的函数！

```c
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
int listen(int sockfd, int backlog);
sockfd:
	socket文件描述符
backlog:
	排队建立3次握手队列和刚刚建立3次握手队列的链接数和
```

典型的服务器程序可以同时服务于多个客户端，当有客户端发起连接时，服务器调用的accept()返回并接受这个连接，如果有大量的客户端发起连接而服务器来不及处理，尚未accept的客户端就处于连接等待状态，listen()声明sockfd处于监听状态，并且最多允许有backlog个客户端处于连接待状态，如果接收到更多的连接请求就忽略。listen()成功返回0，失败返回-1。

### accept函数

```c
#include <sys/types.h> 		/* See NOTES */
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
sockdf:
	socket文件描述符
addr:
	传出参数，返回链接客户端地址信息，含IP地址和端口号
addrlen:
	传入传出参数（值-结果）,传入sizeof(addr)大小，函数返回时返回真正接收到地址结构体的大小
返回值：
	成功返回一个新的socket文件描述符，用于和客户端通信，失败返回-1，设置errno
```

三方握手完成后，服务器调用accept()接受连接，如果服务器调用accept()时还没有客户端的连接请求，就阻塞等待直到有客户端连接上来。addr是一个传出参数，accept()返回时传出客户端的地址和端口号。addrlen参数是一个传入传出参数（value-result argument），传入的是调用者提供的缓冲区addr的长度以避免缓冲区溢出问题，传出的是客户端地址结构体的实际长度（有可能没有占满调用者提供的缓冲区）。如果给addr参数传NULL，表示不关心客户端的地址。

 

## 多进程并发服务器思路分析

![image-20220109162011953](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201151049265.png)

```c
1. socket，创建套接字
2. bind，绑定地址结构到struct addr_in addr
3. listen，
4. while(1) // 接受客户端连接请求
   { 
  		cfd = accept(); 
  		pid = fork();
  		if (pid == 0) {
        close(lfd); // 关闭建立用于连接的lfd套接字（子进程不需要）
        read();
        write();
      }
  		else {
        close(cfd); // 关闭用于与客户端连接的cfd套接字（子进程不需要）
        continue;
      }
		}
5. 子进程
	      close(lfd); // 关闭建立用于连接的lfd套接字（子进程不需要）
        read();
        write();
	 父进程
     		close(cfd)
     		注册信号，捕捉函数SIGCHILD
```

**问题：为何要使用SIGCHILD信号捕捉？**

## 多线程并发服务器思路分析

线程的lfd不能关，

```c
// 多线程并发实现

#define SERV_PORT 9999
void *pthread_work(void *arg)
{
    char buf[1024];
    int* cfd = (int*)arg;
    int ret;
    int i;
    while (1)
    {
        ret = Read(*cfd, buf, sizeof(buf));
        if (ret == 0)
        {
            // close(*cfd);
            // exit(1);
            printf("Now the client %d closed\n",*cfd);
            break;
        }
        for (i = 0; i < ret; ++i)
        {
            buf[i] = toupper(buf[i]);
        }
        write(*cfd, buf, ret);
        write(STDOUT_FILENO, buf, ret);
    }
    Close(*cfd);
    return (void *)0;
}
int main(void)
{
    pthread_t tid;
    struct sockaddr_in serv_addr, cli_addr;
    pthread_attr_t th_attr;
    // 地址清零
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    int lis_fd;
    int i = 0;
    int con_fd[256];

    lis_fd = Socket(AF_INET, SOCK_STREAM, 0);
    Bind(lis_fd, (const struct sockaddr *)&serv_addr, sizeof(serv_addr));
    Listen(lis_fd, 128);
    socklen_t cli_len = sizeof(cli_addr);

    while (1)
    {
        // accept接收客户端的addr
        con_fd[i] = Accept(lis_fd, (struct sockaddr *)&cli_addr, &cli_len);
        // pthread_attr_init(&th_attr);
        pthread_create(&tid, NULL, &pthread_work, (void *)&con_fd[i]);
        pthread_detach(tid);
        i++;
    }
}
```



## read函数返回值问题

要记住read 函数的返回值：

1. 大于0 实际读到的字节数 
2. = 0 的话，已经读到了**套接字**的结尾（相当于对端已经关闭）【 ！重 ！点 ！】
3. 如果返回-1， 应进一步判断errno的值：
   1. errno = EAGAIN or EWOULDBLOCK: 设置了**非阻塞方式**读。 没有数据到达，而read函数默认是阻塞读取的。
   2. errno = EINTR 慢速系统调用被中断。补救行为：重新启动。
   3. errno = “其他情况” 异常。

### 慢速系统调用（slow system call）

指可能会永远阻塞的系统调用，该术语适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用永远无法返回，多数网络支持函数都属于这一类。如：若没有客户连接到服务器上，那么服务器的accept调用就会一直阻塞。

#### EINTR异常

早期的Unix系统，如果进程在一个慢系统调用(slow system call)中阻塞时，当捕获到某个信号且相应信号处理函数返回时，这个系统调用被中断，调用返回错误，设置errno为EINTR（相应的错误描述为“Interrupted system call”）。

#### 处理EINTR异常

有三种处理方式：

1.   人为重启被中断的系统调用
2.   在初始化信号时，设置 SA_RESTART属性（该方法对有的系统调用无效）
3.   忽略信号（让系统不产生信号中断）

主要了解一下人为重启的方式：

accept、read、write、select、和open之类的函数来说，是可以进行重启的。但connect函数是不能重启的，若connect函数返回一个EINTR错误的时候，不能再次调用它，否则将立即返回一个错误。

如 read 等待输入期间，如果收到一个信号，系统将中断read， 转而执行信号处理函数. 当信号处理返回后， 系统遇到了一个问题： 是重新开始这个系统调用， 还是让系统调用失败？**早期UNIX系统的做法是， 中断系统调用，并让系统调用失败， 比如read返回 -1， 同时设置 errno 为EINTR。**

**中断了的系统调用是没有完成的调用，它的失败是临时性的，如果再次调用则可能成功，这并不是真正的失败，所以要对这种情况进行处理，** 典型的方式为：

```c
ssize_t Read(int fd, void *ptr, size_t nbytes)
{
	ssize_t n;

again:
	if ( (n = read(fd, ptr, nbytes)) == -1) {
    // 对errno == EINTR的情况进行特殊处理
		if (errno == EINTR)
			goto again;
		else
			return -1;
	}
	return n;
}
```

对于剩余两种方式，有一个博客讲的非常好，可以看一看：[信号中断 与 慢系统调用](https://blog.csdn.net/benkaoya/article/details/17262053#:~:text=1.1.%20%E6%85%A2%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8%EF%BC%88Slow,%E8%B0%83%E7%94%A8%E5%B0%B1%E4%BC%9A%E4%B8%80%E7%9B%B4%E9%98%BB%E5%A1%9E%E3%80%82)

## 端口复用函数

使用方法：

```c
int opt = 1;
setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, (void *)&opt, sizeof(opt));
```

### 半关闭

通信双方只有一方处于关闭通信状态，也就是`fin_wait2`

## shutdown函数

shutdown在关闭多个文件描述符时，采用全关闭方法

![Screen Shot 2022-01-16 at 16.55.05](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201161655666.png)

## IO复用

不可取的服务器模型：

对于上面的为每一个请求来建立一个工作线程的服务器模型是不可取的，会浪费大量的服务器资源。

1.   阻塞：
2.   非阻塞忙轮询
3.   响应式-多路IO转接

![Screen Shot 2022-01-16 at 20.58.29](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201162058888.png)

## select函数



1.   nfds参数，监听的所有文件描述符中最大的文件描述符+1。
2.   监听集合：传入传出参数，监听读，写异常事件，传入有兴趣的监听文件描述符，**传出的是实际有事件发生的**。
     -   readfds：读文件描述符集合
     -   writefds：写文件描述符集合（NULL）
     -   exceptfds：异常文件描述符集合（NULL）
3.   timeout超时时长，三种情况：
     1.   NULL永远等下去
     2.   传入0则为非阻塞，则应该轮询
     3.   设置timeva>0，等待固定时间

返回值：

大于0，返回所有监听集合中，有发生事件的总数

0，没有满足监听条件的文件描述符

-1，则为异常。

一些辅助的函数：

```c
void FD_CLR(int fd, fd_set *set)		//把某一个fd清除出去
int FD_ISSET(int fd, fd_set *set)		//判定某个fd是否在位图中
void FD_SET(int fd, fd_set *set)		//把某一个fd添加到位图
void FD_ZERO(fd_set *set)				//位图所有二进制位置零
```

### 使用的步骤：

```c
fd_set rset;

```

![](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201182013613.png)

## 多路IO转接服务器设计思路（select函数实现）

多路IO转接服务器也叫做多任务IO服务器。该类服务器实现的主旨思想是，不再由应用程序自己监视客户端连接，取而代之由内核替应用程序监视文件。

思路如下：

![Screen Shot 2022-01-19 at 16.25.57](https://raw.githubusercontent.com/fsZhuangB/Photos_Of_Blog/master/photos/202201191626009.png)

```c
int maxfd = 0;
lfd = socket(); // 创建套接字
maxfd = lfd;
bind();
listen();
// 创建两个监听集合, rset作为传入传出参数，all_set作为参数的备份
fd_set rset, all_set; 
// 将r监听集合清空
FD_ZERO(&allset);				
// 将 lfd 添加至读集合中
FD_SET(lfd, &allset);
while (1)
{
  rset = all_set;
  ret = select(maxfd + 1, &rset, NULL, NULL, NULL);
  // 如果有新的监听事件来临
  if (ret > 0)
  {
    		if (FD_ISSET(lfd, &rset)) {				// 1 在。 0不在。
          cfd = accept（）；				// 建立连接，返回用于通信的文件描述符
          maxfd = cfd；
          FD_SET(cfd, &allset);				// 添加到监听通信描述符集合中。
			}
  }
  // 遍历感兴趣的文件描述符集合，查看哪个有读写事件
  for （i = lfd+1； i <= 最大文件描述符; i++）{
				FD_ISSET(i, &rset)				// 有read、write事件
				read（）
				// 小 -- 大
				write();
			}	
}
```



### select函数优缺点

缺点： 监听上限受文件描述符限制。 最大 1024.

检测满足条件的fd， 需要自己添加业务逻辑提高效率， 提高了编码难度，因为`select`代码里有个可以优化的地方，用数组存下文件描述符，这样就不需要每次扫描一大堆无关文件描述符了。 

优点： **跨平台**。win、linux、macOS、Unix、类Unix、mips

### select实现优化

这里来改进之前代码的问题，因为对于之前的代码，如果最大fd是1023，每次确定有事件发生的fd时，就要扫描3-1023的所有文件描述符，这看起来很蠢。于是定义一个数组，把要监听的文件描述符存下来，每次扫描这个数组就行了。看起来科学得多。



