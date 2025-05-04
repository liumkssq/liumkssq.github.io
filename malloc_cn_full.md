# malloc(3) C 风格动态内存管理详解 (结合 C++ 实践原则)

本文档基于 Linux man-pages 6.13 (2024-11-17) 对 C 标准库中的动态内存管理函数 `malloc`, `free`, `calloc`, `realloc`, 和 `reallocarray` 进行详细的中文解释。同时，将结合 C++ Core Guidelines [https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines] 的现代 C++ 实践原则（特别是资源管理章节 R）以及 C++ Reference [https://en.cppreference.com/w/c/memory] [https://en.cppreference.com/w/cpp/memory/c] 的定义进行探讨，并警示如 Stack Overflow 帖子 [https://stackoverflow.com/questions/40339840/funny-write-syscall-error-prints-chinese-characters-in-txt] 中提到的内存使用不当风险。

**重要提示 (C++ Core Guidelines R.10 & R.11):** 在现代 C++ 编程中，**强烈建议避免直接使用 `malloc()`, `free()`, `calloc()`, `realloc()`** 以及 C++ 的 `new` 和 `delete`。应优先使用 RAII (Resource Acquisition Is Initialization) 原则，通过标准库容器（如 `std::vector`, `std::string`）或智能指针（如 `std::unique_ptr`, `std::shared_ptr`）来自动管理内存生命周期。理解这些 C 函数对于维护旧代码或与 C 库交互仍然重要。

---

## 1. 名称 (NAME)
`malloc`, `free`, `calloc`, `realloc`, `reallocarray` - 分配和释放动态内存

## 2. 所属库 (LIBRARY)
标准 C 库 (`libc`, 编译时链接 `-lc`)。在 C++ 中，这些函数通过 `<cstdlib>` 头文件提供 [https://en.cppreference.com/w/cpp/header/cstdlib]。

## 3. 函数原型 (SYNOPSIS)
```c
#include <stdlib.h> // C
// #include <cstdlib> // C++

void *malloc(size_t size);
void free(void *ptr); // _Nullable 注解提示 ptr 可以是 NULL
void *calloc(size_t n, size_t size);
void *realloc(void *ptr, size_t size); // _Nullable 注解提示 ptr 可以是 NULL
void *reallocarray(void *ptr, size_t n, size_t size); // _Nullable 注解提示 ptr 可以是 NULL

// Feature Test Macro Requirements for glibc (例如 reallocarray):
// glibc >= 2.29: _DEFAULT_SOURCE
// glibc <= 2.28: _GNU_SOURCE
```

## 4. 描述 (DESCRIPTION)

### `malloc(size_t size)`
*   **功能:** 分配 `size` 字节的**未初始化**存储空间 [https://en.cppreference.com/w/c/memory/malloc]。
*   **返回值:** 如果分配成功，返回一个指向分配内存的指针。该指针保证对任何具有基本对齐要求（fundamental alignment）的对象类型都是正确对齐的。
*   **`size == 0`:** 如果 `size` 为 0，行为是实现定义的。可能返回 `NULL`，也可能返回一个有效的、非 `NULL` 的指针。这个非 `NULL` 指针**不应被解引用**，但应传递给 `free()` 以避免内存泄漏。
*   **注意:** 返回的内存内容是**未定义**的（垃圾值）。

### `free(void *ptr)`
*   **功能:** 释放由 `malloc()`, `calloc()`, `realloc()`, `reallocarray()` 或 `aligned_alloc()` 先前分配的内存空间 [https://en.cppreference.com/w/cpp/memory/c/free]。
*   **`ptr == NULL`:** 如果 `ptr` 是 `NULL`，`free()` 不执行任何操作。
*   **未定义行为 (UB):**
    *   如果 `ptr` 指向的内存不是通过上述分配函数获得的。
    *   如果 `ptr` 指向的内存已经被 `free()` 或 `realloc()` 释放。
    *   释放后继续使用该指针（悬挂指针）。
*   **`errno`:** `free()` 不返回值，并且（自 glibc 2.33 起）保留 `errno` 的值。

### `calloc(size_t n, size_t size)`
*   **功能:** 为 `n` 个元素的数组分配内存，每个元素大小为 `size` 字节。与 `malloc` 不同，`calloc` 会将分配的内存**初始化为全零** [https://en.cppreference.com/w/c/memory/calloc]。
*   **`n == 0` 或 `size == 0`:** 如果 `n` 或 `size` 为 0，`calloc` 返回一个可以成功传递给 `free` 的唯一指针值（可能是 `NULL` 或非 `NULL`，取决于实现）。
*   **整数溢出:** 如果 `n * size` 的乘积会导致整数溢出，`calloc()` 会检测到并返回错误（通常是 `NULL`）。这比直接调用 `malloc(n * size)` 更安全，后者不会检测溢出，可能导致分配大小不正确的内存块。

### `realloc(void *ptr, size_t size)`
*   **功能:** 改变 `ptr` 指向的内存块的大小为 `size` 字节。
*   **行为:**
    *   内容保留: 从内存区域开始到新旧大小中较小者的那部分内容将保持不变。
    *   扩大内存: 如果新大小 `size` 大于旧大小，增加的内存部分是**未初始化**的。
    *   缩小内存: 如果新大小 `size` 小于旧大小，多余的部分会被丢弃。
    *   `ptr == NULL`: 调用等价于 `malloc(size)`。
    *   `size == 0` 且 `ptr != NULL`: 调用等价于 `free(ptr)`，并返回 `NULL` (非错误情况，见返回值)。
    *   内存移动: 如果无法在原地调整大小，`realloc` 会分配一个新的内存块，将旧内存块的内容（最多到新旧大小的最小值）复制到新块，然后**释放旧内存块 `ptr`**，最后返回新内存块的指针。如果调整大小成功但内存块**没有移动**，则返回原始的 `ptr`。
*   **前提:** 除非 `ptr` 是 `NULL`，否则它必须是由先前调用 `malloc` 系列函数返回的指针。
*   **失败:** 如果 `realloc` 失败（例如内存不足），它会返回 `NULL`，并且**原始内存块 `ptr` 保持不变**（不会被释放或移动）。

### `reallocarray(void *ptr, size_t n, size_t size)`
*   **功能:** 改变 `ptr` 指向的内存块大小，使其足够容纳 `n` 个元素的数组，每个元素大小为 `size` 字节。
*   **等价于:** `realloc(ptr, n * size)`。
*   **优势:** 与 `realloc` 不同，`reallocarray` 会在 `n * size` 乘法发生**整数溢出**时**安全地失败**（返回 `NULL` 并设置 `errno`），而不是尝试用错误的大小去调用 `realloc`。

## 5. 返回值 (RETURN VALUE)
*   **`malloc()`, `calloc()`, `realloc()`, `reallocarray()`:**
    *   成功时: 返回指向分配内存的指针，该指针已适当对齐。
    *   失败时: 返回 `NULL`，并设置 `errno`。尝试分配超过 `PTRDIFF_MAX` 字节也被视为错误。
*   **`free()`:** 无返回值。
*   **`realloc()`, `reallocarray()` 特殊情况:** 如果 `ptr` 非 `NULL` 且请求的 `size` (或 `n*size`) 为 0，函数返回 `NULL`。这**不被视为错误**，原始 `ptr` 实际上已被释放 (效果等同于 `free(ptr)`)。

## 6. 错误 (ERRORS)
`malloc`, `calloc`, `realloc`, `reallocarray` 可能因以下错误而失败：
*   `ENOMEM`: 内存不足。可能是程序达到了 `RLIMIT_AS` 或 `RLIMIT_DATA` 资源限制，或者内核映射数量 (`/proc/sys/vm/max_map_count`) 达到上限。

## 7. 属性 (ATTRIBUTES)
*   **线程安全:** 这些函数在 glibc 中是线程安全的 (MT-Safe)，内部使用互斥锁保护共享数据结构。在高并发下可能使用多 arena 机制减少锁竞争。

## 8. 标准 (STANDARDS)
*   `malloc()`, `free()`, `calloc()`, `realloc()`: C11, POSIX.1-2008.
*   `reallocarray()`: 非标准，glibc 2.26, OpenBSD 5.6, FreeBSD 11.0 等提供。

## 9. 历史 (HISTORY)
*   `malloc()`, `free()`, `calloc()`, `realloc()`: POSIX.1-2001, C89.
*   glibc 2.30 起拒绝大于 `PTRDIFF_MAX` 的分配。
*   glibc 2.33 起 `free()` 保留 `errno`。

## 10. 注意事项 (NOTES)
*   **Linux 乐观内存分配:** `malloc` 返回非 `NULL` 不保证内存真的可用。系统内存不足时，OOM killer 可能会终止进程。
*   **底层实现:** glibc 的 `malloc` 通常从堆 (heap) 分配，必要时使用 `sbrk(2)` 调整堆大小。对于大于 `MMAP_THRESHOLD` (默认 128KB) 的大块内存，会使用 `mmap(2)` 进行私有匿名映射。
*   **私有分配器:** 如果替换标准分配器，必须确保兼容 glibc 的行为（包括 `errno` 处理、零大小分配、溢出检查），否则可能导致库函数崩溃。C++ Core Guidelines R.10 反对这样做。
*   **堆损坏:** 内存分配器的崩溃几乎总是与堆损坏有关，如缓冲区溢出或重复释放。使用工具 (如 Valgrind) 检测此类问题至关重要。
*   **Stack Overflow 警示:** 正如帖子 [https://stackoverflow.com/questions/40339840/funny-write-syscall-error-prints-chinese-characters-in-txt] 中用户遇到的问题，即使分配成功，如果向分配的缓冲区写入的数据超出了实际分配的大小，也会导致未定义行为 (UB)，可能出现看似随机的错误（如乱码）。**必须**确保写入操作不超过分配的边界。
*   **非可移植行为:**
    *   零大小分配的行为在不同实现中可能不同，POSIX 程序应能容忍返回 `NULL` 且不设置 `errno` 的情况。
    *   虽然 POSIX 要求分配失败时设置 `errno`，但 C 标准不要求，可移植性要求不应假定这一点。

## 11. C++ 示例 (对比 C 风格与现代 C++)

```cpp
#include <iostream>
#include <vector>
#include <memory> // for std::unique_ptr
#include <cstdlib> // for malloc, free, calloc
#include <cstdio> // for perror
#include <cstring> // for memset (demonstration only)
#include <new> // for std::nothrow

int main() {
    // --- C 风格内存分配 (不推荐在现代 C++ 中使用) ---
    std::cout << "--- C-style allocation (discouraged) ---\n";
    int* c_array = nullptr;
    size_t num_elements = 5;

    // 1. 使用 malloc 分配 (未初始化)
    c_array = (int*)malloc(num_elements * sizeof(int));
    if (c_array == nullptr) {
        perror("malloc failed");
        // 处理错误...
    } else {
        std::cout << "malloc allocated memory (content is undefined):\n";
        // 必须手动初始化或填充
        for (size_t i = 0; i < num_elements; ++i) {
            c_array[i] = static_cast<int>(i * 10); // 填充值
            std::cout << "c_array[" << i << "] = " << c_array[i] << std::endl;
        }
        // 必须手动释放
        free(c_array);
        c_array = nullptr; // 好习惯：释放后置空指针
        std::cout << "Memory freed using free().\n\n";
    }

    // 2. 使用 calloc 分配 (初始化为 0)
    c_array = (int*)calloc(num_elements, sizeof(int));
    if (c_array == nullptr) {
        perror("calloc failed");
    } else {
        std::cout << "calloc allocated memory (zero-initialized):\n";
        for (size_t i = 0; i < num_elements; ++i) {
            std::cout << "c_array[" << i << "] = " << c_array[i] << std::endl;
        }
        free(c_array);
        c_array = nullptr;
        std::cout << "Memory freed using free().\n";
    }

    std::cout << "\n--- Modern C++ style allocation (recommended) ---\n";

    // --- 现代 C++ 风格 ---

    // 1. 使用 std::vector (推荐用于动态数组)
    std::cout << "Using std::vector:\n";
    try {
        std::vector<int> cpp_vector(num_elements); // 自动分配内存，默认初始化为 0
        std::cout << "std::vector allocated and zero-initialized:\n";
        for (size_t i = 0; i < cpp_vector.size(); ++i) {
            cpp_vector[i] = static_cast<int>(i * 100); // 填充值
            std::cout << "cpp_vector[" << i << "] = " << cpp_vector[i] << std::endl;
        }
        // 内存会在 cpp_vector 离开作用域时自动释放 (RAII)
        std::cout << "Memory managed automatically by std::vector.\n\n";
    } catch (const std::bad_alloc& e) {
        std::cerr << "std::vector allocation failed: " << e.what() << '\n';
    }


    // 2. 使用 std::unique_ptr 管理动态数组 (如果必须手动管理)
    std::cout << "Using std::unique_ptr for an array:\n";
    // 使用 std::make_unique (C++14+) 或 new (如果需要 nothrow 或自定义删除器)
    // make_unique 会进行值初始化（对于 int 是零初始化）
    try {
        // C++14 及以后推荐
        auto cpp_uptr_array = std::make_unique<int[]>(num_elements);

        // 或者使用 new (如果需要 nothrow)
        // int* raw_ptr = new(std::nothrow) int[num_elements](); // () 进行零初始化
        // if (!raw_ptr) { /* 处理分配失败 */ }
        // std::unique_ptr<int[]> cpp_uptr_array(raw_ptr);

        std::cout << "std::unique_ptr allocated array (zero-initialized):\n";
         for (size_t i = 0; i < num_elements; ++i) {
             cpp_uptr_array[i] = static_cast<int>(i * 1000); // 填充值
             std::cout << "cpp_uptr_array[" << i << "] = " << cpp_uptr_array[i] << std::endl;
         }
        // 内存会在 cpp_uptr_array 离开作用域时自动释放 (RAII)
         std::cout << "Memory managed automatically by std::unique_ptr.\n";
    } catch (const std::bad_alloc& e) {
         std::cerr << "unique_ptr allocation failed: " << e.what() << '\n';
    }

    return 0;
}
```

**示例解释:**

*   **C 风格部分:** 展示了如何使用 `malloc`（需要手动初始化）和 `calloc`（自动零初始化），并强调了**必须手动调用 `free`** 来释放内存，以及分配失败时需要检查 `NULL` 返回值。
*   **现代 C++ 部分:**
    *   **`std::vector`:** 这是在 C++ 中处理动态大小数组的首选方式。它自动处理内存分配、初始化（默认零初始化或可指定初始值）和释放。使用起来更安全、更方便。
    *   **`std::unique_ptr<int[]>`:** 如果确实需要更底层的数组内存管理（例如与 C API 交互），`unique_ptr` 是管理动态分配数组所有权的现代 C++ 方式。它也遵循 RAII 原则，在其生命周期结束时自动调用 `delete[]`。`std::make_unique<T[]>(size)` (C++14) 是创建它的推荐方式。

这个对比清晰地展示了为什么 C++ Core Guidelines 推荐使用现代 C++ 的资源管理机制，它们能显著减少内存泄漏和悬挂指针等常见错误。

## 12. 参见 (SEE ALSO)
`valgrind(1)`, `brk(2)`, `mmap(2)`, `alloca(3)`, `aligned_alloc(3)`, `malloc_get_state(3)`, `malloc_info(3)`, `malloc_trim(3)`, `malloc_usable_size(3)`, `mallopt(3)`, `mcheck(3)`, `mtrace(3)`, `posix_memalign(3)`。另请参阅 C++ 的 `<memory>` 和 `<vector>` 头文件。

---
