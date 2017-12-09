# tinyhttpd学习

* 由客户端程序和服务器程序构成，服务器程序由web服务器实现

## 相关文件函数

###  sys/socket.h

Socket是应用层与TCP/IP协议族通信的中间软件抽象层，**它是一组接口**。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。

![socket原理](D:\BLOG\source\_drafts\http协议\socket原理.PNG)

###struct sockaddr结构体

struct sockaddr和struct sockaddr_in这两个结构体用来处理网络通信的地址。

在各种系统调用或者函数中，只要和网络地址打交道，就得用到这两个结构体。

网络中的地址包含3个方面的属性：

1 地址类型: ipv4还是ipv6

2 ip地址

3 端口

例如

```c
struct sockaddr {
    unsigned short    sa_family; //通信类型，最常用的是"AF_INET"
    char              sa_data[14]; // 目标地址 和 端口信息
                };
//sockaddr缺陷：将ip和端口号放到一起
struct sockaddr_in {
    short sin_family;       // 通信类型 e.g. AF_INET,AF_INET6
    unsigned short  sin_port;    // 端口号 e.g.htons(3490)
    struct in_addr sin_addr;     //ip地址
    char sin_zero[8];     // 8 bytes zero this if you want to
                    };

//ip地址结构体  多余结构体？
struct in_addr{
    unsigned long s_addr;
               };
```



### socket( )

​      socket()函数的原型如下，这个函数建立一个协议族为domain、协议类型为type、协议编号为protocol的**套接字文件描述符**。如果函数调用成功，会返回一个标识这个套接字的文件描述符，失败的时候返回-1。socket函数是任何套接口网络编程中第一个使用的函数，它向用户提供一个套接字，即套接口描述文件字，它是一个整数，如同文件描述符一样，是**内核标识一个IO结构的索引**(UID?)。通过socket函数，我们指定一个套接口的协议相关的属性，为进行使用socket api做好准备。

```c
int socket(int domain, int type, int protocol);
```

* domain用于设置网络通信的域，函数socket()根据这个参数选择通信协议的族。通信协议族在文件sys/socket.h中定义。

* type用于设置套接字通信的类型，主要有SOCKET_STREAM（流式套接字）、SOCK——DGRAM（数据包套接字）等

* protocol用于制定某个协议的特定类型，即type类型中的某个类型。通常某协议中只有一种特定类型，这样protocol参数仅能设置为0；但是有些协议有多种特定的类型，就需要设置这个参数来选择特定的类型

* [具体见此处](http://blog.csdn.net/xc_tsao/article/details/44123331)...

### htonl( )

​         htonl把本机字节顺序转化为网络字节顺序，网络字节顺序（大尾顺序）就是指一个数在内存中存储的时候“高对低，低对高”（即一个数的高位字节存放于低地址单元，低位字节存放在高地址单元中）。但是计算机的内存存储数据时有可能是大尾顺序或者小尾顺序。

###bind( ) 

​        在建立套接字文件描述符成功后，需要对套接字进行地址和端口的绑定，才能进行数据的接收和发送操作。

函数原型:
`int bind(int sockfd, struct sockaddr * my_addr, int addrlen);`

* sockfd: socket file descriptor 套接字文件描述符
* struct sockaddr:网络通信的地址 ，协议类型，ip，端口
* 成功则返回0, 失败返回-1, 错误原因存于errno 中

### getsockname( )

​         用getsockname获得一个与socket相关的地址，服务器端可以通过它得到相关客户端地址，而客户端也可以得到当前已连接成功的socket的ip和端口。可用于验证某个socket是否存在。
[具体见此处](https://baike.baidu.com/item/getsockname()/10081920)

### listen ( )

listen函数使用主动连接套接口变为被连接套接口，使得一个进程可以接受其它进程的请求，从而成为一个服务器进程。在TCP服务器编程中listen函数把进程变为一个服务器，并指定相应的套接字变为被动连接。

函数原型：`int listen(int sockfd, int backlog);`

* sockfd：socket file descriptor 套接字文件描述符

  ​        在被socket函数返回的套接字sockfd之时，它是一个**主动连接的套接字**，也就是此时系统假设用户会对这个套接字调用connect函数，期待它主动与其它进程连接，然后在服务 器编程中，用户希望这个套接字可以接受外来的连接请求，也就是被动等待用户来连接。由于系统默认时认为一个套接字是主动连接的，所以需要通过某种方式来告 诉系统，用户进程通过系统调用listen来完成这件事。

* backlog：连接请求队列(queue of pending connections)的最大长度

  ​        在进程正理一个一个连接请求的时候，可能还存在其它的连接请求。因为TCP连接是一个过程，所以可能存在一种半连接的状态，有时由 于同时尝试连接的用户过多，使得服务器进程无法快速地完成连接请求。如果这个情况出现了，服务器进程希望内核如何处理呢？**内核会在自己的进程空间里维护一 个队列以跟踪这些完成的连接**，但服务器进程还没有接手处理或正在进行的连接，这样的一个队列内核不可能让其任意大，所以必须有一个大小的上限。这个 backlog告诉内核使用这个数值作为上限。

* **listen函数一般在调用bind之后、调用accept之前调用**

 ### accept ( )

​     当套接字处于监听（listen）状态时，可以通过 accept() 函数来接收客户端请求。accept() 返回一个新的套接字来和客户端通信，addr 保存了**客户端的IP地址和端口号**，而 sock 是服务器端的套接字，

函数原型：
`int accept(int sever_sock, struct sockaddr *addr, socklen_t *addrlen);`

* 输入 sever_sock

* 返回client_sockfd

* struct sockaddr: 客户端ip和端口

###pthread

​        pthread定义了创建和操纵线程的一套API。具体Pthreads定义了一套C语言的类型、函数与常量，它以`pthread.h`头文件和一个线程库实现。Pthreads API中大致共有100个函数调用，全都以"pthread_"开头，并可以分为四类：
- 线程管理，例如创建线程，等待(join)线程，查询线程状态等。
- 互斥锁（Mutex）：创建、摧毁、锁定、解锁、设置属性等操作
- 条件变量（Condition Variable）：创建、摧毁、等待、通知、设置与查询属性等操作
- 使用了互斥锁的线程间的同步管理

####pthread_create( )

创建线程，若成功则返回0，否则返回出错编号

函数原型：
`　int pthread_create(pthread_t*restrict tidp,const pthread_attr_t *restrict_attr,void*（*start_rtn)(void*),void *restrict arg);`

* tidp：线程ID，指向线程标识符的指针

* restrict_attr：线程属性

* (\*start_rtn)(void*)：线程运行函数的起始地址(函数名是函数的入口的指针)

* arg：运行函数的参数

* e.g.: `pthread_create(&newthread , NULL, accept_request, client_sock)`

  ​

###HTTP报文

请求

```http
GET /webDemo/Hellow HTTP/1.1  //请求行
Host: localhost:8080      //请求头
Connection: keep-alive
Accept: text/html
Accept-Encoding: gzip, deflate, sdch, br
Accept-Language: zh-CN,zh;q=0.8
                //空行，表示请求头已经结束
data                //实体内容，GET方法此项为空。POST方法为提交的数据
```

响应

```http
HTTP/1.1 200 OK       //响应行
Server: Apache-Coyote/1.1  //相应头
Content-Length: 0
Date: Thu, 18 May 2017 13:21:23 GMT
              //空行，表示响应头已经结束
data              //实体内容
```

### recv ( )

函数原型：`int recv( SOCKET s, char *buf, int  len, int flags)`

* s: 接收端套接字描述符

* \*buf：缓冲区，用来存放recv函数接收到的数据

* len：指明buf的长度

* flags ：flags取值有：

  * 0：常规操作，与read()相同
  * MSG_DONTWAIT:将单个I／O操作设置为非阻塞模式
  * MSG_OOB:指明发送的是带外信息
  * MSG_PEEK:可以查看可读的信息，在接收数据后不会将这些数据丢失
  * MSG_WAITALL:通知内核直到读到请求的数据字节数时，才返回。

* 成功执行时，返回接收到的字节数。

* 另一端已关闭则返回0

* 失败返回-1

###send ( )

函数原型：`int send(int s, const void *msg, size_t len, int flags);`

- s: 接收端套接字描述符

- \*buf：缓冲区，用来存放send函数接收到的数据

- len：指明buf的长度

- flags ：flags取值有：

  * 0： 与write()无异

  * MSG_DONTROUTE:告诉内核，目标主机在本地网络，不用查路由表

  * MSG_DONTWAIT:将单个I／O操作设置为非阻塞模式

  * MSG_OOB:指明发送的是带外信息

###CGI

CGI(Common Gateway Interface)是一个用于定Web服务器与外部程序之间通信方式的标准，其程序须运行在网络服务器上

### GET方法

GET方法是默认的HTTP请求方法，用GET方法提交的数据将作为URL的一部分向Web服务器发送。?之后的内容为参数，不同参数之间用&隔开 。（php获取get参数？）
例如：
http://news.baidu.com/ns?word=akg&tn=news&from=news&cl=2&rn=20&ct=1

###stat()

定义： int stat(const char *file_name, struct stat *buf); 

通过文件名filename获取文件信息，并保存在buf所指的结构体stat中 。执行成功则返回0，失败返回-1，错误代码存于errno（需要include \<errno.h>）



## 执行流程

* starup：在相应端口启动httpd服务
  * 用socket(int domain, int type, int protocol) 指定一个套接口的协议相关的属性，为进行使用socket api做好准备

  * 用struct sockaddr指定网络通信地址

  * 用bind()将socket套接字和一个IP地址及端口绑定

  * 如果端口是 0，随机分配一个端口 。

  * 用listen( )进行监听

  * 返回**套接字描述符** （socket file descriptor）给主函数

    ​
* while( 1 )
  * 
    `client_sock = accept( int sockfd, struct sockaddr *addr, socklen_t *addrlen);`
    不断从已完成的服务端连接队列返回下一个已完成连接

  * `pthread_create(&newthread , NULL, accept_request, client_sock)`

     用pthread_creat( )派生新线程，用accept_request( )处理新请求

     **accept_request(client_sockfd )**

       * `numchars = get_line( );//得到请求的第一行 `

       * 方法不是GET或者POST， 则用unimplemented( )返回错误信息（http请求方法？？）    
         * unimplemented( )发送http 错误代码 501，返回http相应报文

       * 若为POST方法则启用CGI

       * 用get_line( )获取url，若有？开启CGI

       * 根据url生成资源路径path

       * 用stat()获取路径path对应文件(夹)信息

       * 如果路径是目录则返回该目录下的index.html

       * 如果是文件，先判断文件是否为cgi文件（是否有可执行权限）

       * 若是cgi，则用execute_cgi( )执行该cgi

         **execute_cgi( )**

         * 若method是GET，则完全丢弃请求头部
         * 若method是POST，则找出：Content-Length
         * Content-Length没有找到，用bad_request( )返回 400 bad request
         * 否则 返回 200，开始执行cgi
         * .......待续

         ​

       * 如果不是cgi，则返用serve_file( )回该文件

       * 若没有找到相应文件或者目录，则用not_found()返回404 not fount

       * 关闭http连接
* 结束服务