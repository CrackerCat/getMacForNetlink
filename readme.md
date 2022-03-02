## Android netlink&svc 获取 Mac方法深入分析

### 前文:

随着市面上获取指纹的方式越来越多，获取的方式也千奇百怪。

比如最开始的system_property_get , system_property_find ,  system_property_read 直接调用native底层获取。

到后来的进阶包括svc读取boot_id 文件，内存反射mValues 的 map 变量获取android id 

（需要过掉反射限制） , 都是很不错的方法指纹获取方法。

今天主要介绍的是通过内核通讯的方式获取设备网卡mac指纹，主要通过netlink的方式和内核通讯去获取mac网卡地址 。

这种方式可以直接绕过android的权限。

在不给app授权的时候也可以直接获取到网卡信息。因为很难进行mock，所以很多大厂app也都是采用这种办法去获取。

我在原有的基础上继续完善了一下逻辑，在接收消息的时候通过内联svc的方式处理接收收到的数据包，大大增加了数据的安全性。

也防止有人通过inlinehook 直接hook recv ,recvform,recvmsg 直接在收到数据包的时候被拦截和替换掉。

理论上这种方式可以过掉99%以上的改机软件。



### netlink简介：

- Netlink是linux提供的用于内核和用户态进程之间的通信方式。
  - 但是注意虽然Netlink主要用于**用户空间**和**内核空间**的通信，但是也能用于**用户空间的两个进程通信**。
  - 只是进程间通信有其他很多方式，一般不用Netlink。除非需要用到Netlink的广播特性时。
  - NetLink机制是一种特殊的socket，**它是Linux特有的**，由于传送的消息是暂存在socket接收缓存中，并不为接受者立即处理，所以netlink是一种异步通信机制。系统调用和ioctl是同步通信机制。



一般来说用户空间和内核空间的通信方式有三种：

> proc
>
> ioctl
>
> Netlink

而前两种都是单向的，但是Netlink可以实现**双工通信**。

Netlink协议基于BSD socket和AF_NETLINK地址簇(address family)。



使用32位的端口号寻址(以前称为PID)，每个Netlink协议(或称作总线，man手册中则称之为netlink family)，通常与一个或者一组内核服务/组件相关联，如NETLINK_ROUTE用于获取和设置路由与链路信息、NETLINK_KOBJECT_UEVENT用于内核向用户空间的udev进程发送通知等。

### netlink特点：

> 1,支持全双工、异步通信
>
> 2,用户空间可以使用标准的BSD socket接口(但netlink并没有屏蔽掉协议包的构造与解析过程，推荐使用libnl等第三方库)
>
> 3,在内核空间使用专用的内核API接口
>
> 4,支持多播(因此支持“总线”式通信，可实现消息订阅)
>
> 5,在内核端可用于进程上下文与中断上下文

### netlink优点：

> - netlink使用简单，只需要在include/linux/netlink.h中增加一个新类型的 netlink 协议定义即可,(如 #define NETLINK_TEST 20 然后，内核和用户态应用就可以立即通过 socket API 使用该 netlink 协议类型进行数据交换);
> - netlink是一种异步通信机制，在内核与用户态应用之间传递的消息保存在socket缓存队列中，发送消息只是把消息保存在接收者的socket的接收队列，而不需要等待接收者收到消息；
> - 使用 netlink 的内核部分可以采用模块的方式实现，使用 netlink 的应用部分和内核部分没有编译时依赖;
> - netlink 支持多播，内核模块或应用可以把消息多播给一个netlink组，属于该neilink 组的任何内核模块或应用都能接收到该消息，内核事件向用户态的通知机制就使用了这一特性；
> - 内核可以使用 netlink 首先发起会话;



## 如何通过netlink获取网卡信息？

### android 是如何通过netlink获取网卡地址的？

不管是ip命令行还是Java的network接口，最终都是调用到ifaddrs.cpp -> getifaddrs

### getifaddrs方法介绍

源码摘抄自：

http://aospxref.com/android-10.0.0_r47/xref/bionic/libc/bionic/ifaddrs.cpp#236

```
//传入对应的结构体指针
int getifaddrs(ifaddrs** out) {
  // We construct the result directly into `out`, so terminate the list.
  *out = nullptr;

  // Open the netlink socket and ask for all the links and addresses.
  NetlinkConnection nc;
  //判断get addresses 和 get link是否打开成功，返回成功则返回0
  bool okay = nc.SendRequest(RTM_GETLINK) && nc.ReadResponses(__getifaddrs_callback, out) &&
              nc.SendRequest(RTM_GETADDR) && nc.ReadResponses(__getifaddrs_callback, out);
  if (!okay) {
    out = nullptr;
    freeifaddrs(*out);
    // Ensure that callers crash if they forget to check for success.
    *out = nullptr;
    return -1;
  }

  return 0;
}
```

NetlinkConnection这个结构体是一个netlink的封装类

重点看一下ReadResponses的实现过程



代码摘抄自：

http://aospxref.com/android-10.0.0_r47/xref/bionic/libc/bionic/bionic_netlink.cpp



```

/**
 * @param type  发送参数的类型,具体获取的内容参考
 * @see rtnetlink.h
 * @return
 */
bool NetlinkConnection::SendRequest(int type) {
  // Rather than force all callers to check for the unlikely event of being
  // unable to allocate 8KiB, check here.
  // NetlinkConnection构造方法 的时候生成的8kb的data内存
  if (data_ == nullptr) return false;

  // Did we open a netlink socket yet?
  if (fd_ == -1) {
    //尝试建立socket netlink 链接
    fd_ = socket(PF_NETLINK, SOCK_RAW | SOCK_CLOEXEC, NETLINK_ROUTE);
    if (fd_ == -1) return false;
  }

  // Construct and send the message.
  // 构造要发送的消息
  struct NetlinkMessage {
    nlmsghdr hdr;
    rtgenmsg msg;
  } request;

  memset(&request, 0, sizeof(request));
  request.hdr.nlmsg_flags = NLM_F_DUMP | NLM_F_REQUEST;
  request.hdr.nlmsg_type = type;
  request.hdr.nlmsg_len = sizeof(request);
  // All families
  request.msg.rtgen_family = AF_UNSPEC;
  //使用socket数据发送
  return (TEMP_FAILURE_RETRY(send(fd_, &request, sizeof(request), 0)) == sizeof(request));
}
```



```
/*
 * 获取socket的返回结果
 */
bool NetlinkConnection::ReadResponses(void callback(void*, nlmsghdr*), void* context) {
  // Read through all the responses, handing interesting ones to the callback.
  ssize_t bytes_read;
  while ((bytes_read = TEMP_FAILURE_RETRY(recv(fd_, data_, size_, 0))) > 0) {
    //将拿到的data数据进行赋值
    auto* hdr = reinterpret_cast<nlmsghdr*>(data_);

    for (; NLMSG_OK(hdr, static_cast<size_t>(bytes_read)); hdr = NLMSG_NEXT(hdr, bytes_read)) {
      //判断是否读取结束,否则读取callback
      if (hdr->nlmsg_type == NLMSG_DONE) return true;
      if (hdr->nlmsg_type == NLMSG_ERROR) {
        auto* err = reinterpret_cast<nlmsgerr*>(NLMSG_DATA(hdr));
        errno = (hdr->nlmsg_len >= NLMSG_LENGTH(sizeof(nlmsgerr))) ? -err->error : EIO;
        return false;
      }
      //处理具体逻辑
      callback(context, hdr);
    }
  }

  // We only get here if recv fails before we see a NLMSG_DONE.
  return false;
}
```



### 使用流程：

通过遍历拿到我们需要的内容，输出即可。

```
int listmacaddrs(void) {
    struct ifaddrs *ifap, *ifaptr;

    if (myGetifaddrs(&ifap) == 0) {
        for (ifaptr = ifap; ifaptr != NULL; ifaptr = (ifaptr)->ifa_next) {
            char macp[INET6_ADDRSTRLEN];
            if(ifaptr->ifa_addr!= nullptr) {
                if (((ifaptr)->ifa_addr)->sa_family == AF_PACKET) {
                    auto *sockadd = (struct sockaddr_ll *) (ifaptr->ifa_addr);
                    int i;
                    int len = 0;
                    for (i = 0; i < 6; i++) {
                        len += sprintf(macp + len, "%02X%s", sockadd->sll_addr[i],( i < 5 ? ":" : ""));
                    }
                    //LOGE("%s  %s  ",(ifaptr)->ifa_name,macp)
                    if(strcmp(ifaptr->ifa_name,"wlan0")== 0){
                        LOGE("%s  %s  ",(ifaptr)->ifa_name,macp)
                        freeifaddrs(ifap);
                        return 1;
                    }
                }
            }

        }
        freeifaddrs(ifap);
        return 0;
    } else {
        return 0;
    }
}
```



### SVC内联安全封装：

在接受消息的时候android源码是采用recv去接受的消息

通过循环的方式去判断结束位置。

```
/*
 * 获取socket的返回结果
 */
bool NetlinkConnection::ReadResponses(void callback(void*, nlmsghdr*), void* out) {
  // Read through all the responses, handing interesting ones to the callback.
  ssize_t bytes_read;
  
//  while ((bytes_read = TEMP_FAILURE_RETRY(recv(fd_, data_, size_, 0))) > 0) {
//  while ((bytes_read = TEMP_FAILURE_RETRY(recvfrom(fd_, data_, size_, 0 ,NULL,0))) > 0) {

while ((bytes_read = TEMP_FAILURE_RETRY(raw_syscall(__NR_recvfrom,fd_, data_, size_, 0, NULL,0))) > 0) {
    auto* hdr = reinterpret_cast<nlmsghdr*>(data_);
    for (; NLMSG_OK(hdr, static_cast<size_t>(bytes_read)); hdr = NLMSG_NEXT(hdr, bytes_read)) {
      
      if (hdr->nlmsg_type == NLMSG_DONE) return true;
      if (hdr->nlmsg_type == NLMSG_ERROR) {
        auto* err = reinterpret_cast<nlmsgerr*>(NLMSG_DATA(hdr));
        errno = (hdr->nlmsg_len >= NLMSG_LENGTH(sizeof(nlmsgerr))) ? -err->error : EIO;
        return false;
      }
    
      callback(out, hdr);
    }
  }

  // We only get here if recv fails before we see a NLMSG_DONE.
  return false;
}
```

但是recv这种函数很容易被hook，inlinehook recv ,recvfrom ,recvmsg 

在方法执行完毕以后直接就可以处理参数二的返回值。

在不直接使用系统提供的recv 以后，有两种方式可以选择。

- 方法1：

直接调用syscall函数，通过syscall函数进行切入到recv 。

这种方式可以更好的兼容32和64位，但是可能被直接hook syscall这个函数入口 。

因为和设备指纹相关的函数，是重点函数，侧重安全。所以重点采用方法2

将syscall 汇编代码嵌入到指定方法内部。

- 方法2：我们直接把recv换成svc内联汇编代码如下

   相当于自己实现syscall （代码摘抄自libc syscall）

  

使用的话也很简单，导入函数头就好。

```
extern "C" {
    __inline__ __attribute__((always_inline))  long raw_syscall(long __number, ...);
}
```



#### 32位：

```
    .text
    .global raw_syscall
    .type raw_syscall,%function

raw_syscall:
        MOV             R12, SP
        STMFD           SP!, {R4-R7}
        MOV             R7, R0
        MOV             R0, R1
        MOV             R1, R2
        MOV             R2, R3
        LDMIA           R12, {R3-R6}
        SVC             0
        LDMFD           SP!, {R4-R7}
        mov             pc, lr

```

#### 64位：

```
    .text
    .global raw_syscall
    .type raw_syscall,@function

raw_syscall:
        MOV             X8, X0
        MOV             X0, X1
        MOV             X1, X2
        MOV             X2, X3
        MOV             X3, X4
        MOV             X4, X5
        MOV             X5, X6
        SVC             0
        RET

```



将代码替换成如下：

```
while ((bytes_read = TEMP_FAILURE_RETRY(raw_syscall(__NR_recv,fd_, data_, size_, 0, NULL,0))) > 0) {
    auto* hdr = reinterpret_cast<nlmsghdr*>(data_);
    for (; NLMSG_OK(hdr, static_cast<size_t>(bytes_read)); hdr = NLMSG_NEXT(hdr, bytes_read)) {
      //判断是否读取结束,否则读取callback
      if (hdr->nlmsg_type == NLMSG_DONE) return true;
      if (hdr->nlmsg_type == NLMSG_ERROR) {
        auto* err = reinterpret_cast<nlmsgerr*>(NLMSG_DATA(hdr));
        errno = (hdr->nlmsg_len >= NLMSG_LENGTH(sizeof(nlmsgerr))) ? -err->error : EIO;
        return false;
      }
      //处理具体逻辑
      callback(out, hdr);
    }
  }

  // We only get here if recv fails before we see a NLMSG_DONE.
  return false;
}
```

很不幸，报错了，安卓8内核上使用了seccomop 过滤掉了svc 直接调用 recv 

```

2022-03-02 21:47:13.753 5867-5867/? A/DEBUG: Build fingerprint: 'Xiaomi/cmi/cmi:11/RKQ1.200826.002/21.11.3:user/release-keys'
2022-03-02 21:47:13.753 5867-5867/? A/DEBUG: Revision: '0'
2022-03-02 21:47:13.753 5867-5867/? A/DEBUG: ABI: 'arm'
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: Timestamp: 2022-03-02 21:47:13+0800
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: pid: 5773, tid: 5773, name: example.jnihook  >>> com.example.jnihook <<<
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: uid: 10019
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: signal 31 (SIGSYS), code 1 (SYS_SECCOMP), fault addr --------
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: Cause: seccomp prevented call to disallowed arm system call 291
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG:     r0  0000004c  r1  da3ea000  r2  00002000  r3  00000000
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG:     r4  00000000  r5  00000000  r6  00000000  r7  00000123
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG:     r8  00000000  r9  f1ff2e00  r10 ffeedb20  r11 f1ff2e00
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG:     ip  ffeed9f8  sp  ffeed9e8  lr  c4459b03  pc  c445a3a4
2022-03-02 21:47:13.756 5867-5867/? A/DEBUG: backtrace:
2022-03-02 21:47:13.757 5867-5867/? A/DEBUG:       #00 pc 0000c3a4  /data/app/~~O88Sqqnxjf7EjHid_THMIA==/com.example.jnihook-j2EVKCAjF3Cpu3p_RLym8A==/lib/arm/libhelloword.so (BuildId: 95d05421436486cc260cc32f813488b04b882b78)
....
```

报错的原因一句话 

**seccomp prevented call to disallowed arm system call 291**

#### secomp简介：

seccomp是Linux的一种安全机制，android 8.1以上使用了seccomp

主要功能是限制直接通过syscall去调用某些系统函数

seccomp的过滤模式有两种(strict&filter)

第一种strict只支持如下四种,如果一旦使用了其他的syscall 则会收到SIGKILL信号

> read()
>
> write()
>
> exit()
>
> rt_sigreturn



通过下面方式进行设置。

```
seccomp（SECCOMP_SET_MODE_STRICT）
prctl (PR_SET_SECCOMP, SECCOMP_MODE_STRICT)。
```

##### strict

```cpp
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <linux/seccomp.h>
#include <sys/prctl.h>

int main(int argc, char **argv)
{
int output = open(“output.txt”, O_WRONLY);
const char *val = “test”;
//通过prctl函数设置seccomp的模式为strict
printf(“Calling prctl() to set seccomp strict mode…\n”);

prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);

printf(“Writing to an already open file…\n”);
//尝试写入
write(output, val, strlen(val)+1);
printf(“Trying to open file for reading…\n”);

//设置完毕seccomp以后再次尝试open （因为设置了secomp的模式是strict，所以这行代码直接sign -9 信号）
int input = open(“output.txt”, O_RDONLY);
printf(“You will not see this message — the process will be killed first\n”);
}
```

## filter（BPF）

Seccomp-bpf

bpf是一种过滤模式，只有在linux高版本会存在该功能

当某进程调用了svc 首先会进入我们自己写的bpf规则

通过我们自己的写的规则，进行判断该函数是否被运行调用。



**常用的就是ptrace+seccomp去修改svc的参数内容&返回值结果。**



回到正文，不过还好，

在android底层 recv的实现是recvfom代码如下

```
__BIONIC_FORTIFY_INLINE
ssize_t recv(int socket, void* const buf __pass_object_size0, size_t len, int flags)
    __overloadable
    __clang_error_if(__bos_unevaluated_lt(__bos0(buf), len),
                     "'recv' called with size bigger than buffer") {
  return recvfrom(socket, buf, len, flags, NULL, 0);
}
```

我们将svc调用号切换到recvform

```
bool NetlinkConnection::ReadResponses(void callback(void*, nlmsghdr*), void* out) {
  // Read through all the responses, handing interesting ones to the callback.
  ssize_t bytes_read;

while ((bytes_read = TEMP_FAILURE_RETRY(raw_syscall(__NR_recvfrom,fd_, data_, size_, 0, NULL,0))) > 0) {
    auto* hdr = reinterpret_cast<nlmsghdr*>(data_);
    for (; NLMSG_OK(hdr, static_cast<size_t>(bytes_read)); hdr = NLMSG_NEXT(hdr, bytes_read)) {
      
      if (hdr->nlmsg_type == NLMSG_DONE) return true;
      if (hdr->nlmsg_type == NLMSG_ERROR) {
        auto* err = reinterpret_cast<nlmsgerr*>(NLMSG_DATA(hdr));
        errno = (hdr->nlmsg_len >= NLMSG_LENGTH(sizeof(nlmsgerr))) ? -err->error : EIO;
        return false;
      }
      //处理具体逻辑
      callback(out, hdr);
    }
  }

  // We only get here if recv fails before we see a NLMSG_DONE.
  return false;
}
```

程序完美运行起来，网卡获取成功。

```
2022-03-02 22:05:53.790 11145-11145/com.example.jnihook E/Netlink: wlan0  A4:4B:D5:0B:51:57  
```







参考文章：
https://blog.csdn.net/zhizhengguan/article/details/120448337