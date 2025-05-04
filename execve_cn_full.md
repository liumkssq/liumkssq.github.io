# execve(2) 系统调用详解 (结合 C++ 与 Linux 实践)

本文档基于 Linux man-pages 6.13 (2024-07-23) 对 `execve` 系统调用进行详细的中文解释。内容将结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 的实践原则（特别是资源管理和错误处理）、C++ Reference [https://en.cppreference.com/w/cpp/memory/c] 的上下文、Tutorialspoint [https://www.tutorialspoint.com/unix_system_calls/execve.htm] 的核心概念以及 Linux [https://www.kernel.org/doc/html/latest/translations/zh_CN/core-api/memory-allocation.html] 特有的行为进行探讨，并说明其与新程序 `main` 函数的关系。

**核心概念:** `execve` 是一个强大的系统调用，它**用一个全新的程序替换当前正在运行的进程映像**。调用成功后，原程序的代码、数据、堆栈等都会被新程序覆盖，但进程 ID (PID) 保持不变。

---

## 1. 名称 (NAME)
`execve` - 执行程序

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <unistd.h> // POSIX 标准头文件

// pathname: 要执行的程序路径
// argv: 传递给新程序的命令行参数数组 (必须以 NULL 结尾)
// envp: 传递给新程序的环境变量数组 (必须以 NULL 结尾)
int execve(const char *pathname, char *const _Nullable argv[],
           char *const _Nullable envp[]);
```
*   `_Nullable` 注解 (非标准，但常见于文档) 提示 `argv` 和 `envp` 本身可以是 `NULL` (尽管不推荐依赖此特性，见 VERSIONS)。

## 4. 描述 (DESCRIPTION)

`execve()` 执行由 `pathname` 指向的程序。这会导致当前调用进程正在运行的程序被一个**新程序替换**，新程序拥有全新初始化的栈、堆以及（初始化和未初始化的）数据段。

`pathname` 必须是以下两者之一：
1.  一个**二进制可执行文件**。
2.  一个**解释器脚本**，其起始行格式为： `#!interpreter [optional-arg]` (详见下文 "解释器脚本")。

`argv` 是一个指向字符串的指针数组，作为新程序的**命令行参数**传递。按照惯例，`argv[0]` 应该包含正在执行的文件名。`argv` 数组**必须以 `NULL` 指针结尾**。因此，在新程序中，`argv[argc]` 将是 `NULL`。

`envp` 是一个指向字符串的指针数组，通常格式为 `key=value`，作为新程序的**环境变量**传递。`envp` 数组**必须以 `NULL` 指针结尾**。

**与 `exec(3)` 系列函数的关系:** `execve` 是最底层的 `exec` 函数，也是内核实际提供的系统调用。C 库提供了许多更易用的变体（如 `execl`, `execv`, `execlp`, `execvp`, `execle`, `execvpe`），它们通常封装了 `execve`，并提供了诸如自动搜索 `PATH` 环境变量、接受可变参数列表等便利功能。除非需要精确控制环境变量，否则通常优先使用 `exec(3)` 系列函数。

**新程序的 `main` 函数:** 传递给 `execve` 的 `argv` 和 `envp` 可以被新程序的 `main` 函数访问，当其定义为：
```c
int main(int argc, char *argv[], char *envp[])
```
然而，`main` 函数的第三个参数 (`envp`) **并非 POSIX.1 标准规定**。POSIX.1 推荐通过外部变量 `environ` (参见 `environ(7)`) 来访问环境变量。

**执行流程:**
*   `execve()` **成功时不会返回**到调用程序。控制权完全转移给新程序。
*   原进程的文本段、初始化数据段、未初始化数据段 (bss) 和栈会被新程序的内容覆盖。
*   如果当前进程正在被 `ptrace(2)` 跟踪，`execve` 成功后会向其发送一个 `SIGTRAP` 信号。

**权限与 ID 变化:**
*   **Set-User-ID (SUID) / Set-Group-ID (SGID):** 如果 `pathname` 指向的文件设置了 SUID 位，则调用进程的**有效用户 ID (effective user ID)** 会变为文件所有者的 UID。类似地，如果设置了 SGID 位，则**有效组 ID (effective group ID)** 会变为文件所属组的 GID。
*   **忽略 SUID/SGID 的情况:** 以下情况会忽略 SUID/SGID 位：
    *   调用线程设置了 `no_new_privs` 属性 (`prctl(2)`)。
    *   底层文件系统以 `nosuid` 模式挂载 (`mount(2)` 的 `MS_NOSUID` 标志)。
    *   调用进程正在被 `ptrace(2)` 跟踪。
*   **能力 (Capabilities):** 如果上述任一情况为真，程序文件的能力 (`capabilities(7)`) 也会被忽略。
*   **保存的 Set-ID:** 进程的有效用户 ID 会被复制到**保存的 set-user-ID (saved set-user-ID)**。类似地，有效组 ID 会被复制到**保存的 set-group-ID (saved set-group-ID)**。这个复制发生在 SUID/SGID 位导致的有效 ID 变更**之后**。
*   **实际 ID 和附加组:** 进程的**实际用户 ID (real UID)**、**实际组 ID (real GID)** 以及**附加组 ID** 在 `execve` 调用中**保持不变**。

**动态链接:**
*   **a.out 格式:** 如果是 a.out 动态链接可执行文件，Linux 动态链接器 `ld.so(8)` 会在执行开始时被调用，加载所需的共享库并进行链接。
*   **ELF 格式:** 如果是 ELF 动态链接可执行文件，`PT_INTERP` 段中指定的解释器（通常是 glibc 的 `/lib/ld-linux.so.2`，参见 `ld-linux.so(8)`) 会被用来加载所需的共享对象。

### 对进程属性的影响

`execve` 调用会保留大部分进程属性，但以下属性**不会被保留** (会被重置或清除)：

**POSIX.1 标准规定的不保留属性:**
*   **信号处理方式:** 所有被捕获 (caught) 的信号的处理方式被**重置为默认** (`signal(7)`)。被忽略或设为默认的信号处理方式通常不变 (SIGCHLD 是例外，见下文)。
*   **备用信号栈:** 不保留 (`sigaltstack(2)`)。
*   **内存映射:** 不保留 (`mmap(2)`)。
*   **System V 共享内存段:** 分离 (`shmat(2)`)。
*   **POSIX 共享内存区:** 取消映射 (`shm_open(3)`)。
*   **POSIX 消息队列描述符:** 关闭 (`mq_overview(7)`)。
*   **POSIX 命名信号量:** 关闭 (`sem_overview(7)`)。
*   **POSIX 定时器:** 不保留 (`timer_create(2)`)。
*   **打开的目录流:** 关闭 (`opendir(3)`)。
*   **内存锁:** 不保留 (`mlock(2)`, `mlockall(2)`)。
*   **退出处理程序:** 不保留 (`atexit(3)`, `on_exit(3)`)。
*   **浮点环境:** 重置为默认 (`fenv(3)`)。

**Linux 特有的不保留属性:**
*   **dumpable 属性:** 通常重置为 1。但在执行 SUID/SGID 程序或带能力的程序时，可能根据 `/proc/sys/fs/suid_dumpable` 的值重置 (见 `prctl(2)` 的 `PR_SET_DUMPABLE`)。改变 dumpable 可能导致 `/proc/<pid>` 目录下的文件所有权变为 `root:root`。
*   **`PR_SET_KEEPCAPS` 标志:** 清除 (`prctl(2)`)。
*   **父进程死亡信号:** 如果执行 SUID/SGID 程序，`PR_SET_PDEATHSIG` 设置的信号被清除 (`prctl(2)`)。
*   **进程名:** 重置为新可执行文件的名称 (`prctl(2)` 的 `PR_SET_NAME`)。
*   **`SECBIT_KEEP_CAPS` securebits 标志:** 清除 (`capabilities(7)`)。
*   **终止信号:** 重置为 `SIGCHLD` (`clone(2)`)。
*   **文件描述符表共享:** 取消共享，撤销 `clone(2)` 的 `CLONE_FILES` 标志效果。

**其他重要注意事项:**
*   **多线程:** `execve` 调用期间，**除调用线程外的所有其他线程都会被销毁**。互斥锁、条件变量和其他 pthreads 对象的状态不会被保留。**C++ Core Guidelines (CP.con):** 这再次强调了 `fork` 后立即 `execve` 的重要性，避免在子进程中执行复杂的多线程不安全代码。
*   **Locale:** 程序启动时，效果等同于执行了 `setlocale(LC_ALL, "C")`。
*   **`SIGCHLD` 处理:** POSIX.1 规定被忽略或设为默认的信号处理不变，但 `SIGCHLD` 是例外：实现可以保持忽略状态或重置为默认。Linux 选择**保持忽略状态**。
*   **异步 I/O:** 未完成的异步 I/O 操作会被取消 (`aio_read(3)`, `aio_write(3)`)。
*   **能力处理:** 详见 `capabilities(7)`。
*   **文件描述符:**
    *   默认情况下，文件描述符在 `execve` 后保持打开。
    *   标记为**执行时关闭 (close-on-exec, `FD_CLOEXEC`)** 的文件描述符会被关闭 (见 `fcntl(2)`)。关闭描述符会释放该进程在该文件上获得的所有记录锁。
    *   **C++ Core Guidelines (资源管理):** 这是 RAII 特别重要的场景。对于不需要传递给新程序的资源（尤其是文件描述符），应在调用 `execve` 前确保它们被关闭。对于需要传递的描述符（如重定向后的 stdin/stdout/stderr），需要确保它们没有设置 `FD_CLOEXEC` 标志。使用 RAII 包装器管理文件描述符时，需要在 `execve` 前适当地处理它们（例如，调用 `release()` 放弃所有权，或确保析构函数不关闭需要保留的描述符）。
    *   POSIX.1 特殊情况：如果文件描述符 0, 1, 2 (stdin, stdout, stderr) 本应被关闭，并且新程序因为 SUID/SGID 位获得了特权，系统可能会为这些描述符打开未指定的文件。因此，可移植程序（无论是否特权）不能假设这三个描述符在 `execve` 后一定保持关闭状态。

### 解释器脚本 (Interpreter Scripts)

解释器脚本是一个具有执行权限的文本文件，其第一行格式为：
```
#!interpreter [optional-arg]
```
`interpreter` 必须是一个有效的可执行文件路径。

当 `execve` 的 `pathname` 参数指向一个解释器脚本时，内核实际上会执行 `interpreter`，并传递以下参数给它：
```
interpreter [optional-arg] pathname arg...
```
其中：
*   `interpreter` 是 `#!` 后指定的解释器路径。
*   `[optional-arg]` 是解释器路径后的可选参数（如果有）。
*   `pathname` 是原始传递给 `execve` 的脚本文件路径。
*   `arg...` 是原始传递给 `execve` 的 `argv` 数组中从 `argv[1]` 开始的所有参数。
*   **注意:** 原始的 `argv[0]` (通常是调用程序自身的名称) 无法通过这种方式传递给解释器。

为了可移植性，`optional-arg` 要么不存在，要么应该是一个**没有空格的单词**。

从 Linux 2.6.28 开始，内核允许解释器本身也是一个脚本（递归解释），最多支持 4 层递归。

### 参数和环境的大小限制

大多数 UNIX 系统限制了传递给新程序的命令行参数 (`argv`) 和环境变量 (`envp`) 的总大小。POSIX.1 允许实现通过 `ARG_MAX` 常量（定义在 `<limits.h>` 或通过 `sysconf(_SC_ARG_MAX)` 获取）来声明这个限制。

*   **Linux 2.6.23 之前:** 限制为 32 个页 (内核常量 `MAX_ARG_PAGES`)，在 4KB 页大小的系统上约为 128KB。
*   **Linux 2.6.23 及之后:** 大多数架构的限制基于调用 `execve` 时生效的软 `RLIMIT_STACK` 资源限制 (`getrlimit(2)`)。总大小限制为允许栈大小的 1/4，且不超过内核常量 `_STK_LIM` (8 MiB) 的 3/4。同时，有一个最低 32 页的保证，以及每个字符串 32 页 (`MAX_ARG_STRLEN`) 和最多 `0x7FFFFFFF` 个字符串的限制。

## 5. 返回值 (RETURN VALUE)
*   **成功时:** `execve()` **不会返回**。
*   **失败时:** 返回 **-1**，并设置 `errno` 以指示错误原因。控制权返回给调用程序。

## 6. 错误 (ERRORS)
`execve` 可能因多种原因失败，常见的 `errno` 值包括：
*   `E2BIG`: 参数列表 (`argv`) 和环境列表 (`envp`) 的总字节数太大，或单个字符串太长，或可执行文件完整路径太长。
*   `EACCES`: 路径前缀的搜索权限被拒绝；文件或脚本解释器不是常规文件；文件或解释器没有执行权限；文件系统以 `noexec` 模式挂载。
*   `EAGAIN`: (Linux 3.1+) 调用者之前通过 `set*uid()` 改变了真实 UID，导致超过了 `RLIMIT_NPROC` 限制，并且在调用 `execve` 时限制仍然超出 (详见 NOTES)。
*   `EFAULT`: `pathname`, `argv` 或 `envp` 中的指针指向了不可访问的地址空间。
*   `EINVAL`: ELF 可执行文件有多个 `PT_INTERP` 段。
*   `EIO`: 发生 I/O 错误。
*   `EISDIR`: ELF 解释器是一个目录。
*   `ELIBBAD`: ELF 解释器格式无法识别。
*   `ELOOP`: 解析 `pathname` 或解释器路径时遇到过多的符号链接，或者达到脚本递归解释的最大限制。
*   `EMFILE`: 进程打开的文件描述符数量达到上限。
*   `ENAMETOOLONG`: `pathname` 太长。
*   `ENFILE`: 系统范围内的打开文件总数达到上限。
*   `ENOENT`: `pathname` 指定的文件、脚本解释器或 ELF 解释器不存在，或者所需的共享库找不到。
*   `ENOEXEC`: 可执行文件格式无法识别、架构不匹配或存在其他格式错误导致无法执行。
*   `ENOMEM`: 内核内存不足。
*   `ENOTDIR`: `pathname` 或解释器路径中的某个组件不是目录。
*   `EPERM`: 文件系统以 `nosuid` 挂载，用户非超级用户，且文件设置了 SUID/SGID 位；进程被跟踪，用户非超级用户，且文件设置了 SUID/SGID 位；"capability-dumb" 应用无法获得文件授予的全部能力集。
*   `ETXTBSY`: 指定的可执行文件正被一个或多个进程以写入模式打开。

## 7. 版本 (VERSIONS)
*   `#!` 行为并非 POSIX 标准，但在其他 UNIX 系统上存在变体。
*   **Linux 特有 (不推荐依赖):** `argv` 和 `envp` 可以是 `NULL`，效果等同于指向只包含 `NULL` 的列表。其他系统可能报错 (`EFAULT`)。
*   Linux 2.6.23 后，`_SC_ARG_MAX` 的值可能因 `RLIMIT_STACK` 改变而改变。
*   **解释器脚本 `#!` 后文本长度限制:** Linux 5.1 前为 127 字符，之后为 255 字符。
*   `optional-arg` 的处理在不同系统上有差异，Linux 将其视为单个参数（可包含空格）。
*   Linux（和其他现代 UNIX）忽略脚本文件的 SUID/SGID 位。

## 8. 标准 (STANDARDS)
POSIX.1-2008.

## 9. 历史 (HISTORY)
POSIX.1-2001, SVr4, 4.3BSD.

## 10. 注意事项 (NOTES)
*   **`execve` 不创建新进程:** 它只是让**现有进程**执行一个**新程序**，PID 不变。
*   SUID/SGID 进程不能被 `ptrace(2)`。
*   `nosuid` 挂载的影响因内核版本而异。
*   **极端失败情况:** 极少数情况下（通常是资源耗尽），`execve` 可能在破坏了原程序映像但未能完全建立新映像时失败。此时内核会用 `SIGSEGV` (Linux 3.17 前是 `SIGKILL`) 信号杀死该进程。
*   **`EAGAIN` 错误的详细说明 (Linux 3.1+):** 当 `set*uid()` 调用成功改变了真实 UID 但导致进程超出 `RLIMIT_NPROC` 限制时，内核会设置一个内部标志 `PF_NPROC_EXCEEDED`。如果此标志被设置，并且在随后的 `execve()` 调用时资源限制仍然超出，`execve()` 就会失败并返回 `EAGAIN`。这是为了确保 `RLIMIT_NPROC` 限制在常见的特权守护进程工作流 (`fork` + `set*uid` + `execve`) 中仍然有效。如果在 `execve` 调用时限制不再超出（例如其他进程已退出），`execve` 会成功并清除该标志。`fork` 成功也会清除该标志。

## 11. C++ 示例 (fork + execve)

这个例子演示了在 C++ 中使用 `fork()` 创建子进程，然后在子进程中使用 `execve()` 执行另一个程序 (`ls -l /`) 的常见模式。

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unistd.h>    // fork(), execve(), getpid()
#include <sys/wait.h>  // waitpid()
#include <cstdlib>     // exit(), EXIT_SUCCESS, EXIT_FAILURE
#include <cstdio>      // perror()
#include <cstring>     // strcmp() for basic arg check

// Helper function to convert std::vector<std::string> to char*[] for execve
// Note: The vector must contain C-style strings (null-terminated)
std::vector<char*> create_argv(const std::vector<std::string>& args) {
    std::vector<char*> argv;
    argv.reserve(args.size() + 1); // +1 for the terminating NULL
    for (const auto& arg : args) {
        // const_cast is generally discouraged, but necessary for the C-style
        // execve interface which doesn't have const correctness for argv.
        // We assume execve won't modify the strings.
        argv.push_back(const_cast<char*>(arg.c_str()));
    }
    argv.push_back(nullptr); // Must be null-terminated
    return argv;
}

int main() {
    std::cout << "[父进程 " << getpid() << "] 准备 fork...\n";

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork 失败");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        // --- 子进程 ---
        std::cout << "  [子进程 " << getpid() << "] 开始执行 execve...\n";

        // 准备要执行的程序路径、参数和环境变量
        const char* program_path = "/bin/ls"; // 要执行的程序

        // 准备 argv (命令行参数)
        std::vector<std::string> args = {"ls", "-l", "/"}; // argv[0] 通常是程序名
        std::vector<char*> argv = create_argv(args);

        // 准备 envp (环境变量)
        // 这里我们传递一个空的环境变量列表，新进程将拥有最小的环境
        // 你也可以复制父进程的环境 extern char **environ; 或自定义
        char* const envp[] = {nullptr}; // 必须以 NULL 结尾

        // 调用 execve 执行新程序
        // 如果 execve 成功，下面的代码将不会被执行
        execve(program_path, argv.data(), envp);

        // 如果 execve 返回，说明发生了错误
        perror("  [子进程] execve 失败");
        exit(EXIT_FAILURE); // 子进程异常退出

    } else {
        // --- 父进程 ---
        std::cout << "[父进程 " << getpid() << "] 等待子进程 " << pid << " 结束...\n";

        int status;
        // 等待特定的子进程 pid 结束
        if (waitpid(pid, &status, 0) == -1) {
            perror("waitpid 失败");
        } else {
            if (WIFEXITED(status)) {
                std::cout << "[父进程 " << getpid() << "] 子进程 " << pid << " 正常退出，退出码: " << WEXITSTATUS(status) << std::endl;
            } else if (WIFSIGNALED(status)) {
                std::cout << "[父进程 " << getpid() << "] 子进程 " << pid << " 被信号 " << WTERMSIG(status) << " 终止。\n";
            } else {
                 std::cout << "[父进程 " << getpid() << "] 子进程 " << pid << " 异常结束。\n";
            }
        }
         std::cout << "[父进程 " << getpid() << "] 退出。\n";
    }

    return EXIT_SUCCESS;
}
```

**编译与运行:**

```bash
g++ your_execve_program.cpp -o your_execve_program
./your_execve_program
```

**可能的输出:**

```
[父进程 1234] 准备 fork...
[父进程 1234] 等待子进程 1235 结束...
  [子进程 1235] 开始执行 execve...
total <some_number>
lrwxrwxrwx 1 root root ... bin -> usr/bin
drwxr-xr-x 1 root root ... boot
... (ls -l / 的输出) ...
drwxr-xr-x 1 root root ... var
[父进程 1234] 子进程 1235 正常退出，退出码: 0
[父进程 1234] 退出。
```

这个 C++ 示例展示了标准的 `fork-exec` 模式：父进程 `fork` 出子进程，子进程调用 `execve` 来执行一个新程序 (`ls`)，父进程使用 `waitpid` 等待子进程完成。注意参数和环境变量是如何构造成 `char*` 数组并传递给 `execve` 的。

## 12. 参见 (SEE ALSO)
`chmod(2)`, `execveat(2)`, `fork(2)`, `get_robust_list(2)`, `ptrace(2)`, `exec(3)`, `fexecve(3)`, `getauxval(3)`, `getopt(3)`, `system(3)`, `capabilities(7)`, `credentials(7)`, `environ(7)`, `path_resolution(7)`, `ld.so(8)`

---
