# dup(2) 系统调用详解 (结合 C++ 实践原则)

本文档基于 Linux man-pages 6.13 (2024-11-01) 对 `dup`, `dup2`, 和 `dup3` 系统调用进行详细的中文解释，并结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 中的资源管理和错误处理原则进行探讨。这些调用用于复制文件描述符，在 C 和 C++ 底层编程中处理 I/O 重定向、管道连接等场景时非常关键。

---

## 1. 名称 (NAME)
`dup`, `dup2`, `dup3` - 复制一个文件描述符

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <unistd.h> // POSIX 标准头文件

int dup(int oldfd);
int dup2(int oldfd, int newfd);

// dup3 是 GNU 扩展，需要定义 _GNU_SOURCE 宏
#define _GNU_SOURCE
#include <fcntl.h> // O_* 常量的定义
#include <unistd.h>

int dup3(int oldfd, int newfd, int flags);
```

## 4. 描述 (DESCRIPTION)

这些系统调用创建一个新的文件描述符，使其指向与现有文件描述符 `oldfd` **相同的打开文件描述 (open file description)**。

**核心概念：文件描述符 vs. 打开文件描述**
*   **文件描述符 (File Descriptor):** 内核为每个进程维护的一个小的非负整数索引，指向进程打开文件表中的一个条目。
*   **打开文件描述 (Open File Description):** 内核中代表一个已打开文件的实体，存储了文件状态（如 `O_APPEND`）、访问模式（读/写）、当前文件偏移量（读写指针位置）等信息。多个文件描述符可以指向同一个打开文件描述。

### `dup(oldfd)`
*   **功能:** 分配一个新的文件描述符，使其指向与 `oldfd` 相同的打开文件描述。
*   **新描述符号码:** 内核保证分配的是当前进程中**未使用的最小编号**的文件描述符。
*   **共享特性:**
    *   **共享:** 新旧文件描述符共享同一个打开文件描述，因此它们共享文件偏移量和文件状态标志。对一个描述符使用 `lseek(2)` 修改偏移量会影响另一个。
    *   **不共享:** 文件描述符标志（特别是**执行时关闭 (close-on-exec)** 标志 `FD_CLOEXEC`）不共享。新描述符的 `FD_CLOEXEC` 标志总是**关闭**的。

### `dup2(oldfd, newfd)`
*   **功能:** 与 `dup()` 类似，但强制使用 `newfd` 指定的编号作为新的文件描述符。
*   **处理 `newfd`:**
    *   如果 `newfd` 原本已经打开，`dup2()` 会在重用它之前**原子地**将其关闭。这个关闭操作是**静默**的，任何潜在的关闭错误都会被忽略。
    *   **原子性**是 `dup2` 的关键优势，避免了 `close(newfd)` 和 `dup(oldfd)` 分步操作可能引入的竞态条件。
*   **C++ 核心指南视角 (错误处理):** `dup2` 静默关闭 `newfd` 的行为隐藏了可能的错误。如果确保 `newfd` 被正确关闭很重要，应遵循 man page 中建议的更复杂的错误检查模式（先 `dup` `newfd` 自身，再 `dup2`，最后 `close` 临时描述符并检查错误），或者在 C++ 中考虑使用 RAII 包装器来管理文件描述符生命周期，虽然直接封装 `dup2` 的原子性比较困难。
*   **注意事项:**
    *   若 `oldfd` 无效，调用失败，`newfd` 不会被关闭。
    *   若 `oldfd` 有效且 `oldfd == newfd`，`dup2()` 不做任何事，成功返回 `newfd`。

### `dup3(oldfd, newfd, flags)`
*   **功能:** 与 `dup2()` 基本相同，但增加了 `flags` 参数。
*   **`flags` 参数:**
    *   `O_CLOEXEC`: 允许调用者在创建新描述符 `newfd` 时**原子地**设置 `FD_CLOEXEC` 标志。这符合 C++ Core Guidelines 中关于资源管理和防止资源泄露到子进程的建议。
*   **与 `dup2()` 的区别:** 若 `oldfd == newfd`，`dup3()` 会失败并返回 `EINVAL`。

## 5. 返回值 (RETURN VALUE)
*   **成功:** 返回新的文件描述符。
*   **失败:** 返回 -1，并设置 `errno` 指示错误。

## 6. 错误 (ERRORS)
常见错误码包括：
*   `EBADF`: `oldfd` 无效，或 `newfd` 超出范围。
*   `EBUSY`: (Linux 特有) 竞态条件。
*   `EINTR`: 被信号中断。
*   `EINVAL`: (`dup3`) `flags` 无效，或 `oldfd == newfd`。
*   `EMFILE`: 达到进程或系统文件描述符数量上限。
*   `ENOMEM`: 内核内存不足。

## 7. 标准 (STANDARDS)
*   `dup()`, `dup2()`: POSIX.1-2008.
*   `dup3()`: Linux 特有 (GNU 扩展)。

## 8. 历史 (HISTORY)
*   `dup()`, `dup2()`: 存在于 POSIX.1-2001, SVr4, 4.3BSD。
*   `dup3()`: Linux 内核 2.6.27, glibc 2.9 引入。

## 9. 注意事项 (NOTES)
*   `dup2` 的错误处理复杂性提示我们，在 C++ 中，直接使用底层系统调用需要格外小心。尽可能使用标准库提供的抽象（如 iostream 重定向）或可靠的第三方库来管理资源。当必须使用 `dup` 系列调用时，务必仔细处理错误情况和资源生命周期。
*   **C++ 核心指南视角 (资源管理):** 文件描述符是典型的需要 RAII (Resource Acquisition Is Initialization) 管理的资源。虽然标准库没有直接的 RAII 文件描述符包装器，但可以自行实现或使用 Boost 等库提供的类似功能，以确保即使在异常情况下也能正确关闭文件描述符，防止资源泄露。`dup` 系列函数创建了资源的别名，RAII 包装器在处理这种情况时需要特别注意所有权和生命周期管理。

## 10. 参见 (SEE ALSO)
`close(2)`, `fcntl(2)`, `open(2)`, `pidfd_getfd(2)`
