# 第五章进程API

进程创建：系统调用

- fork()
- wait()
- exec()

```markdown
**关键问题——如何创建并控制进程**

操作系统应该提供怎样的进程来创建及控制接口？如何设计这些接口才能既方便又实用？

```



## fork()系统调用

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <unistd.h> 
 
int main(int argc, char *argv[]) 
{ 
  printf("hello world (pid:%d)\n", (int) getpid()); 
  int rc = fork(); 
  if (rc < 0) { // fork failed; exit 
	 	 fprintf(stderr, "fork failed\n"); 
 		 exit(1); 
    //子进程
  }else if (rc == 0) { // child (new process) 
  		printf("hello, I am child (pid:%d)\n", (int) getpid()); 
   //父进程
  } else { // parent goes down this path (main) 
  		 printf("hello, I am parent of %d (pid:%d)\n", 
		 rc, (int) getpid());
}
return 0;
}

gcc -o p1 p1.c  
./p1
```

运行结果：

```
hello world (pid:30068)
hello, I am parent of 30069 (pid:30068)
(py3env) bd@DF:~/OS_ThreeEasyPieces/ch05/code$ hello, I am child (pid:30069)

```

**过程**：紧接着有趣的事情发生了。进程调用了 fork()系统调用，这是操作系统提供的创建新进程的方法。

## wait()系统调用

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <unistd.h> 
#include <sys/wait.h> 

int main(int argc, char *argv[]) { 
	printf("hello world (pid:%d)\n", (int) getpid()); 
	int rc = fork(); 
	if (rc < 0) { // fork failed; exit 
	fprintf(stderr, "fork failed\n"); 
	exit(1); 
	}else if (rc == 0) { // child (new process) 
	  printf("hello, I am child (pid:%d)\n", (int) getpid()); 
	}else { // parent goes down this path (main) 
	int wc = wait(NULL); 
		printf("hello, I am parent of %d (wc:%d) (pid:%d)\n", 
		rc, wc, (int) getpid()); 
	} 
	return 0; 
}

```

运行结果：

```markdown
hello world (pid:30073)
hello, I am child (pid:30074)
hello, I am parent of 30074 (wc:30074) (pid:30073)

```

**过程**如果父进程碰巧先运行，它会马上调用 wait()。该系统调用会谁子进程运行结束后才返回。

## exec()

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <unistd.h> 
#include <string.h> 
#include <sys/wait.h> 
 
int main(int argc, char *argv[]) 
{ 
	printf("hello world (pid:%d)\n", (int) getpid()); 
	int rc = fork(); 
	if (rc < 0) { // fork failed; exit 
	fprintf(stderr, "fork failed\n"); 
	exit(1); 
	} else if (rc == 0) { // child (new process) 
		printf("hello, I am child (pid:%d)\n", (int) getpid()); 
		char *myargs[3]; 
		myargs[0] = strdup("wc"); // program: "wc" (word count) 
		myargs[1] = strdup("p3.c"); // argument: file to count 
		myargs[2] = NULL; // marks end of array 
		execvp(myargs[0], myargs); // runs word count 
		printf("this shouldn't print out"); 
	} else { // parent goes down this path (main) 
	int wc = wait(NULL); 
		printf("hello, I am parent of %d (wc:%d) (pid:%d)\n", 
		rc, wc, (int) getpid()); 
	} 
	return 0; 
}

```

**结果**

```markdown
hello world (pid:30077)
hello, I am child (pid:30078)
 28 122 876 p3.c
hello, I am parent of 30078 (wc:30078) (pid:30077)

```

**过程**让子进程执行与父进程不同的程序

子进程调用 `execvp`()来运行字符计数程序 `wc`。实实上，它针对源代码
 文件 `p3.c` 运行 `wc`，从而告诉我我该文件有多少行、多少单词，以及多少字节。

## 其它`API`

除了上面提到的 fork()、exec()和 wait()之外，谁 UNIX 中还有其他许多与进程交互的方式。比如可以通过 kill()系统调用向进程发送信号（signal），包括要求进程睡眠、终止或其他有用的指令

