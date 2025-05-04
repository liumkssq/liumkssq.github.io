# mmap(2) 与 munmap(2) 系统调用详解 (结合 C++ 与 Linux 实践)

本文档基于 Linux man-pages 6.13 (2024-07-23) 对 `mmap` 和 `munmap` 系统调用进行详细的中文解释。内容将结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 的实践原则（特别是资源管理 R、指针使用）、C++ Reference [https://en.cppreference.com/w/c/memory] 上下文、IBM Docs [https://www.ibm.com/docs/en/zos/2.4.0?topic=functions-mmap-map-pages-memory] 和 Stack Overflow [https://stackoverflow.com/questions/26259421/use-mmap-in-c-to-write-into-memory] 的讨论，以及 Linux [https://www.kernel.org/doc/html/latest/translations/zh_CN/mm/index.html] 特有的行为进行探讨。`mmap` 是一个强大的工具，用于在进程的虚拟地址空间中创建文件或设备的内存映射。

**核心概念:** `mmap` 允许将文件内容或设备直接映射到进程的地址空间，使得可以像访问普通内存一样读写文件（或与设备交互），通常比传统的 `read()`/`write()` 系统调用效率更高，尤其是在处理大文件或需要频繁访问文件数据时。`munmap` 则用于解除这种映射。

---

## 1. 名称 (NAME)
`mmap`, `munmap` - 映射或取消映射文件或设备到内存

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)

## 3. 函数原型 (SYNOPSIS)
```c
#include <sys/mman.h> // 需要包含此头文件

// 创建内存映射
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);

// 解除内存映射
int munmap(void *addr, size_t length);

// (关于特性测试宏的要求，请参见 man page 的 VERSIONS 部分)
```

## 4. 描述 (DESCRIPTION)

### `mmap()`

`mmap()` 在调用进程的虚拟地址空间中创建一个新的**映射 (mapping)**。

*   **`addr` (地址建议):**
    *   指定映射区域的**起始地址**。
    *   如果 `addr` 为 `NULL`，内核会自行选择一个合适的（页对齐的）地址来创建映射。这是**最可移植**的方式。
    *   如果 `addr` 非 `NULL`，内核会将其视为一个**建议 (hint)**，尝试在 `addr` 附近（通常是大于等于 `/proc/sys/vm/mmap_min_addr` 的某个页边界）创建映射。如果该地址已被占用，内核会选择新的地址，不保证与建议相关。
    *   使用 `MAP_FIXED` 标志时，`addr` 不再是建议，而是**强制要求**的地址（详见下文 `flags`）。
*   **`length` (映射长度):**
    *   指定映射区域的**长度**（字节数），**必须大于 0**。
    *   内核实际操作的是整个页面，因此即使 `length` 不是页大小的整数倍，包含该范围的**所有页面**都会被映射。
*   **`prot` (内存保护):**
    *   描述映射区域的**访问权限**，不能与打开文件 `fd` 的模式冲突。
    *   它是 `PROT_NONE` (不可访问) 或以下一个或多个标志的**按位或 (bitwise OR)**：
        *   `PROT_EXEC`: 页面可执行。
        *   `PROT_READ`: 页面可读。
        *   `PROT_WRITE`: 页面可写。
    *   **注意:** `PROT_WRITE` 权限通常隐含 `PROT_READ` 权限 (i386 架构)。`PROT_READ` 是否隐含 `PROT_EXEC` 取决于具体架构。可移植程序若需执行映射区域代码，应显式设置 `PROT_EXEC`。
*   **`flags` (映射标志):**
    *   决定映射的更新是否对映射同一区域的其他进程可见，以及是否写回底层文件。
    *   **必须包含以下两者之一:**
        *   `MAP_SHARED`: **共享映射**。对映射的修改对映射同一区域的其他进程可见，并且（对于文件映射）会**写回底层文件** (写回时机可通过 `msync(2)` 控制)。
        *   `MAP_PRIVATE`: **私有写时复制 (Copy-on-Write) 映射**。对映射的修改**不**对其他进程可见，也**不**会写回底层文件。修改时会创建页面的私有副本。文件在 `mmap()` 调用后发生的改变是否在映射区域可见是未指定的。
    *   **可选标志 (按位或):**
        *   `MAP_ANONYMOUS` (或 `MAP_ANON`): **匿名映射**。不与任何文件关联，映射内容初始化为**零**。此时 `fd` 参数被**忽略**（可移植代码应设为 -1），`offset` 参数应为 0。常用于分配大块内存。与 `MAP_SHARED` 结合使用始于 Linux 2.4。
        *   `MAP_FIXED`: **强制使用 `addr`**。不将 `addr` 视为建议，精确地在该地址创建映射。`addr` 必须是页对齐的。如果该区域与现有映射重叠，**现有映射会被覆盖（危险！）**。如果地址无法使用，`mmap` 失败。**C++ Core Guidelines:** 强烈建议**避免**使用 `MAP_FIXED`，除非你完全确定该地址范围是预先保留且未被使用的，否则极易破坏进程地址空间，尤其在多线程环境下。
        *   `MAP_FIXED_NOREPLACE` (Linux 4.17+): 类似 `MAP_FIXED` 强制使用 `addr`，但**不会覆盖**现有映射。如果地址冲突，调用失败并返回 `EEXIST`。可用于原子地尝试映射特定地址范围。旧内核不识别此标志，可能回退到非 `MAP_FIXED` 行为（返回不同地址），因此需要检查返回值。
        *   `MAP_GROWSDOWN`: 用于栈。映射向下增长。返回地址比实际创建区域低一页。"保护页"被访问时映射自动增长一页。
        *   `MAP_HUGETLB` (Linux 2.6.32+): 使用**大页 (Huge Pages)** 进行映射，需要特定权限或配置。可配合 `MAP_HUGE_2MB`, `MAP_HUGE_1GB` (Linux 3.8+) 或 `MAP_HUGE_SHIFT` 指定大页尺寸。
        *   `MAP_LOCKED` (Linux 2.5.37+): 尝试将映射区域锁定在物理内存中 (类似 `mlock(2)`)，尝试预读页面，但不保证成功（可能后续发生缺页中断）。比 `mlock(2)` 语义弱。
        *   `MAP_NORESERVE`: **不为此映射预留交换空间**。写入时若无可用物理内存，可能收到 `SIGSEGV` 信号。默认行为受 `/proc/sys/vm/overcommit_memory` 控制。
        *   `MAP_POPULATE` (Linux 2.5.46+): **预读页表**。对于文件映射，会触发文件预读。有助于减少后续访问时的缺页阻塞。与私有映射结合始于 Linux 2.6.23。
        *   `MAP_STACK` (Linux 2.6.27+): 在适合做栈的地址分配映射（目前在 Linux 上是空操作，主要为可移植性和未来兼容性）。
        *   `MAP_SYNC` (Linux 4.15+): 仅用于 `MAP_SHARED_VALIDATE` 类型和支持 DAX 的文件。保证写入的内存即使系统崩溃也能持久化到文件中（配合 CPU 指令）。
        *   `MAP_SHARED_VALIDATE` (Linux 4.15+): 类似 `MAP_SHARED`，但会**校验所有 `flags`**，遇到未知标志则失败 (EOPNOTSUPP)。某些标志 (如 `MAP_SYNC`) 必须与此类型配合使用。
        *   `MAP_UNINITIALIZED` (Linux 2.6.33+): 不清零匿名页面（提高嵌入式设备性能），需内核配置 `CONFIG_MMAP_ALLOW_UNINITIALIZED`（有安全风险）。
        *   其他 (`MAP_DENYWRITE`, `MAP_EXECUTABLE`, `MAP_FILE`): 已废弃或被忽略。
*   **`fd` (文件描述符):**
    *   对于文件映射，指定要映射的文件。
    *   对于匿名映射 (`MAP_ANONYMOUS`)，此参数被忽略，应设为 -1。
    *   `mmap` 调用返回后，即使立即 `close(fd)`，映射依然有效。映射关系与文件描述符生命周期无关，而是与底层的打开文件描述相关。
*   **`offset` (文件偏移):**
    *   指定文件映射的起始**偏移量**。
    *   **必须是页大小的整数倍** (`sysconf(_SC_PAGE_SIZE)`)。

### `munmap()`

`munmap()` 删除指定地址范围 (`[addr, addr + length)`) 的映射。

*   **`addr`:** 要取消映射区域的起始地址，**必须是页大小的整数倍**。
*   **`length`:** 要取消映射区域的长度（字节数）。不必是页大小的整数倍，内核会自动处理包含在该范围内的所有页面。
*   **效果:** 解除映射后，访问该范围内的地址将产生无效内存引用（通常是 `SIGSEGV` 信号）。
*   **自动解除:** 进程终止时，所有映射会自动解除。
*   **`close(fd)` 的影响:** 关闭创建映射时使用的文件描述符**不会**解除映射。
*   **部分解除:** 如果解除的范围位于一个较大映射的中间，会导致原映射分裂成两个较小的映射。
*   **错误:** 如果指定的范围没有包含任何映射页面，`munmap` 不会报错。

## 5. 返回值 (RETURN VALUE)
*   **`mmap()`:**
    *   成功时: 返回指向映射区域的**指针**。
    *   失败时: 返回 `MAP_FAILED` (即 `(void *) -1`)，并设置 `errno`。
*   **`munmap()`:**
    *   成功时: 返回 **0**。
    *   失败时: 返回 **-1**，并设置 `errno` (通常是 `EINVAL`)。

## 6. 错误 (ERRORS)

`mmap` 可能因多种原因失败，常见的 `errno` 值包括：
*   `EACCES`: 文件描述符 `fd` 非普通文件；文件未以读模式打开；请求 `MAP_SHARED` 和 `PROT_WRITE` 但 `fd` 不是读写模式打开；`PROT_WRITE` 但文件是只追加模式。
*   `EAGAIN`: 文件被锁定，或锁定的内存过多 (`setrlimit(2)`)。
*   `EBADF`: `fd` 不是有效的文件描述符 (且未设置 `MAP_ANONYMOUS`)。
*   `EEXIST`: 指定了 `MAP_FIXED_NOREPLACE`，但请求范围与现有映射冲突。
*   `EINVAL`: `addr`, `length`, 或 `offset` 无效（如太大、非页对齐）；`length` 为 0 (Linux 2.6.12+)；`flags` 无效 (未包含 `MAP_PRIVATE` 或 `MAP_SHARED`/`MAP_SHARED_VALIDATE`)。
*   `ENFILE`: 系统范围内的打开文件总数达到上限。
*   `ENODEV`: `fd` 指向的文件所在的文件系统不支持内存映射。
*   `ENOMEM`: 可用内存不足；进程映射数量达到上限；`munmap` 中间区域导致分裂后映射数超限；超出 `RLIMIT_DATA` 限制 (Linux 4.7+)。
*   `EOVERFLOW`: (32位 + LFS) `offset + length` 溢出 `unsigned long`。
*   `EPERM`: 请求 `PROT_EXEC` 但文件系统以 `noexec` 挂载；操作被文件密封 (file seal) 阻止 (`fcntl(2)`)；使用 `MAP_HUGETLB` 但无权限 (`CAP_IPC_LOCK` 或不在 `sysctl_hugetlb_shm_group` 组)。
*   `ETXTBSY`: (`MAP_DENYWRITE` 标志，已废弃) `fd` 指定的对象正以写入模式打开。

使用映射区域可能导致以下信号：
*   `SIGSEGV`: 尝试写入只读映射区域。
*   `SIGBUS`: 尝试访问映射文件末尾之外的缓冲区页面。

## 7. 属性 (ATTRIBUTES)
*   **线程安全:** `mmap()`, `munmap()` 都是线程安全的 (MT-Safe)。

## 8. 版本 (VERSIONS)
*   可移植的创建映射方式是指定 `addr` 为 `NULL`，并且 `flags` 中不包含 `MAP_FIXED`。
*   许多 Linux 特有的 `flags` (如 `MAP_ANONYMOUS`, `MAP_HUGETLB` 等) 需要定义特性测试宏才能使用 (如 `_DEFAULT_SOURCE` 或 `_GNU_SOURCE`)。

## 9. C 库/内核差异 (C library/kernel differences)
glibc 的 `mmap()` 包装函数现在内部调用的是 `mmap2(2)` 系统调用（为了支持大文件偏移量）。

## 10. 标准 (STANDARDS)
POSIX.1-2008 (包含 `mmap`, `munmap`)。

## 11. 历史 (HISTORY)
POSIX.1-2001, SVr4, 4.4BSD.

## 12. 注意事项 (NOTES)
*   **`fork()` 与映射:** `mmap()` 创建的内存映射在 `fork(2)` 后会被子进程**继承**，并保持相同的属性。**C++ Core Guidelines:** 这意味着父子进程可能共享映射（如果是 `MAP_SHARED`），需要注意并发访问和资源管理。
*   **文件末尾部分页:** 文件大小不是页大小整数倍时，映射末尾的部分页会被零填充，对这部分的修改不会写回文件。文件大小改变对映射的影响未指定。
*   **`MAP_FIXED` 的安全使用:** **极其危险**，只应用于先前已保留的地址范围。否则，多线程程序可能因覆盖其他线程/库创建的映射而损坏地址空间。Linux 4.17+ 的 `MAP_FIXED_NOREPLACE` 提供更安全的选择。
*   **时间戳:** 文件映射 (`MAP_SHARED` + `PROT_WRITE`) 会影响文件的 `st_atime`, `st_ctime`, `st_mtime`，具体更新时机与访问和 `msync(2)` 调用有关。
*   **大页映射:** `mmap` 和 `munmap` 的 `offset`, `addr`, `length` 参数需要满足大页尺寸的对齐要求。
*   **Linux 默认行为:** 默认情况下，系统内存不足时任何进程都可能被 OOM killer 杀死。`MAP_NORESERVE` 只是不预留交换空间，不能保证进程不会被杀。

## 13. C++ 示例 (使用 mmap 读取文件)

这个 C++ 示例演示了如何使用 `mmap` 将一个文件映射到内存，并读取其内容，最后使用 `munmap` 解除映射。

```cpp
#include <iostream>
#include <fcntl.h>     // open, O_RDONLY
#include <unistd.h>    // close, sysconf, _SC_PAGE_SIZE
#include <sys/mman.h>  // mmap, munmap, PROT_READ, MAP_PRIVATE
#include <sys/stat.h>  // fstat, struct stat
#include <cstdio>      // perror
#include <cstdlib>     // exit, EXIT_FAILURE, EXIT_SUCCESS
#include <cstring>     // strerror
#include <stdexcept>   // std::runtime_error
#include <string>
#include <memory>      // std::unique_ptr (for RAII file descriptor)

// 简单的 RAII 文件描述符包装器
class FileDescriptor {
    int fd_ = -1;
public:
    FileDescriptor() = default;
    FileDescriptor(const char* pathname, int flags) {
        fd_ = open(pathname, flags);
        if (fd_ == -1) {
            throw std::runtime_error(std::string("无法打开文件 '") + pathname + "': " + strerror(errno));
        }
    }
    ~FileDescriptor() {
        if (fd_ != -1) {
            if (close(fd_) == -1) {
                perror("析构时关闭文件描述符失败");
            }
        }
    }
    FileDescriptor(FileDescriptor&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }
    FileDescriptor& operator=(FileDescriptor&& other) noexcept {
        if (this != &other) {
            if (fd_ != -1) close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }
    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;
    int get() const { return fd_; }
};

int main(int argc, char *argv[]) {
    if (argc != 2) {
        std::cerr << "用法: " << argv[0] << " <文件名>\n";
        return EXIT_FAILURE;
    }

    const char* filename = argv[1];
    void* mapped_mem = MAP_FAILED; // 初始化为失败状态
    size_t file_size = 0;
    FileDescriptor fd; // RAII 包装器

    try {
        // 1. 打开文件 (RAII)
        fd = FileDescriptor(filename, O_RDONLY);

        // 2. 获取文件大小
        struct stat sb;
        if (fstat(fd.get(), &sb) == -1) {
            throw std::runtime_error(std::string("fstat 失败: ") + strerror(errno));
        }
        file_size = sb.st_size;

        if (file_size == 0) {
             std::cout << "文件 '" << filename << "' 为空。\n";
             return EXIT_SUCCESS; // 空文件，无需映射
        }


        // 3. 映射文件到内存
        // addr = NULL (内核选择地址), length = file_size, prot = 只读,
        // flags = 私有映射, fd = 文件描述符, offset = 0 (从文件头开始)
        mapped_mem = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd.get(), 0);

        if (mapped_mem == MAP_FAILED) {
            throw std::runtime_error(std::string("mmap 失败: ") + strerror(errno));
        }

        // 4. 像访问内存一样访问文件内容
        std::cout << "文件 '" << filename << "' (大小: " << file_size << " 字节) 映射到地址: " << mapped_mem << std::endl;
        std::cout << "文件内容:\n-------\n";

        // 将映射的内存视为 char 数组
        const char* file_content = static_cast<const char*>(mapped_mem);

        // 打印前 100 个字节或整个文件（如果小于 100 字节）
        size_t print_len = (file_size < 100) ? file_size : 100;
        for (size_t i = 0; i < print_len; ++i) {
            std::cout << file_content[i];
        }
        if (file_size > 100) {
            std::cout << "\n[...内容过长，仅显示前 100 字节...]";
        }
        std::cout << "\n-------\n";

        // 5. 解除映射 (必须！)
        if (munmap(mapped_mem, file_size) == -1) {
            // munmap 失败通常问题不大，但应该记录
            perror("munmap 失败");
            // 即使 munmap 失败，也继续尝试关闭 fd (通过 RAII)
        } else {
             std::cout << "内存映射已解除。\n";
             mapped_mem = MAP_FAILED; // 标记为无效
        }

        // 文件描述符 fd 会在 FileDescriptor 对象离开作用域时自动关闭

    } catch (const std::exception& e) {
        std::cerr << "发生错误: " << e.what() << std::endl;
        // 如果映射成功但后续出错，需要确保解除映射
        if (mapped_mem != MAP_FAILED) {
            if (munmap(mapped_mem, file_size) == -1) {
                 perror("异常处理中 munmap 失败");
            }
        }
        // 文件描述符由 RAII 处理
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

**编译与运行:**

```bash
g++ your_mmap_program.cpp -o your_mmap_program
./your_mmap_program some_existing_file.txt
```

**结果:**

程序会打开指定的文本文件，将其内容映射到内存，然后打印文件的前 100 个字节（或全部内容，如果文件较小）。最后，它会解除内存映射并关闭文件。这个例子展示了 `mmap` 的基本流程，并使用了 RAII 来管理文件描述符，但**注意 `munmap` 仍需手动调用**，因为 `mmap` 返回的是原始指针，标准智能指针不直接适用（需要自定义删除器）。

## 14. 参见 (SEE ALSO)
`ftruncate(2)`, `getpagesize(2)`, `memfd_create(2)`, `mincore(2)`, `mlock(2)`, `mmap2(2)`, `mprotect(2)`, `mremap(2)`, `msync(2)`, `remap_file_pages(2)`, `setrlimit(2)`, `shmat(2)`, `userfaultfd(2)`, `shm_open(3)`, `shm_overview(7)`, `proc(5)` (特别是 `/proc/<pid>/maps`, `/proc/<pid>/map_files`, `/proc/<pid>/smaps`)。

---
