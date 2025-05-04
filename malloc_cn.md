# malloc(3) 函数详解

> 本文档基于 glibc 的 `malloc`、`free`、`calloc`、`realloc` 和 `reallocarray` 函数，详细说明它们的用法、返回值、错误处理及示例。

---

## 目录

1. [函数概述](#函数概述)
2. [函数原型](#函数原型)
3. [参数说明](#参数说明)
4. [详细说明](#详细说明)
   - [malloc](#malloc)
   - [free](#free)
   - [calloc](#calloc)
   - [realloc](#realloc)
   - [reallocarray](#reallocarray)
5. [返回值](#返回值)
6. [错误处理](#错误处理)
7. [线程安全与实现细节](#线程安全与实现细节)
8. [标准与历史](#标准与历史)
9. [示例代码](#示例代码)
10. [相关函数](#相关函数)

---

## 函数概述

这些函数负责在运行时动态分配和释放内存。

- `malloc`：分配指定字节数的未初始化内存。
- `free`：释放先前分配的内存。
- `calloc`：分配并清零内存。
- `realloc`：调整已有内存块的大小。
- `reallocarray`：安全调整数组所需的内存大小，防止乘法溢出。

---

## 函数原型

```c
#include <stdlib.h>

void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t n, size_t size);
void *realloc(void *ptr, size_t size);
void *reallocarray(void *ptr, size_t n, size_t size);
```

---

## 参数说明

- `size`：要分配的字节数。
- `n`：元素个数。
- `ptr`：指向已分配内存的指针。

---

## 详细说明

### malloc

- 功能：分配 `size` 字节的内存，返回一个指向未初始化区域的指针。
- 特殊情况：当 `size == 0` 时，会返回一个可传递给 `free` 的非 `NULL` 唯一指针。

### free

- 功能：释放 `ptr` 指向的内存空间，该 `ptr` 必须由 `malloc`、`calloc`、`realloc` 等函数返回。
- 特殊情况：如果 `ptr == NULL`，不执行任何操作。
- 未定义行为：重复释放同一指针或释放非分配指针将导致未定义行为。

### calloc

- 功能：分配 `n` 个大小为 `size` 字节的元素，并将所有字节初始化为 0。
- 特殊情况：当 `n == 0` 或 `size == 0` 时，返回一个可安全调用 `free` 的唯一指针。
- 安全检查：若 `n * size` 溢出，会检测并返回错误，而不是错误分配。

### realloc

- 功能：调整 `ptr` 指向的内存块大小为 `size`，并保留原有内容的前部。
- 行为：
  - 如果 `ptr == NULL`，等同于 `malloc(size)`。
  - 如果 `size == 0` 且 `ptr != NULL`，等同于 `free(ptr)`（glibc 会保留 `errno`）。
  - 扩大时，新内存不初始化；缩小时，超出部分丢弃。
  - 如果需要搬迁，旧块被释放并返回新地址。

### reallocarray

- 功能：等价于 `realloc(ptr, n * size)`，但会在乘法溢出时安全失败并返回 `NULL`。

---

## 返回值

- 成功时：返回分配内存的指针，已对齐到满足任何类型边界。
- 失败时：返回 `NULL` 并设置 `errno`。
- 对于 `realloc` 与 `reallocarray`：若 `ptr != NULL` 且 `size == 0`，返回 `NULL`，但不视为错误，原内存保持不变。

---

## 错误处理

- 内存不足或超出资源限制时，分配函数返回 `NULL` 并将 `errno` 设为 `ENOMEM`。
- 常见原因：
  - 达到 `RLIMIT_AS` 或 `RLIMIT_DATA` 限制。
  - 映射数量超过 `/proc/sys/vm/max_map_count` 限制。

---

## 线程安全与实现细节

- 这些函数在多线程环境下使用内部互斥锁保护数据结构，保证 MT-Safe。
- 当检测到锁竞争，glibc 会创建多个 arena 以减少分配时的互斥开销。
- 默认使用 `brk(2)` 扩展堆；当分配大小超过 `MMAP_THRESHOLD`（默认为 128 KB）时，采用 `mmap(2)` 映射。
- glibc 2.30 及以上版本对大于 `PTRDIFF_MAX` 的分配请求直接返回错误。

---

## 标准与历史

| 函数            | 标准               | 引入版本                         |
|---------------|------------------|------------------------------|
| malloc, free, calloc, realloc      | C11, POSIX.1-2008   | C89 / POSIX.1-2001            |
| reallocarray   | 无标准            | glibc 2.26, OpenBSD 5.6 / FreeBSD 11 |

---

## 示例代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MALLOCARRAY(n, type) ((type *) reallocarray(NULL, n, sizeof(type)))

int main(void) {
    char *p = MALLOCARRAY(32, char);
    if (p == NULL) {
        perror("reallocarray");
        return EXIT_FAILURE;
    }
    strncpy(p, "Hello, malloc!", 32);
    puts(p);
    free(p);
    return EXIT_SUCCESS;
}
```

---

## 相关函数

- `valgrind(1)`：内存检测工具。
- `brk(2)、mmap(2)`：底层系统调用。
- `mallopt(3)、malloc_trim(3)、malloc_info(3)`：内存分配调优与监控。
- `malloc_usable_size(3)`：获取分配块大小。

---

*本文档基于 Linux man-pages 6.13（2024-11-17）及 glibc 官方实现说明撰写.*
