# sendfile(2) 系统调用详解 (结合 C++ 与 Linux 实践)

本文档基于 Linux man-pages 6.13 (2024-07-23) 对 `sendfile` 系统调用进行详细的中文解释。内容将结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 的实践原则（特别是资源管理和性能）、C++ Reference [https://en.cppreference.com/w/c/memory] 上下文、Jason's Blog 关于 Linux 零拷贝的文章 [https://jasonblog.github.io/note/linux_system/sendfilelinuxzhong_768422_ling_kao_8c9d22.html] 的深入分析，以及 Linux [https://www.kernel.org/doc/html/latest/translations/zh_CN/mm/index.html] 特有的行为进行探讨。`sendfile` 是 Linux 提供的一种在两个文件描述符之间高效传输数据的机制，尤其以其“零拷贝”特性闻名。

**核心概念 (零拷贝 Zero-Copy):** `sendfile` 的主要优势在于它可以在内核空间内直接传输数据，避免了传统 `read()`/`write()` 模式下数据在内核缓冲区和用户空间缓冲区之间的多次复制。理想情况下（需要文件系统和硬件支持，如网卡的聚合操作和校验和计算能力），数据可以直接从源文件描述符（通常是文件）的页缓存 (page cache) 传输到目标文件描述符（通常是套接字）的缓冲区，显著减少 CPU 占用和内存带宽消耗，提高数据传输效率。即使硬件不支持完整的零拷贝，内核内的直接拷贝也通常比用户空间中转更高效 [https://jasonblog.github.io/note/linux_system/sendfilelinuxzhong_768422_ling_kao_8c9d22.html]。

---

## 1. 名称 (NAME)
`sendfile` - 在文件描述符之间传输数据

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <sys/sendfile.h> // 需要包含此头文件

ssize_t sendfile(int out_fd, int in_fd, off_t *_Nullable offset,
                 size_t count);
```
*   `out_fd`: 数据写入的目标文件描述符。
*   `in_fd`: 数据读取的源文件描述符。
*   `offset`: 指向一个偏移量变量的指针，用于指定从 `in_fd` 的哪个位置开始读取。如果非 `NULL`，`sendfile` 会从该偏移量开始读取，并在返回时更新该变量为最后读取字节之后的偏移量；`in_fd` 本身的偏移量不受影响。如果为 `NULL`，则从 `in_fd` 当前的文件偏移量开始读取，并更新 `in_fd` 的偏移量。
*   `count`: 要传输的字节数。
*   返回值 `ssize_t`: 成功时返回写入 `out_fd` 的字节数，失败时返回 -1。

## 4. 描述 (DESCRIPTION)

`sendfile()` 在两个文件描述符之间复制数据。由于这种复制在**内核内部**完成，`sendfile()` 比 `read(2)` 和 `write(2)` 的组合效率更高，后者需要在内核空间和用户空间之间传输数据。

*   `in_fd` 必须是一个为**读取**而打开的文件描述符。它必须对应一个支持**类 `mmap(2)` 操作**的文件（通常是普通文件，不能是套接字，但在 Linux 5.12+ 中若 `out_fd` 是管道则有例外，此时行为类似 `splice(2)`）。
*   `out_fd` 必须是一个为**写入**而打开的文件描述符。
    *   在 Linux 2.6.33 之前，`out_fd` **必须**引用一个**套接字 (socket)**。
    *   从 Linux 2.6.33 开始，`out_fd` 可以是**任何文件**。如果它是一个可查找 (seekable) 的文件，`sendfile()` 会相应地改变其文件偏移量。

**`offset` 参数详解:**
*   当 `offset` **不为 NULL** 时：
    *   `sendfile` 从 `in_fd` 的 `*offset` 指定的位置开始读取数据。
    *   `sendfile` 返回时，`*offset` 会被更新为指向最后读取字节之后的位置。
    *   `in_fd` 的当前文件偏移量**保持不变**。
*   当 `offset` **为 NULL** 时：
    *   `sendfile` 从 `in_fd` 的**当前文件偏移量**开始读取数据。
    *   `in_fd` 的当前文件偏移量会根据读取的字节数**被更新**。

`count` 参数指定了尝试在两个文件描述符之间复制的字节数。

## 5. 返回值 (RETURN VALUE)
*   **成功时:** 返回实际写入 `out_fd` 的**字节数**。**注意：** 一次成功的 `sendfile()` 调用可能写入**少于** `count` 请求的字节数。调用者必须准备好在有未发送字节时**重试**该调用。
*   **失败时:** 返回 **-1**，并设置 `errno` 来指示错误。

## 6. 错误 (ERRORS)
常见的 `errno` 值包括：
*   `EAGAIN`: `out_fd` 设置了 `O_NONBLOCK` 且写入操作会阻塞。
*   `EBADF`: `in_fd` 未以读模式打开，或 `out_fd` 未以写模式打开。
*   `EFAULT`: 地址错误（例如 `offset` 指针无效）。
*   `EINVAL`: 描述符无效或被锁定；`in_fd` 不支持类 `mmap(2)` 操作；`count` 为负数；`out_fd` 设置了 `O_APPEND` 标志（当前 `sendfile` 不支持）。
*   `EIO`: 从 `in_fd` 读取时发生未指定的 I/O 错误。
*   `ENOMEM`: 读取 `in_fd` 时内存不足。
*   `EOVERFLOW`: `count` 太大，操作会导致超出输入或输出文件的最大大小。
*   `ESPIPE`: `offset` 非 `NULL`，但 `in_fd` 不可查找（例如管道或套接字）。

## 7. 版本 (VERSIONS)
其他 UNIX 系统（如 Solaris, HP-UX）也实现了 `sendfile()`，但其**语义和原型有很大不同** [https://jasonblog.github.io/note/linux_system/sendfilelinuxzhong_768422_ling_kao_8c9d22.html]。例如：
*   Linux 的 `sendfile` 可用于任意两个符合要求的文件描述符之间（尤其新版本），而其他系统可能仅限于文件到套接字的传输。
*   Linux 的 `sendfile` 不直接支持向量化传输（即在传输文件数据的同时发送额外的头数据），而其他系统可能通过额外参数支持。
因此，`sendfile()` **不应**用于需要跨平台的可移植程序中。

## 8. 标准 (STANDARDS)
无。这是一个 Linux 特有的系统调用。

## 9. 历史 (HISTORY)
*   Linux 2.2 引入，glibc 2.1 支持。
*   关于 `out_fd` 可以是普通文件的支持在 2.6.x 内核系列中曾被移除，后在 Linux 2.6.33 中恢复。
*   最初的 Linux `sendfile()` 不支持大文件偏移量。Linux 2.4 增加了 `sendfile64()`，使用更宽的类型表示 `offset`。glibc 的 `sendfile()` 包装函数透明地处理了这些内核差异。历史上的 2GB 文件传输限制问题因此得到解决 [https://jasonblog.github.io/note/linux_system/sendfilelinuxzhong_768422_ling_kao_8c9d22.html]。

## 10. 注意事项 (NOTES)
*   **最大传输限制:** `sendfile()` 一次最多传输 `0x7ffff000` (2,147,479,552) 字节，并返回实际传输的字节数。这在 32 位和 64 位系统上都适用。
*   **TCP 优化 (`TCP_CORK`):** 如果使用 `sendfile()` 向 TCP 套接字发送文件，并且需要在文件内容前发送一些头数据（例如 HTTP 响应头），使用 `tcp(7)` 中描述的 `TCP_CORK` 选项会很有用。它可以将头数据和 `sendfile()` 发送的数据聚合到更少的 TCP 包中，调整性能。这在某种程度上弥补了 Linux `sendfile` 缺少向量化传输的问题 [https://jasonblog.github.io/note/linux_system/sendfilelinuxzhong_768422_ling_kao_8c9d22.html]。
*   **错误回退:** 应用程序可能希望在 `sendfile()` 因 `EINVAL` 或 `ENOSYS` 失败时，回退到使用 `read(2)`/`write(2)`。
*   **零拷贝数据一致性:** 如果 `out_fd` 指向支持零拷贝的套接字或管道，调用者**必须**确保在 `out_fd` 的另一端（读取者）完全消耗掉已传输的数据之前，`in_fd` 引用的文件的那部分内容**保持不变**。否则可能导致数据损坏。
*   **与 `splice(2)` 的关系:** Linux 特有的 `splice(2)` 调用支持在任意两个文件描述符之间传输数据，只要其中一个（或两个）是管道。

**C++ Core Guidelines 视角:**
*   **资源管理 (RAII):** `in_fd`, `out_fd` 以及网络编程中接受的客户端连接描述符都应使用 RAII 技术管理（例如，通过自定义类或库提供的包装器），确保它们在不再需要时被正确关闭。
*   **错误处理:** 必须检查 `sendfile()` 的返回值。不仅要检查是否为 -1 (错误)，还要处理返回值小于 `count` 的情况（部分写入），通常需要在一个循环中调用 `sendfile` 直到所有数据传输完毕或发生不可恢复的错误。
*   **性能:** `sendfile` 是实现高性能文件传输（特别是网络服务器）的关键系统调用，应在适用的场景下优先考虑。
*   **抽象:** 由于 `sendfile` 的不可移植性，如果需要跨平台，应将其封装在抽象层之后，提供可移植的回退实现（如 `read`/`write`）。

## 11. C++ 示例 (简单文件服务器)

这个 C++ 示例演示了一个极其简化的 TCP 服务器，它接受一个客户端连接，然后使用 `sendfile` 将指定的文件发送给客户端。

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <fcntl.h>       // open, O_RDONLY
#include <unistd.h>      // close, sysconf, _SC_PAGE_SIZE
#include <sys/sendfile.h> // sendfile
#include <sys/stat.h>    // fstat, struct stat
#include <sys/socket.h>  // socket, bind, listen, accept, sockaddr_in
#include <netinet/in.h>  // sockaddr_in, htons
#include <arpa/inet.h>   // inet_addr (演示用，实际应用推荐 getaddrinfo)
#include <cstdio>        // perror
#include <cstdlib>       // exit, EXIT_FAILURE, EXIT_SUCCESS
#include <cstring>       // strerror
#include <stdexcept>     // std::runtime_error
#include <memory>        // std::unique_ptr (非用于fd，用于演示RAII概念)


// 简单的 RAII 文件/Socket 描述符包装器 (基础版)
class Descriptor {
    int fd_ = -1;
public:
    Descriptor() = default;
    explicit Descriptor(int fd) : fd_(fd) {} // 包装现有fd，通常不拥有
    Descriptor(int domain, int type, int protocol) {
        fd_ = socket(domain, type, protocol);
        if (fd_ == -1) throw std::runtime_error("socket() failed");
    }
     Descriptor(const char* pathname, int flags) {
        fd_ = open(pathname, flags);
        if (fd_ == -1) throw std::runtime_error(std::string("open() failed for ") + pathname);
    }
    ~Descriptor() {
        if (fd_ != -1) {
            // std::cout << "Closing fd " << fd_ << std::endl; // Debug
            if (close(fd_) == -1) {
                 perror("~Descriptor: close failed"); // 不在析构抛异常
            }
        }
    }
    Descriptor(Descriptor&& other) noexcept : fd_(other.fd_) { other.fd_ = -1; }
    Descriptor& operator=(Descriptor&& other) noexcept {
        if (this != &other) {
            if (fd_ != -1) close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }
    Descriptor(const Descriptor&) = delete;
    Descriptor& operator=(const Descriptor&) = delete;
    int get() const { return fd_; }
    int release() { int tmp = fd_; fd_ = -1; return tmp; } // 释放所有权
};


int main(int argc, char *argv[]) {
    if (argc != 3) {
        std::cerr << "用法: " << argv[0] << " <要发送的文件> <监听端口>\n";
        return EXIT_FAILURE;
    }

    const char* filename = argv[1];
    int port = std::stoi(argv[2]);

    try {
        // 1. 创建监听 Socket (RAII)
        Descriptor listener_sock(AF_INET, SOCK_STREAM, 0);

        // 设置 SO_REUSEADDR 选项，允许快速重启服务器
        int opt = 1;
        if (setsockopt(listener_sock.get(), SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1) {
            perror("setsockopt(SO_REUSEADDR) failed");
            // 非致命错误，继续
        }


        // 2. 绑定地址
        sockaddr_in server_addr{};
        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = htonl(INADDR_ANY); // 监听所有接口
        server_addr.sin_port = htons(port);

        if (bind(listener_sock.get(), (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
            throw std::runtime_error("bind() failed");
        }

        // 3. 开始监听
        if (listen(listener_sock.get(), 1) == -1) { // 简化，只处理一个连接
            throw std::runtime_error("listen() failed");
        }

        std::cout << "服务器正在监听端口 " << port << "...\n";

        // 4. 接受客户端连接
        sockaddr_in client_addr{};
        socklen_t client_len = sizeof(client_addr);
        int client_fd_raw = accept(listener_sock.get(), (struct sockaddr*)&client_addr, &client_len);
        if (client_fd_raw == -1) {
             throw std::runtime_error("accept() failed");
        }
        Descriptor client_sock(client_fd_raw); // RAII 管理客户端 socket
        std::cout << "接受来自客户端的连接 fd=" << client_sock.get() << "\n";


        // 5. 打开要发送的文件 (RAII)
        Descriptor file_fd(filename, O_RDONLY);

        // 6. 获取文件大小
        struct stat sb;
        if (fstat(file_fd.get(), &sb) == -1) {
            throw std::runtime_error("fstat() failed");
        }
        off_t file_size = sb.st_size;
        off_t bytes_sent = 0;
        off_t current_offset = 0; // sendfile 将使用并更新这个偏移量

        std::cout << "开始发送文件 '" << filename << "' (大小: " << file_size << " 字节)...\n";

        // 7. 使用 sendfile 循环发送文件
        while (bytes_sent < file_size) {
            // 使用 current_offset 指针，sendfile 会更新它
            ssize_t sent_this_call = sendfile(client_sock.get(), file_fd.get(), &current_offset, file_size - bytes_sent);

            if (sent_this_call == -1) {
                // EAGAIN 表示暂时不可写，可以重试或等待，这里简化处理
                if (errno == EAGAIN) {
                     std::cerr << "sendfile 暂时阻塞，稍后重试...\n";
                     // 在实际应用中可能需要使用 select/poll/epoll
                     sleep(1); // 极简化的重试
                     continue;
                }
                throw std::runtime_error(std::string("sendfile() failed: ") + strerror(errno));
            }
            if (sent_this_call == 0) {
                // 文件大小可能在 fstat 后发生了变化？或者对方关闭连接？
                std::cerr << "sendfile 返回 0，可能文件大小改变或连接关闭。\n";
                break;
            }

            bytes_sent += sent_this_call;
            std::cout << "已发送: " << bytes_sent << " / " << file_size << " 字节\n";
            // current_offset 已被 sendfile 更新
        }

        std::cout << "文件发送完成。\n";

        // client_sock 和 file_fd 会在作用域结束时自动关闭
        // listener_sock 也会自动关闭

    } catch (const std::exception& e) {
        std::cerr << "服务器错误: " << e.what() << std::endl;
        // RAII 确保已打开的描述符被关闭
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}

```

**编译与运行:**

```bash
g++ your_sendfile_server.cpp -o sendfile_server
./sendfile_server my_large_file.dat 8080
```

然后在另一个终端使用 `netcat` 或其他客户端连接到服务器的 8080 端口，服务器就会将 `my_large_file.dat` 的内容发送给客户端。

这个 C++ 示例展示了 `sendfile` 在网络服务器中的典型用法，并结合了 RAII 来管理文件和套接字描述符，以及基本的错误处理和对部分写入情况的循环处理。

## 12. 参见 (SEE ALSO)
`copy_file_range(2)`, `mmap(2)`, `open(2)`, `socket(2)`, `splice(2)`, `tcp(7)`

---
