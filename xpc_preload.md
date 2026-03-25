# 模块介绍

**用途**：劫持 TCP 相关系统调用，走共享内存通信。

**使用场景**：透明劫持 TCP 通信，实现本地进程间高性能通信。

**核心功能**：
1. 加载原生系统调用
2. 定义劫持的系统调用（socket/bind/listen/connect/accept/send/recv/readv/writev/close 等）
3. 初始化全局 Socket 管理器 g_socketMgr

---

# 关键功能

## 1. SocketInit - 系统调用加载

从 libc 库中动态加载原生系统调用，并重命名为 `sys_` 前缀的函数，例如：
`sys_socket`、`sys_bind`、`sys_listen`、`sys_connect`、`sys_accept`、`sys_send`、`sys_recv`、`sys_readv`、`sys_writev`、`sys_close`

## 2. IsSupportedSocket - 支持判断

检查是否满足劫持条件：

| 参数       | 条件                  |
| -------- | ------------------- |
| domain   | `AF_INET`           |
| type     | `SOCK_STREAM`       |
| protocol | `0` 或 `IPPROTO_TCP` |

**返回值**：满足返回 true，不满足返回 false

## 3. socket - 创建套接字

```
1. 调用 IsSupportedSocket() 判断是否支持
2. 不支持 → 回退到 sys_socket() 直接返回
3. 支持 →
   a. 调用 sys_socket() 创建套接字 fd
   b. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
   c. 不在范围内 → 返回 fd
   d. 在范围内 → 为 fd 创建 Socket 结构体，注册到 g_socketMgr
   e. 返回 fd
```

**返回值**：成功返回非负 fd，失败返回 -1

## 4. bind - 绑定地址

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_bind() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已绑定
   b. 已绑定 → 返回 -1（禁止重复绑定）
   c. 未绑定 → 调用 XPC 模块为 fd 绑定地址
```

**返回值**：成功返回 0，失败返回 -1

## 5. listen - 监听连接

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_listen() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已绑定
   b. 已绑定 → 调用 XPC 模块完成监听准备（创建请求队列）
   c. 未绑定 → 返回 -1（错误）
```

**返回值**：成功返回 0，失败返回 -1

## 6. connect - 发起连接

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_connect() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已绑定
   b. 已绑定 → 调用 XPC 模块发送连接请求建立连接
   c. 未绑定 → 返回 -1（错误）
```

**返回值**：成功返回 0，失败返回 -1

## 7. accept - 接受连接

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_accept() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已监听
   b. 已监听 → 调用 XPC 模块获取连接请求，创建新套接字
   c. 未监听 → 返回 -1（错误）
```

**返回值**：成功返回新的文件描述符，失败返回 -1

## 8. send - 发送数据

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_send() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已连接
   b. 已连接 → 调用 XPC 模块发送数据
   c. 未连接 → 返回 -1（错误）
```

**返回值**：成功返回实际发送的字节数，失败返回 -1，连接关闭返回 0

## 9. recv - 接收数据

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_recv() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已连接
   b. 已连接 → 调用 XPC 模块接收数据到 buffer
   c. 未连接 → 返回 -1（错误）
```

**返回值**：成功返回实际接收的字节数，对端关闭返回 0，失败返回 -1

## 10. readv - 分散读取

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_readv() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已连接
   b. 已连接 → 调用 XPC 模块分散读取数据
   c. 未连接 → 返回 -1（错误）
```

**返回值**：成功返回实际读取的字节数，失败返回 -1

## 11. writev - 分散写入

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_writev() 直接返回
3. 在范围内 →
   a. 检查 Socket 状态是否为已连接
   b. 已连接 → 调用 XPC 模块分散写入数据
   c. 未连接 → 返回 -1（错误）
```

**返回值**：成功返回实际写入的字节数，失败返回 -1

## 12. close - 关闭套接字

```
1. 调用 IsFdOutOfCapacity() 检查 fd 是否在可支持范围内
2. 不在范围内 → 回退到 sys_close() 直接返回
3. 在范围内 →
   a. 从 g_socketMgr 中移除对应的 Socket 结构体
   b. 释放相关资源
   c. 调用 sys_close() 关闭原生 fd
```

**返回值**：成功返回 0，失败返回 -1

---

# 配置与接口

## 内部 API

| API 接口                                                       | 主要用途                     | 返回值说明                     |
| ------------------------------------------------------------ | ------------------------ | ------------------------- |
| `int SocketInit()`                                           | 加载原生系统调用，初始化 g_socketMgr | 成功返回 0，失败返回 -1            |
| `bool IsSupportedSocket(int domain, int type, int protocol)` | 判断是否支持劫持（IPv4 TCP）       | 支持返回 true，不支持返回 false     |
| `bool IsFdOutOfCapacity(int fd)`                             | 判断 fd 是否在可管理的套接字范围内      | 在范围内返回 false，不在范围内返回 true |

## 公开 API（劫持的系统调用）

| API 接口                                                                     | 主要用途          | 返回值说明                        |
| -------------------------------------------------------------------------- | ------------- | ---------------------------- |
| `int socket(int domain, int type, int protocol);`                          | 创建一个新的通信端点    | 成功返回非负整数（文件描述符），失败返回 -1      |
| `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`    | 将地址绑定到套接字     | 成功返回 0，失败返回 -1               |
| `int listen(int sockfd, int backlog);`                                     | 监听连接请求，建立请求队列 | 成功返回 0，失败返回 -1               |
| `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);` | 主动向服务器发起连接请求  | 成功返回 0，失败返回 -1               |
| `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`       | 接受连接请求，创建新套接字 | 成功返回新的文件描述符，失败返回 -1          |
| `ssize_t send(int sockfd, const void *buf, size_t len, int flags);`        | 向已连接套接字发送数据   | 成功返回实际发送字节数，失败返回 -1，连接关闭返回 0 |
| `ssize_t recv(int sockfd, void *buf, size_t len, int flags);`              | 从已连接套接字接收数据   | 成功返回实际接收字节数，0 表示对端关闭，失败返回 -1 |
| `ssize_t readv(int fd, const struct iovec *iov, int iovcnt);`             | 分散读取数据           | 成功返回实际读取字节数，失败返回 -1 |
| `ssize_t writev(int fd, const struct iovec *iov, int iovcnt);`            | 分散写入数据           | 成功返回实际写入字节数，失败返回 -1 |
| `int close(int fd);`                                                       | 关闭文件描述符       | 成功返回 0，失败返回 -1 |

---

# 异常处理

## 场景 1：fd 超出管理范围

**现象**：调用被劫持的函数，但 fd 不在 g_socketMgr 管理范围内

**处理**：直接回退到对应的 `sys_*` 原生系统调用

## 场景 2：Socket 状态不允许操作

**现象**：例如对未绑定的 socket 调用 listen，或对未连接的 socket 调用 send

**处理**：返回 -1

## 场景 3：重复绑定

**现象**：已绑定的 socket 再次调用 bind

**处理**：返回 -1

## 场景 4：非 TCP 套接字

**现象**：domain/type/protocol 不满足支持条件

**处理**：回退到原生系统调用

---

