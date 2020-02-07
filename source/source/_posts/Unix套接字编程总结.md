---
title: Unix套接字编程总结
date: 2018-12-26 09:54:52
categories: 网络
tags: [Unix, 网络编程, 套接字]
---
## 常用POSIX数据类型
数据类型|说明|头文件
---|---|---
int8_t|带符号的8位整数|`<sys/types.h>`
uint8_t|无符号的8位整数|`<sys/types.h>`
int16_t|带符号的16位整数|`<sys/types.h>`
uint16_t|无符号的16位整数|`<sys/types.h>`
int32_t|带符号的32位整数|`<sys/types.h>`
uint32_t|无符号的32位整数|`<sys/types.h>`
sa_family_t|套接字地址结构的地址族，可以是任何无符号整型|`<sys/socket.h>`
socklen_t|套接字地址结构的长度，一般为uint32_t|`<sys/socket.h>`
in_addr_t|IPv4地址，一般为uint32_t|`<netinet/in.h>`
in_port_t|TCP或UDP端口，一般为uint16_t|`<netinet/in.h>`

## 套接字地址结构
### IPv4套接字结构
IPv4的地址结构是`struct sockaddr_in`，定义在`<netinet/in.h>`头文件中：

``` c
struct in_addr {
	in_addr_t s_addr;	/* 32bit IPv4地址 */
}

struct sockaddr_in{
	unit8_t sin_len;	/* 这个结构体长度，16字节 */
	sa_family_t sin_family;	/* AF_INET */
	in_port_t sin_port;	/* 16bit TCP 或 UDP 端口值 */
	struct in_addr sin_addr; /* 之所以用了结构体是因为以前对IP地址进行了A、B...分类处理 */
	char sin_zero[8];	/* 未使用 */
}
```
POSIX规范只需要结构中的3个字段：`sin_family`、`sin_addr`和`sin_port`。

良好的习惯是在使用`struct sockaddr_in`之前将整个结构置为0。

### 通用套接字结构
`<sys/socket.h>`头文件中定义通用套接字结构地址：

``` c
struct sockaddr {
	unit8_t sa_len;
	sa_family_t sa_family;	/* 套接字地址：AF_XXX */
	char sa_data[14];	/* 2字节端口号 + 4字节的IPv4地址 + 8字节的sin_zero数据 */
}
```
案例：

``` c
struct sockaddr_in serv;

/* fill in serv{} */

bind(sockfd, (struct sockaddr *)&serv, sizeof(serv));
```

### IPv6套接字结构
IPv6结构是`struct sockaddr_in6`，定义在`<netinet/in.h>`头文件中：

``` c
struct in6_addr{
	uint8_t s6_addr[16]; /* 128bit */
}

struct sockaddr_in6 {
	uint8_t sin6_len; /* 这个结构体长度，28字节 */
	sa_family_t sin6_family; /* AF_INET6 */
	in_port_t sin6_port;
	unit32_t sin6_flowinfo; /* flow information, undefined */
	struct in6_addr sin6_addr; 
	uint32_t sin6_scope_id;
}
```
### 新的通用套接字结构
`struct sockaddr_storage`定义在`<netinet/in.h>`中：

``` c
struct sockaddr_storage {
	unit8_t ss_len;
	sa_family_t ss_family;
	
	/* 用户透明 */
}
```
`struct sockaddr_storage`能够容纳系统所支持的任何套接字地址结构。

## 字节序
网络协议使用**大端字节序**传输数据。

``` c
#include <netinet/in.h>  /* 下面4个操作定义在 <netinet/in.h> 中 */

uint16_t htons(uint16_t host16bitvalue);
uint32_t htonl(uint32_t host32bitvalue);  /* 主机字节序转网络字节序 */

uint16_t ntohs(uint16_t net16bitvalue);
uint32_t ntohl(uint32_t net32bitvalue);  /* 网络字节序转主机字节序 */
```
h表示host，n表示net，s表示short，l表示long。

## 字节操作函数
有两组操作字节的函数，它们不对数据做解释，也不假设数据是以空字符结束的字符串。

名字以b开头的一组函数起源于4.2BSD：

``` c
#include <strings.h>  /* 注意：这儿是strings.h而不是string.h */

void bzero(void *dst, size_t nbytes);
void bcopy(const void *src, void *dst, size_t nbytes);
int bcmp(const void *ptr1, const void *ptr2, size_t nbytes); /* unsigned char 前提下比较 */
```

名字以mem开头的一组函数起源于ANSI C标准：

``` c
#include <string.h>  /* 注意：这儿是string.h而不是strings.h */

void *memset(void *dst, int c, size_t len); /* 把目标置为 c */
void *memcpy(void *dst, const void *src, size_t nbytes);
int memcmp(const void *ptr1, const void *ptr2, size_t nbytes); /* unsigned char 前提下比较 */
```

## 地址转换函数
### 只适用于IPv4
`inet_aton`, `inet_addr`和`inet_ntoa`只适用于IPv4进行点分十进制(例如："206.168.112.96")转换为网际地址和网际地址转换为点分十进制。函数定义在`<arpa/inet.h>`中。

``` c
#include <arpa/inet.h>

int inet_aton(const char *strptr, struct in_addr *addrptr); /* 若字符串有效返回1，否则为0 */

in_addr_t inet_addr(const char *strptr); /* **已废弃**，若字符串有效返回32位网络IPv4地址，否则为INADDR_NONE */

char *inet_ntoa(struct in_addr inaddr); /* 返回点分十进制字符串指针 */
```
### 通用地址转换
`inet_pton`和`inet_ntop`对于IPv4和IPv6都适用，p(presentation)表示"表达"，n表示"数值"。

``` c
#include <arpa/inet.h>

int inet_pton(int family, const char *strptr, void *addrptr);

const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t n);
```

## 套接字编程
### socket函数
``` c
#include <sys/socket.h>

int socket(int family, int type, int protocol); /* 若成功则为非负描述符，否则为-1 */
```
**family常值**：

family|说明
---|---
AF_INET|IPv4协议
AF_INET6|IPv6协议
AF_LOCAL|Unix域协议
AF_ROUTE|路由套接字
AF_KEY|密钥套接字

**type常值**：

type|说明
---|---
SOCK_STREAM|字节流套接字
SOCK_DGRAM|数据报套接字
SOCK_SEQPACKET|有序分组套接字
SOCK_RAW|原始套接字

### connect函数
``` c
#include <sys/socket.h>

/* 若成功返回0，若失败返回-1 */
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```

### bind函数
``` c
#include <sys/socket.h>

/* 若成功返回0，若失败返回-1 */
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

### listen函数
``` c
#include <sys/socket.h>

/* 若成功返回0，若失败返回-1 */
int listen(int sockfd, int backlog);
```

### accept函数
``` c
#include <sys/socket.h>

/* 若成功返回非负描述符，若出错返回-1 */
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```

### close函数
``` c
#include <unistd.h>

close(int sockfd); /* 若成功返回0，若出错返回-1 */
```
