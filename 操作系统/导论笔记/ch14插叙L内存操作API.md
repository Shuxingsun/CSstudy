

# 插叙：内存操作`API`

### 关键问题：如何分配和管理内存

## 内存类型

- 栈内存
  - 它的申请和释放操作是编译器来隐式管理的，所以有时也称为自动（automatic）内存
- 堆内存(heap):
  - 长期内存需求，所有申请和释放有程序员显式完成

## `malloc`()调用

- `maclloc`():**传入要申请的堆空间的大小，它成功就返回一个指向新申请空间的指针，失败就返回 NULL

```c
#include <stdlib.h> 
... 
void *malloc(size_t size);
```

`malloc` 只需要一个 size_t 类型参数，该参数表示你需要多少个字.

- 函数和宏：

大多数程序员并不会直接传入数字（比如 10）。实实上，这样做会被但为是不太好的形式。替代方案
是使用**各种函数和宏**,例如：`sizeof`()

```c
int *x = malloc(10 * sizeof(int)); 
printf("%d\n", sizeof(x));
```

- `string`

如果为一个字符串声明空间，请使用以下习惯用法：`malloc(strlen(s) + 1)`，**它使用函数 `strlen()`获取字符串的长度，并加上 1，以便为字符串结束符留出空间**。这里使用 ~可能会导致麻烦

## `free`()调用

该函数接受一个参数，即一个由 `malloc()`返回的指针

```c
int *x = malloc(10 * sizeof(int)); 
... 
free(x);
```

## 常见错误

- 忘记分配内存

  ```c
  char *src = "hello"; 
  char *dst; // oops! unallocated 
  strcpy(dst, src); // segfault and die
  ```

- 没有分配足够内存
- 忘记初始化分配的内存
- 忘记释放内存
- 在用完之前释放内存
- 反复释放内存
- 错误调用free()

## 底层操作系统支持

- 库调用
  - `malloc()`和 free()时，我们没有讨论系统调用。原因很简单：它们不是系统调用，而是库调用
  - `calloc`()分配内存

- 系统调用
  - `brk`:改变程序分断(break)位置

## 调试C程序

- `gdb`

  ```shell
  gcc a.c -o a 
  gdb a
  run
  ```

  输出信息

  <pre>
  Type "apropos word" to search for commands related to "word"...
  Reading symbols from a.out...

  Starting program: Operating-Systems-Three-Easy-Pieces-NOTES/第十四章/代码/a.out 
  [Inferior 1 (process 14420) exited normally]

- `valgrind`

  ```shell
  valgrind --leak-check=yes ./(a.out)a
  ```

  输出

  <pre>
  ==15225== Memcheck, a memory error detector
  ==15225== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
  ==15225== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
  ==15225== Command: ./a.out
  ==15225== 
  ==15225== 
  ==15225== HEAP SUMMARY:
  ==15225==     in use at exit: 0 bytes in 0 blocks
  ==15225==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
  ==15225== 
  ==15225== All heap blocks were freed -- no leaks are possible
  ==15225== 
  ==15225== For lists of detected and suppressed errors, rerun with: -s
  ==15225== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
  </pre>

  信息:
  堆使用情况: 分配 0 次, 释放 0 次, 分配 0 字节.

- 区别：

  - `gdb`指出错误，`valgrind`明确指出错误在哪



