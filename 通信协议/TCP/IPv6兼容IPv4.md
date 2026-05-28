在 C 语言代码层面，IPv4-mapped IPv6（IPv4 映射的 IPv6 地址）最直观的表现就是：**它本质上是一个 IPv6 结构体，但它的内部藏着一个完整的 IPv4 地址。**

具体可以从**内存布局**和**调试打印**两个角度来观察：

### 🧬 结构体与内存布局的表现

当你拿到一个 `struct sockaddr_in6` 结构体时，如果发现前10个字节是固定的“前缀”，后4个字节是你熟悉的 IPv4 地址，那就是 IPv4-mapped。

在代码中，它长这样：

```c
struct sockaddr_in6 mapped_addr;
// ... (省略初始化)

// 它的 sin6_addr 成员（128位/16字节）在内存中是这样分布的：
// | 80 bits (全为0) | 16 bits (全为1) | 32 bits (IPv4地址) |
// |   0x00...00     |     0xFFFF      |    你的IPv4       |
```

**如何写代码检测它？**  
你可以直接读取 `sin6_addr.s6_addr`（这是一个 16 字节的数组）来判断：

```c
// 检查是否是 IPv4-mapped 地址
if (mapped_addr.sin6_family == AF_INET6 && 
    IN6_IS_ADDR_V4MAPPED(&mapped_addr.sin6_addr)) {
    
    // 提取出藏在里面的 IPv4 地址
    struct in_addr extracted_ipv4;
    // IPv4 地址位于最后 4 个字节
    memcpy(&extracted_ipv4, &mapped_addr.sin6_addr.s6_addr[12], 4);
    
    char ip_str[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &extracted_ipv4, ip_str, sizeof(ip_str));
    printf("发现映射地址！实际 IPv4 是: %s\n", ip_str);
}
```

_(注：`IN6_IS_ADDR_V4MAPPED` 是 Linux/Unix 系统提供的宏，专门用来做这个判断)_

### 🖥️ 字符串打印的表现

如果你直接把这种地址用 `inet_ntop` 转换成字符串打印出来，它会呈现出一种**“套娃”**格式。

- **表现：** `::ffff:192.168.1.1`
- **解释：** `::ffff:` 是固定的头部，后面直接跟着点分十进制的 IPv4 地址。

**代码示例：**

```c
char str[INET6_ADDRSTRLEN];
// 假设 mapped_addr 里面存的是 192.168.1.1 的映射
inet_ntop(AF_INET6, &mapped_addr.sin6_addr, str, sizeof(str));
printf("%s\n"); 
// 输出结果将是： ::ffff:192.168.1.1
```

### 📌 总结它在 TCP 通信中的“行为表现”

在你的服务器代码中，如果开启了 IPv6 双栈（没有设置 `IPV6_V6ONLY`），当一个 IPv4 客户端连接进来时：

1. 你调用 `accept()` 拿到的客户端地址结构体类型是 `struct sockaddr_in6`（而不是 `sockaddr_in`）。
2. 但这个结构体的 `sin6_family` 依然是 `AF_INET6`。
3. 唯一的区别就是它的 IP 数据部分变成了上述的“映射格式”。

所以，如果你的代码只处理纯 IPv6，遇到这种地址可能会感到意外；但如果做了兼容处理，你就可以从这个结构体里轻松把原始的 IPv4 地址“抠”出来使用。