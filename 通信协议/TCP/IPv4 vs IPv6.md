### 代码层面差异（C语言）

| 特性     | IPv4                             | IPv6                               |
| :----- | :------------------------------- | :--------------------------------- |
| 通用地址结构 | `struct sockaddr_in`             | `struct sockaddr_in6`              |
| 地址成员变量 | `sin_addr` (类型 `struct in_addr`) | `sin6_addr` (类型 `struct in6_addr`) |
| IP地址长度 | 32位 (4字节)                        | 128位 (16字节)                        |
| 端口成员变量 | `sin_port`                       | `sin6_port`                        |
| 协议族常量  | `AF_INET`                        | `AF_INET6`                         |

#### 1. 定义与初始化地址结构体

- **IPv4 写法：**

```c
# IPv4 写法
struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));

server_addr.sin_family = AF_INET; // 协议族
server_addr.sin_port = htons(8080); // 端口
// IPv4 通常用 inet_pton 转换点分十进制字符串
inet_pton(AF_INET, "192.168.1.1", &server_addr.sin_addr); 

# IPv6 写法
struct sockaddr_in6 server_addr;
memset(&server_addr, 0, sizeof(server_addr));

server_addr.sin6_family = AF_INET6; // 协议族
server_addr.sin6_port = htons(8080); // 端口
server_addr.sin6_flowinfo = 0;       // IPv6特有的流控信息，通常置0
// IPv6 转换冒号分隔的十六进制字符串
inet_pton(AF_INET6, "2001:db8::1", &server_addr.sin6_addr); 
```


#### 2. 创建套接字 (socket)

```C
# IPv4 写法
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

# IPv6 写法
int sockfd = socket(AF_INET6, SOCK_STREAM, 0);
```

#### 3. 强转与传参

在调用 `bind`, `connect`, `accept` 等系统函数时，都需要将上述具体的结构体强转为通用的 `struct sockaddr *`。
```c
# IPv4 写法
(struct sockaddr *)&server_addr

# IPv6 写法
(struct sockaddr *)&server_addr // 虽然名字一样，但指向的是 sockaddr_in6
```

##  两个重要的 API 细节

**1. 地址转换函数 (`inet_pton`)**  

以前常用的老旧函数 `inet_addr` 仅支持 IPv4。在现代代码中，强烈建议统一使用 **`inet_pton`**。它可以通过第一个参数自动判断是处理 IPv4 (`AF_INET`) 还是 IPv6 (`AF_INET6`) 的地址格式。

**2. IPv6 的双栈兼容问题**  

如果你在 Linux 上创建一个 `AF_INET6` 的 socket，默认情况下它可能既能接收 IPv6 连接，也能接收 IPv4 连接（这叫 IPv4-mapped IPv6）。  
如果你希望这个 socket **只处理纯 IPv6** 流量，需要在 bind 之前加一行代码设置选项：