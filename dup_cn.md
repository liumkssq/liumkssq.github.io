# dup(2) 系统调用详解

本文档基于 Linux man-pages 6.13 (2024-11-01) 对 `dup`、`dup2` 和 `dup3` 系统调用进行详细的中文解释。这些调用用于复制文件描述符。

---

## 1. 名称 (NAME)
`dup`, `dup2`, `dup3` - 复制一个文件描述符

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <unistd.h>

int dup(int oldfd);
int dup2(int oldfd, int newfd);

// dup3 需要定义 _GNU_SOURCE 宏
#define _GNU_SOURCE             /* 参见 feature_test_macros(7) */
#include <fcntl.h>              /* O_* 常量的定义 */
#include <unistd.h>

int dup3(int oldfd, int newfd, int flags);
```

## 4. 描述 (DESCRIPTION)

这些系统调用创建一个新的文件描述符，使其指向与现有文件描述符 `oldfd` **相同的打开文件描述 (open file description)**。

**核心概念：文件描述符 vs. 打开文件描述**
*   **文件描述符 (File Descriptor):** 一个小的非负整数，是内核在进程打开文件（或套接字、管道等）时返回的标识。它是进程文件描述符表中的一个索引。
*   **打开文件描述 (Open File Description):** 内核内部的一个结构体，用于维护一个打开文件的状态信息，例如文件偏移量（当前读写位置）、文件状态标志（如 `O_APPEND`）和访问模式（读/写）。多个文件描述符可以指向同一个打开文件描述。

### `dup(oldfd)`
*   **功能:** 分配一个新的文件描述符，使其指向与 `oldfd` 相同的打开文件描述。
*   **新描述符号码:** 内核保证分配的是当前进程中**未使用的最小编号**的文件描述符。
*   **共享特性:**
    *   **共享:** 新旧文件描述符共享文件偏移量和文件状态标志。例如，在一个描述符上使用 `lseek(2)` 修改文件偏移量，另一个描述符的偏移量也会随之改变。
    *   **不共享:** 它们**不共享**文件描述符标志，特别是**执行时关闭 (close-on-exec)** 标志 (`FD_CLOEXEC`)。新复制出来的描述符的 `FD_CLOEXEC` 标志总是**关闭**的。

### `dup2(oldfd, newfd)`
*   **功能:** 与 `dup()` 类似，但它不使用最小可用编号，而是强制使用 `newfd` 指定的编号作为新的文件描述符。换句话说，调整 `newfd` 使其指向与 `oldfd` 相同的打开文件描述。
*   **处理 `newfd`:**
    *   如果 `newfd` 原本已经打开，`dup2()` 会在重用它之前**原子地**将其关闭。这个关闭操作是**静默**的，即关闭过程中发生的任何错误都不会被 `dup2()` 报告。
    *   关闭并重用 `newfd` 的步骤是**原子**操作。这一点很重要，因为它避免了使用 `close(2)` 和 `dup()` 分步实现时可能出现的**竞态条件**（在 `close` 和 `dup` 之间，`newfd` 可能被其他线程或信号处理程序重新分配使用）。
*   **注意事项:**
    *   如果 `oldfd` 不是一个有效的文件描述符，调用会失败，并且 `newfd` **不会**被关闭。
    *   如果 `oldfd` 是有效的，且 `newfd` 与 `oldfd` 的值相同，`dup2()` 不做任何操作，并成功返回 `newfd`。

### `dup3(oldfd, newfd, flags)`
*   **功能:** 与 `dup2()` 基本相同，但增加了 `flags` 参数来控制行为。
*   **`flags` 参数:**
    *   `O_CLOEXEC`: 如果在 `flags` 中指定了这个标志，那么新的文件描述符 (`newfd`) 将会被设置**执行时关闭 (`FD_CLOEXEC`)** 标志。这对于防止文件描述符在执行 `execve(2)` 后被子进程继承非常有用。
*   **与 `dup2()` 的区别:** 如果 `oldfd` 等于 `newfd`，`dup3()` 会失败并返回 `EINVAL` 错误（而 `dup2()` 在这种情况下会成功）。

## 5. 返回值 (RETURN VALUE)
*   **成功:** 这三个系统调用都返回新的文件描述符 (`dup()` 和 `dup3()` 返回新分配的编号，`dup2()` 返回 `newfd` 的值)。
*   **失败:** 返回 -1，并设置 `errno` 来指示具体的错误原因。

## 6. 错误 (ERRORS)
常见的错误码包括：
*   `EBADF`: `oldfd` 不是一个打开的文件描述符；或者（对于 `dup2`/`dup3`）`newfd` 超出了允许的文件描述符范围（参考 `getrlimit(2)` 中的 `RLIMIT_NOFILE`）。
*   `EBUSY`: (仅 Linux) 在与 `open(2)` 和 `dup()` 发生竞态条件时，`dup2()` 或 `dup3()` 可能返回此错误。
*   `EINTR`: `dup2()` 或 `dup3()` 调用被信号中断；参见 `signal(7)`。
*   `EINVAL`: (`dup3()` 特有) `flags` 包含了无效的值；或者 `oldfd` 等于 `newfd`。
*   `EMFILE`: 进程打开的文件描述符数量已达到上限（参考 `getrlimit(2)` 中的 `RLIMIT_NOFILE`）。
*   `ENOMEM`: 可用的内核内存不足。

## 7. 标准 (STANDARDS)
*   `dup()`, `dup2()`: POSIX.1-2008.
*   `dup3()`: Linux 特有。

## 8. 历史 (HISTORY)
*   `dup()`, `dup2()`: 存在于 POSIX.1-2001, SVr4, 4.3BSD。
*   `dup3()`: Linux 内核 2.6.27, glibc 2.9 引入。

## 9. 注意事项 (NOTES)
*   **`newfd` 超出范围的错误:** `dup2()` 在 `newfd` 超出范围时返回的错误 (`EBADF`) 不同于 `fcntl(..., F_DUPFD, ...)` 可能返回的 `EINVAL`。
*   **丢失的 `close()` 错误:** 因为 `dup2()` 会静默地关闭已存在的 `newfd`，所以在关闭原 `newfd` 时可能发生的错误会丢失。如果关心这个错误，man page 建议不要在调用 `dup2()` 之前手动 `close(newfd)`（因为有竞态条件），而是可以先 `dup(newfd)` 得到一个临时副本，然后调用 `dup2(oldfd, newfd)`，最后再 `close()` 那个临时副本并检查其返回值，以此来捕获原 `newfd` 关闭时可能发生的错误。

## 10. 参见 (SEE ALSO)
`close(2)`, `fcntl(2)`, `open(2)`, `pidfd_getfd(2)`
