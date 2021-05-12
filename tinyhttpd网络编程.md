## socket编程
### 1. socket()函数
```c
int socket(int protofamily, int type, int protocol);//返回sockfd
```
- sockfd是描述符。socket函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述字，而socket()用于创建一个socket描述符（socket descriptor），它唯一标识一个socket。这个socket描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。
-  创建socket的时候，可以指定不同的参数创建不同的socket描述符，socket函数的三个参数分别为：
-   protofamily：即协议域，又称为协议族（family）。常用的协议族有，AF_INET(IPV4)、AF_INET6(IPV6)、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。
-  type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等。
-  protocol：故名思意，就是指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议（这个协议我将会单独开篇讨论！）。
-  注意：并不是上面的type和protocol可以随意组合的，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当protocol为0时，会自动选择type类型对应的默认协议。
- 当我们调用socket创建一个socket时，返回的socket描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用bind()函数，否则就当调用connect()、listen()时系统会自动随机分配一个端口。

### 2. bind()函数
- 正如上面所说bind()函数把一个地址族中的特定地址赋给socket。例如对应AF_INET、AF_INET6就是把一个ipv4或ipv6地址和端口号组合赋给socket。
```c
  int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
-  函数的三个参数分别为：
-  sockfd：即socket描述字，它是通过socket()函数创建了，唯一标识一个socket。bind()函数就是将给这个描述字绑定一个名字。
-  addr：一个const struct sockaddr *指针，指向要绑定给sockfd的协议地址。
-  addrlen：对应的是地址的长度。
-  通常服务器在启动的时候都会绑定一个众所周知的地址（如ip地址+端口号），用于提供服务，客户就可以通过它来接连服务器；而客户端就不用指定，有系统自动分配一个端口号和自身的ip地址组合。这就是为什么通常服务器端在listen之前会调用bind()，而客户端就不会调用，而是在connect()时由系统随机生成一个。

### 3. listen()、connect()函数
- 如果作为一个服务器，在调用socket()、bind()之后就会调用listen()来监听这个socket，如果客户端这时调用connect()发出连接请求，服务器端就会接收到这个请求。
```c
 int listen(int sockfd, int backlog);
 int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- listen函数的第一个参数即为要监听的socket描述字，第二个参数为相应socket可以排队的最大连接个数。socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。
- connect函数的第一个参数即为客户端的socket描述字，第二参数为服务器的socket地址，第三个参数为socket地址的长度。客户端通过调用connect函数来建立与TCP服务器的连接。

### 4. accept()函数
- TCP服务器端依次调用socket()、bind()、listen()之后，就会监听指定的socket地址了。TCP客户端依次调用socket()、connect()之后就向TCP服务器发送了一个连接请求。TCP服务器监听到这个请求之后，就会调用accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络I/O操作了，即类同于普通文件的读写I/O操作。
 ```c
 int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen); //返回连接connect_fd
 ```
- 参数sockfd，就是上面解释中的监听套接字，这个套接字用来监听一个端口，当有一个客户与服务器连接时，它使用这个一个端口号，而此时这个端口号正与这个套接字关联。当然客户不知道套接字这些细节，它只知道一个地址和一个端口号。
- 参数addr,这是一个结果参数，它用来接受一个返回值，这返回值指定客户端的地址，当然这个地址是通过某个地址结构来描述的，用户应该知道这一个什么样的地址结构。如果对客户的地址不感兴趣，那么可以把这个值设置为NULL。
- 参数len,它也是结果的参数，用来接受上述addr的结构的大小的，它指明addr结构所占有的字节个数。同样的，它也可以被设置为NULL。
- 如果accept成功返回，则服务器与客户已经正确建立连接了，此时服务器通过accept返回的套接字来完成与客户的通信。
- 注意：
     - accept默认会阻塞进程，直到有一个客户连接建立后返回，它返回的是一个新可用的套接字，这个套接字是连接套接字。
     - 此时我们需要区分两种套接字，
     -  监听套接字: 监听套接字正如accept的参数sockfd，它是监听套接字，在调用listen函数之后，是服务器开始调用socket()函数生成的，称为监听socket描述字(监听套接字)
     -  连接套接字：一个套接字会从主动连接的套接字变身为一个监听套接字；而accept函数返回的是已连接socket描述字(一个连接套接字)，它代表着一个网络已经存在的点点连接。

　　  一个服务器通常通常仅仅只创建一个监听socket描述字，它在该服务器的生命周期内一直存在。内核为每个由服务器进程接受的客户连接创建了一个已连接socket描述字，当服务器完成了对某个客户的服务，相应的已连接socket描述字就被关闭。
      自然要问的是：为什么要有两种套接字？原因很简单，如果使用一个描述字的话，那么它的功能太多，使得使用很不直观，同时在内核确实产生了一个这样的新的描述字。
　　连接套接字socketfd_new 并没有占用新的端口与客户端通信，依然使用的是与监听套接字socketfd一样的端口号

### 5. read()、write()等函数
- 万事具备只欠东风，至此服务器与客户已经建立好连接了。可以调用网络I/O进行读写操作了，即实现了网咯中不同进程之间的通信！网络I/O操作有下面几组：
 ```c
 read()/write()；recv()/send()；readv()/writev()；recvmsg()/sendmsg()；recvfrom()/sendto()
 int send( SOCKET s, const char FAR *buf, int len,int flags);  
 ```
- 不论是客户还是服务器应用程序都用send函数来向TCP连接的另一端发送数据。
- 客户程序一般用send函数向服务器发送请求，而服务器则通常用send函数来向客户程序发送应答。
- 该函数的第一个参数指定发送端套接字描述符；第二个参数指明一个存放应用程序要发送数据的缓冲区；第三个参数指明实际要发送的数据的字节数；第四个参数一般置0。

```c
int recv( SOCKET s, char FAR *buf,  int len,int flags);  
```
- 不论是客户还是服务器应用程序都用recv函数从TCP连接的另一端接收数据。
- 该函数的第一个参数指定接收端套接字描述符；第二个参数指明一个缓冲区，该缓冲区用来存放recv函数接收到的数据；第三个参数指明buf的长度；第四个参数一般置0。返回值是接收到的字节数，如果接收时出错，那么返回SOCKET_ERROR.

### 6. close()函数
- 在服务器与客户端建立连接之后，会进行一些读写操作，完成了读写操作就要关闭相应的socket描述字，好比操作完打开的文件要调用fclose关闭打开的文件。
```c
 #include <unistd.h>
 int close(int fd);
```
- close一个TCP socket的缺省行为时把该socket标记为以关闭，然后立即返回到调用进程。该描述字不能再由调用进程使用，也就是说不能再作为read或write的第一个参数。
- 注意：close操作只是使相应socket描述字的引用计数-1，只有当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。
  
### 7. sockaddr和sockaddr_in结构体
- struct sockaddr和struct sockaddr_in这两个结构体用来处理网络通信的地址
- sockaddr在头文件**include<sys/socket.h>**中定义，sockaddr的缺陷是：sa_data把目标地址和端口信息混在一起了，如下：
```c++
struct sockaddr {  
     sa_family_t sin_family;//地址族
　　  char sa_data[14]; //14字节，包含套接字中的目标地址和端口信息               
　　 };
```

- sockaddr_in在头文件#include<netinet/in.h>或#include <arpa/inet.h>中定义，该结构体解决了sockaddr的缺陷，把port和addr 分开储存在两个变量中，如下：
```c++
struct sockaddr_in {
    sa_family_t   sin_family;  //地址簇
    uint16_t      sin_port;    //16位TCP/UDP端口号
    struct in_addr  sin_addr;   //32位IP地址
    char            sin_zero[8];  //不使用
};

struct in_addr
{
    In_addr   s_addr;  //32位ipv4地址
};
```
- sin_port和sin_addr都必须是网络字节序（NBO），一般可视化的数字都是主机字节序（HBO）
-  二者长度一样，都是16个字节，即占用的内存大小是一致的，因此可以互相转化。二者是并列结构，指向sockaddr_in结构的指针也可以指向sockaddr。
- sockaddr常用于bind、connect、recvfrom、sendto等函数的参数，指明地址信息，是一种通用的套接字地址。
- sockaddr_in 是internet环境下套接字的地址形式。所以在网络编程中我们会对sockaddr_in结构体进行操作，使用sockaddr_in来建立所需的信息，最后使用类型转化就可以了。一般先把sockaddr_in变量赋值后，强制类型转换后传入用sockaddr做参数的函数：sockaddr_in用于socket定义和赋值；sockaddr用于函数参数。

htons()作用是将端口号由主机字节序转换为网络字节序的整数值。(host to net)

inet_addr()作用是将一个IP字符串转化为一个网络字节序的整数值，用于sockaddr_in.sin_addr.s_addr。

inet_ntoa()作用是将一个sin_addr结构体输出成IP字符串(network to ascii)

### 8. 网络字节序和主机字节序
- 网络字节序与主机字节序
- 主机字节序就是我们平常说的大端和小端模式：不同的CPU有不同的字节序类型，这些字节序是指整数在内存中保存的顺序，这个叫做主机序。引用标准的Big-Endian和Little-Endian的定义如下：
　　a) Little-Endian就是低位字节排放在内存的低地址端，高位字节排放在内存的高地址端。
　　b) Big-Endian就是高位字节排放在内存的低地址端，低位字节排放在内存的高地址端。
- 网络字节序：4个字节的32 bit值以下面的次序传输：首先是0～7bit，其次8～15bit，然后16～23bit，最后是24~31bit。这种传输次序称作大端字节序。由于TCP/IP首部中所有的二进制整数在网络中传输时都要求以这种次序，因此它又称作网络字节序。字节序，顾名思义字节的顺序，就是大于一个字节类型的数据在内存中的存放顺序，一个字节的数据没有顺序的问题了。
- 所以：在将一个地址绑定到socket的时候，请先将主机字节序转换成为网络字节序，而不要假定主机字节序跟网络字节序一样使用的是Big-Endian。由于这个问题曾引发过血案！公司项目代码中由于存在这个问题，导致了很多莫名其妙的问题，所以请谨记对主机字节序不要做任何假定，务必将其转化为网络字节序再赋给socket。

0号端口是保留端口，无论是TCP还是UDP网络通信，0号端口都是不能使用的。
如果在bind绑定的时候指定端口0，意味着由系统随机选择一个可用端口来绑定。

### stat函数和结构体
> 作用：获取文件信息
> 头文件：include <sys/types.h> #include <sys/stat.h> #include <unistd.h>
> 函数原型：int stat(const char *path, struct stat *buf)
> ​ 参数：文件路径（名），struct stat 类型的结构体，​ 返回值：成功返回0，失败返回-1；
```cpp
struct stat
{
    dev_t     st_dev;     /* ID of device containing file */文件使用的设备号
    ino_t     st_ino;     /* inode number */    索引节点号 
    mode_t    st_mode;    /* protection */  文件对应的模式，文件，目录等
    nlink_t   st_nlink;   /* number of hard links */    文件的硬连接数  
    uid_t     st_uid;     /* user ID of owner */    所有者用户识别号
    gid_t     st_gid;     /* group ID of owner */   组识别号  
    dev_t     st_rdev;    /* device ID (if special file) */ 设备文件的设备号
    off_t     st_size;    /* total size, in bytes */ 以字节为单位的文件容量   
    blksize_t st_blksize; /* blocksize for file system I/O */ 包含该文件的磁盘块的大小   
    blkcnt_t  st_blocks;  /* number of 512B blocks allocated */ 该文件所占的磁盘块  
    time_t    st_atime;   /* time of last access */ 最后一次访问该文件的时间   
    time_t    st_mtime;   /* time of last modification */ /最后一次修改该文件的时间   
    time_t    st_ctime;   /* time of last status change */ 最后一次改变该文件状态的时间   
};
```
stat结构体中的st_mode 则定义了下列数种情况：
```cpp
S_IFMT   0170000    文件类型的位遮罩
    S_IFSOCK 0140000    套接字
    S_IFLNK 0120000     符号连接
    S_IFREG 0100000     一般文件
    S_IFBLK 0060000     区块装置
    S_IFDIR 0040000     目录
    S_IFCHR 0020000     字符装置
    S_IFIFO 0010000     先进先出
​
    S_ISUID 04000     文件的(set user-id on execution)位
    S_ISGID 02000     文件的(set group-id on execution)位
    S_ISVTX 01000     文件的sticky位
​
    S_IRUSR(S_IREAD) 00400     文件所有者具可读取权限
    S_IWUSR(S_IWRITE)00200     文件所有者具可写入权限
    S_IXUSR(S_IEXEC) 00100     文件所有者具可执行权限
​
    S_IRGRP 00040             用户组具可读取权限
    S_IWGRP 00020             用户组具可写入权限
    S_IXGRP 00010             用户组具可执行权限
​
    S_IROTH 00004             其他用户具可读取权限
    S_IWOTH 00002             其他用户具可写入权限
    S_IXOTH 00001             其他用户具可执行权限
​
    上述的文件类型在POSIX中定义了检查这些类型的宏定义：
    S_ISLNK (st_mode)    判断是否为符号连接
    S_ISREG (st_mode)    是否为一般文件
    S_ISDIR (st_mode)    是否为目录
    S_ISCHR (st_mode)    是否为字符装置文件
    S_ISBLK (s3e)        是否为先进先出
    S_ISSOCK (st_mode)   是否为socket
    若一目录具有sticky位(S_ISVTX)，则表示在此目录下的文件只能被该文件所有者、此目录所有者或root来删除或改名，在linux中，最典型的就是这个/tmp目录啦。
```
st_mode 的结构
st_mode 主要包含了 3 部分信息：
- 15-12 位保存文件类型
- 11-9 位保存执行文件时设置的信息
- 8-0 位保存文件访问权限

### get和post
- get 是从服务器上获取数据，post是向服务器传送数据
- get请求获取request-url所标识资源
- post请求服务器接收附在后面的数据

### pipe（）管道
创建管道
```cpp
int pipe(int pipefd[2]); //成功：0；失败：-1，设置errno
```
函数调用成功返回r/w两个文件描述符。无需open，但需手动close。规定：fd[0] → r； fd[1] → w，就像0对应标准输入，1对应标准输出一样。向管道文件读写数据其实是在读写内核缓冲区。

管道创建成功以后，创建该管道的进程（父进程）同时掌握着管道的读端和写端。

### fork()函数
fork（）函数通过系统调用创建一个与原来进程几乎完全相同的进程，也就是两个进程可以做完全相同的事，但如果初始参数或者传入的变量不同，两个进程也可以做不同的事。

### dup()和dup2()函数
```cpp
#include <unistd.h>
int dup(int oldfd);
```
dup用来复制参数oldfd所指的文件描述符。当复制成功是，返回最小的尚未被使用过的文件描述符，若有错误则返回-1.错误代码存入errno中,返回的新文件描述符和参数oldfd指向同一个文件，这两个描述符共享同一个数据结构，共享所有的锁定，读写指针和各项全现或标志位。

dup2函数
头文件及其定义：
```cpp
 #include <unistd.h>
 int dup2(int oldfd, int newfd);
```
dup2与dup区别是dup2可以用参数newfd指定新文件描述符的数值。若参数newfd已经被程序使用，则系统就会将newfd所指的文件关闭，若newfd等于oldfd，则返回newfd,而不关闭newfd所指的文件。dup2所复制的文件描述符与原来的文件描述符共享各种文件状态。共享所有的锁定，读写位置和各项权限或flags等.
返回值：
若dup2调用成功则返回新的文件描述符，出错则返回-1

### putenv函数
头文件：
```cpp
#include4<stdlib.h>
```
定义函数：
```cpp
int putenv(const char * string);
```
函数说明：putenv()用来改变或增加环境变量的内容. 参数string 的格式为name＝value, 如果该环境变量原先存在, 则变量内容会依参数string 改变, 否则此参数内容会成为新的环境变量.

返回值：执行成功则返回0, 有错误发生则返回-1.

错误代码：ENOMEM 内存不足, 无法配置新的环境变量空间.
