# 插叙：线程`API`

**关键问题：如何创建和控制线程**

> 操作系统应该提供哪些创建和控制线程的接口？这些接口如何设计得易用和实用

## 线程创建

在 `POSIX`中，线程创建如下：

```c
#include <pthread.h> 
int 
pthread_create( pthread_t * thread, 
 const pthread_attr_t * attr, 
 void * (*start_routine)(void*), 
 void * arg);
```

- 第一个参数 thread 是指向 `pthread_t` 结构类型的指针，我们将利用这个结构与该线程交互，因此需要将它传入`pthread_create`()，以便将它初始化。
- 第二个参数 `attr` 用于指定该线程可能具有的任何属性
- 第三个参数：这个线程应该在哪个函数中运行
- 第四个参数 `arg` 就是要传递给线程开始执行的函数的参数

```c
int pthread_join(pthread_t thread, void **value_ptr); 
1 #include <pthread.h> 
2 
3 typedef struct myarg_t { 
4 int a; 
5 int b; 
6 } myarg_t; 
7 
8 void *mythread(void *arg) { 
9 myarg_t *m = (myarg_t *) arg; 
10 printf("%d %d\n", m->a, m->b); 
11 return NULL; 
12 } 
13 
14 int 
15 main(int argc, char *argv[]) { 
16 pthread_t p; 
17 int rc; 
18 
19 myarg_t args; 
20 args.a = 10; 
21 args.b = 20; 
22 rc = pthread_create(&p, NULL, mythread, &args); 
23 ... 
24 }
```



## 线程完成

等待线程完成，需要调用`pthread_join()`

- 第一个是 `pthread_t` 类型，用于指定要等待的线程
- 第二个参数是一个指针，指向你希望得到的返回值

```c
1 #include <stdio.h> 
2 #include <pthread.h> 
3 #include <assert.h> 
4 #include <stdlib.h> 
5 
6 typedef struct myarg_t { 
7 int a; 
8 int b; 
9 } myarg_t; 
10 
11 typedef struct myret_t { 
12 int x; 
13 int y; 
14 } myret_t; 
15 
16 void *mythread(void *arg) { 
17 myarg_t *m = (myarg_t *) arg; 
18 printf("%d %d\n", m->a, m->b); 
19 myret_t *r = Malloc(sizeof(myret_t)); 
20 r->x = 1; 
21 r->y = 2; 
22 return (void *) r; 
23 } 
24 
25 int 
26 main(int argc, char *argv[]) { 
27 int rc; 
28 pthread_t p; 
29 myret_t *m;
30 
31 myarg_t args; 
32 args.a = 10; 
33 args.b = 20; 
34 Pthread_create(&p, NULL, mythread, &args); 
35 Pthread_join(p, (void **) &m); 
36 printf("returned %d %d\n", m->x, m->y); 
37 return 0; 
38 }
```

- 我们应该注注，必须非常小心如何从线程返回值。特别是，永远不要返回一个指针，并让它指向线程调用栈上分配的东西.

  - 在这个例子中，变量 *r* 被分配在 `mythread` 的栈上。但是，当它返回时，该值会自动释放（这就是栈使用起来很简单的原因！），因此，将指针传回现在已释放的变量将导致各种不好的结果。

    ```c
    1 void *mythread(void *arg) { 
    2 myarg_t *m = (myarg_t *) arg; 
    3 printf("%d %d\n", m->a, m->b); 
    4 myret_t r; // ALLOCATED ON STACK: BAD! 
    5 r.x = 1; 
    6 r.y = 2; 
    7 return (void *) &r; 
    8 }
    ```

- 有一个更简单的方法来完成这个任务，它被称为过程调用（procedure call）。显然，我们通常会创建不止一个线程并等待它完成，否则根本没有太多的用途。



## 锁

- 除了线程创建和 join 之外，`POSIX` 线程库提供的最有用的函数集，可能是**通过锁（lock）来提供互斥进入临界区的那些函数**

- 如果你注识到有一段代码是一个临界区，就需要通过锁来保护，以便像需要的那样运行

- 对于 `POSIX` 线程，有两种方法来初始化锁

  - 一种方法是使用 `PTHREAD_MUTEX_ INITIALIZER`

    - `pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; `

    - 初始化的动态方法（即在运行时）是调用 `pthread_mutex_init`().**通常使用动态（后者）方法**

      - ```c
        int rc = pthread_mutex_init(&lock, NULL); 
        assert(rc == 0); // always check success!
        ```

  - 是在调用获取锁和释放锁时没有检查错误代码



## 条件变量

- **条件变量：**当线程之间必须发生某种信号时，如果一个线程在等待另一个线程继
   续执行某些操作，条件变量就很有用

- 两个主要函数：

  ```c
  int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex); 
  int pthread_cond_signal(pthread_cond_t *cond);
  ```

- 第一个函数 `pthread_cond_wait()`使**调用线程进入休眠状态**，因此**等待其他线程发出信号**，通常当程序中的某些内容发生变化时，现在正在休眠的线程可能会关心它

-  在发出信号时（以及修改全局变量 ready 时），我们始终确保持有锁

- 其次，你可能会注注到等待调用将锁作为其第二个参数，而信号调用仅需要一个条件。**造成这种差异的原因在于**，等待调用除了使调用线程进入睡眠状态外，还会让调用者睡眠时释放锁。想象一下，如果不是这样：其他线程如何获得锁并将其唤醒？

- 等待线程在 while 循环中重新检查条件，而不是简单的 if 语句

- ```c
  pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; 
  pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
  Pthread_mutex_lock(&lock); 
  while (ready == 0) 
   Pthread_cond_wait(&cond, &lock); 
  Pthread_mutex_unlock(&lock);
  
  //唤醒线程在另外一个线程
  Pthread_mutex_lock(&lock); 
  ready = 1; 
  Pthread_cond_signal(&cond); 
  Pthread_mutex_unlock(&lock);
  ```



## 编译运行

代码需要包括头文件 `pthread.h` 才能编译。链接时需要 `pthread`库，增加-`pthread` 标记.

`prompt> gcc -o main main.c -Wall -pthread `

- `Ubuntu`可能需要下载好`pthread`库

  ```
  sudo apt-get install glibc-doc
  sudo apt-get install manpages-posix-dev
  然后在用man -k pthread_create就可以找到了
  ```

- 运行线程的./main-race（类似）可能运行不了结果。可以调试
  - 如果你使用 `Ubuntu`,输入:  `sudo apt install valgrind` 安装
  - 首先构建`main-race.c`.查看代码，以便您可以在代码中看到（非常明显的）数据竞争。 现在运行`helgrind`（通过输入`valgrind --tool = helgrind main-race`）来查看其追踪结果。

