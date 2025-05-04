# fork(2) 系统调用详解 (结合 C++ 实践原则)

本文档基于 Linux man-pages 6.13 (2025-01-14) 对 `fork` 系统调用进行详细的中文解释，并结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 中的并发、资源管理和错误处理原则，以及来自 GeeksforGeeks [https://www.geeksforgeeks.org/fork-system-call/] [https://www.geeksforgeeks.org/create-processes-with-fork-in-cpp/] 和 cplusplus.com [https://cplusplus.com/forum/beginner/280002/] 关于 `fork` 的讨论进行探讨。`fork` 是 Unix-like 系统中创建新进程的基本方法。

---

## 1. 名称 (NAME)
`fork` - 创建一个子进程

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <unistd.h> // POSIX 标准头文件

pid_t fork(void); // pid_t 通常是整型，用于表示进程 ID
```

## 4. 描述 (DESCRIPTION)

`fork()` 通过**复制调用进程**来创建一个新进程。新创建的进程被称为**子进程 (child process)**，而原来的进程被称为**父进程 (parent process)**。

父子进程运行在**独立的内存空间**中。在调用 `fork()` 的那一刻，两个进程的内存内容是相同的。但是，之后任何一方进行的内存写入、文件映射 (`mmap(2)`) 或取消映射 (`munmap(2)`) 操作**不会影响**另一方（得益于写时复制 Copy-on-Write 技术）。

子进程是父进程的一个**精确副本**，但有以下关键**区别**：

**POSIX.1 标准规定的区别:**
*   **进程 ID (PID):** 子进程拥有自己**唯一**的进程 ID，且不与任何现有进程组 ID 或会话 ID 相同。
*   **父进程 ID (PPID):** 子进程的 PPID 与父进程的 PID 相同。
*   **内存锁:** 子进程**不继承**父进程的内存锁 (`mlock(2)`, `mlockall(2)`)。
*   **资源使用和 CPU 时间:** 子进程的资源使用计数 (`getrusage(2)`) 和 CPU 时间计数器 (`times(2)`) 被**重置为零**。
*   **挂起信号:** 子进程的挂起信号集 (`sigpending(2)`) 初始为空。
*   **信号量调整:** 子进程**不继承**父进程的信号量调整 (`semop(2)`)。
*   **记录锁:** 子进程**不继承**父进程关联的记录锁 (`fcntl(2)`)。（但它会继承 `fcntl(2)` 的打开文件描述锁和 `flock(2)` 锁）。
*   **定时器:** 子进程**不继承**父进程的定时器 (`setitimer(2)`, `alarm(2)`, `timer_create(2)`)。
*   **异步 I/O:** 子进程**不继承**父进程未完成的异步 I/O 操作 (`aio_read(3)`, `aio_write(3)`) 或异步 I/O 上下文 (`io_setup(2)`)。

**Linux 特有的区别:**
*   **目录通知:** 子进程不继承 dnotify (`fcntl(2)` 中的 `F_NOTIFY`)。
*   **父进程死亡信号:** `prctl(2)` 的 `PR_SET_PDEATHSIG` 设置被重置，子进程在父进程终止时不会收到指定信号。
*   **定时器精度:** 子进程的默认定时器精度 (timer slack) 继承父进程当前的值。
*   **内存映射标志:** 被标记为 `MADV_DONTFORK` (`madvise(2)`) 的内存映射不会被继承。
*   **写时清零:** 被标记为 `MADV_WIPEONFORK` (`madvise(2)`) 的内存区域在子进程中会被清零（但该标志在子进程中仍然有效）。
*   **终止信号:** 子进程的终止信号总是 `SIGCHLD`。
*   **端口权限:** `ioperm(2)` 设置的端口访问权限不被继承。

**其他重要注意事项:**
*   **多线程程序中的 `fork()`:**
    *   子进程只包含**调用 `fork()` 的那个线程**的副本。整个虚拟地址空间被复制，包括互斥锁、条件变量和其他 pthreads 对象的状态。这可能导致子进程中的锁状态不一致（例如，锁可能被一个未在子进程中存在的线程持有）。`pthread_atfork(3)` 可以用来注册清理处理程序以应对这种情况。
    *   **C++ Core Guidelines (CP.con):** 明确指出 `fork` 在多线程程序中非常危险。**强烈建议**：在 `fork()` 之后，子进程在调用 `execve(2)` 之前，**只能安全地调用异步信号安全 (async-signal-safe) 的函数** (参见 `signal-safety(7)`)。这意味着在子进程中进行复杂的 C++ 操作（如内存分配/释放、使用大部分标准库函数、I/O 流操作）是不安全的，除非你确信它们是异步信号安全的。
*   **文件描述符:** 子进程**继承**父进程打开文件描述符集合的**副本**。每个子进程的文件描述符指向与父进程对应描述符**相同的打开文件描述** (open file description, 参见 `open(2)`)。这意味着它们共享文件状态标志、文件偏移量（读写位置）和信号驱动 I/O 属性 (`fcntl(2)` 的 `F_SETOWN`, `F_SETSIG`)。**C++ Core Guidelines (资源管理):** 继承的文件描述符也需要管理。如果父进程使用了 RAII 管理描述符，子进程继承的副本需要特别处理，避免双重释放或资源管理混乱。通常推荐的做法是在 `fork` 后、`exec` 前，子进程关闭不需要的文件描述符。
*   **消息队列描述符:** 子进程继承父进程打开消息队列描述符 (`mq_overview(7)`) 的副本，共享相同的标志 (`mq_flags`)。
*   **目录流:** 子进程继承父进程打开目录流 (`opendir(3)`) 的副本。POSIX 标准称父子进程可能共享目录流位置；在 Linux/glibc 中它们**不共享**。

## 5. 返回值 (RETURN VALUE)
*   **成功时:**
    *   在**父进程**中，返回**子进程的 PID** (一个正整数)。
    *   在**子进程**中，返回 **0**。
*   **失败时:**
    *   在**父进程**中，返回 **-1**。
    *   **不会创建**子进程。
    *   `errno` 被设置以指示错误原因。

## 6. 错误 (ERRORS)
`fork()` 可能因为以下原因失败并设置 `errno`：
*   `EAGAIN`: 遇到了系统对线程/进程数量的限制（如 `RLIMIT_NPROC`、`/proc/sys/kernel/threads-max`、`/proc/sys/kernel/pid_max` 或 cgroup 的 PID 限制）。或者，调用者使用 `SCHED_DEADLINE` 调度策略但没有设置 reset-on-fork 标志。
*   `ENOMEM`: 内核无法分配必要的结构，因为内存不足。或者尝试在已终止 "init" 进程的 PID 命名空间中创建子进程。
*   `ENOSYS`: 当前平台不支持 `fork()` (例如，没有内存管理单元 MMU 的硬件)。
*   `ERESTARTNOINTR`: (Linux 2.6.17+) 系统调用被信号中断，将自动重启（通常只在 `ptrace` 时可见）。

## 7. C 库/内核差异 (VERSIONS)
从 glibc 2.3.3 开始，作为 NPTL 线程实现的一部分，glibc 的 `fork()` 包装器不再直接调用内核的 `fork()` 系统调用，而是调用 `clone(2)` 并传入能产生相同效果的标志（主要是 `SIGCHLD`）。这个包装器还会调用通过 `pthread_atfork(3)` 注册的处理程序。

## 8. 标准 (STANDARDS)
POSIX.1-2008.

## 9. 历史 (HISTORY)
POSIX.1-2001, SVr4, 4.3BSD.

## 10. 注意事项 (NOTES)
Linux 下的 `fork()` 使用**写时复制 (Copy-on-Write, COW)** 页面实现。这意味着 `fork()` 的主要开销在于复制父进程的页表和为子进程创建唯一的任务结构所需的时间和内存。实际的物理内存页只有在父进程或子进程尝试**写入**时才会被复制。

## 11. 示例 (EXAMPLES)

以下是一个简单的 C++ 示例，演示 `fork()` 的基本用法。

```cpp
#include <iostream>
#include <unistd.h> // fork(), getpid(), getppid()
#include <sys/wait.h> // wait()
#include <cstdlib> // exit(), EXIT_SUCCESS, EXIT_FAILURE
#include <cstdio> // perror()

int main() {
    std::cout << "程序开始 (PID: " << getpid() << ")" << std::endl;

    pid_t child_pid = fork(); // 创建子进程

    // fork() 失败
    if (child_pid < 0) {
        perror("fork 失败");
        return EXIT_FAILURE;
    }
    // fork() 成功，当前是子进程
    else if (child_pid == 0) {
        std::cout << "  [子进程] 我的 PID 是: " << getpid() << std::endl;
        std::cout << "  [子进程] 我的父进程 PID 是: " << getppid() << std::endl;
        // 子进程可以执行不同的任务，例如调用 exec 系列函数加载新程序
        // 为了演示，这里只打印信息然后退出
        std::cout << "  [子进程] 正在退出..." << std::endl;
        exit(EXIT_SUCCESS); // 子进程正常退出
    }
    // fork() 成功，当前是父进程
    else {
        std::cout << "[父进程] 我的 PID 是: " << getpid() << std::endl;
        std::cout << "[父进程] 我创建的子进程 PID 是: " << child_pid << std::endl;

        // 父进程通常需要等待子进程结束
        std::cout << "[父进程] 等待子进程结束..." << std::endl;
        int status;
        // wait() 会阻塞父进程，直到其任意一个子进程结束
        // 返回结束子进程的 PID，并将退出状态信息存入 status
        pid_t terminated_pid = wait(&status);

        if (terminated_pid == -1) {
            perror("wait 失败");
        } else {
            if (WIFEXITED(status)) { // 检查子进程是否正常退出
                std::cout << "[父进程] 子进程 " << terminated_pid << " 已正常退出，退出码: " << WEXITSTATUS(status) << std::endl;
            } else if (WIFSIGNALED(status)) { // 检查子进程是否因信号而终止
                 std::cout << "[父进程] 子进程 " << terminated_pid << " 被信号 " << WTERMSIG(status) << " 终止。" << std::endl;
            } else {
                 std::cout << "[父进程] 子进程 " << terminated_pid << " 异常结束。" << std::endl;
            }
        }

        std::cout << "[父进程] 正在退出..." << std::endl;
    }

    return EXIT_SUCCESS; // 父进程正常退出
}
```

**编译与运行:**

```bash
g++ your_fork_program.cpp -o your_fork_program
./your_fork_program
```

**可能的输出 (PID 会变化):**

```
程序开始 (PID: 12345)
[父进程] 我的 PID 是: 12345
[父进程] 我创建的子进程 PID 是: 12346
[父进程] 等待子进程结束...
  [子进程] 我的 PID 是: 12346
  [子进程] 我的父进程 PID 是: 12345
  [子进程] 正在退出...
[父进程] 子进程 12346 已正常退出，退出码: 0
[父进程] 正在退出...
```
*(注意：父子进程的输出顺序可能因调度而不同)*

这个例子展示了：
*   如何调用 `fork()`。
*   如何根据 `fork()` 的返回值区分父子进程。
*   子进程如何获取自己的 PID (`getpid()`) 和父进程的 PID (`getppid()`)。
*   父进程如何使用 `wait()` 等待子进程结束并获取其退出状态。

## 12. 参见 (SEE ALSO)
`clone(2)`, `execve(2)`, `exit(2)`, `_exit(2)`, `setrlimit(2)`, `unshare(2)`, `vfork(2)`, `wait(2)`, `daemon(3)`, `pthread_atfork(3)`, `capabilities(7)`, `credentials(7)`

---
