
## 概念总览

- fork()
    - 作用：当前进程克隆出一个子进程（几乎一样的地址空间与执行上下文，采用写时复制 Copy-on-Write）。
    - 返回值：
        - 在父进程中返回子进程的 PID（>0）
        - 在子进程中返回 0
        - 失败返回 -1（不产生子进程）
- exec()
    - 作用：用一个新的可执行程序“替换”当前进程映像（代码段、数据段、堆栈等），PID 不变。
    - 成功不返回；失败返回 -1 并设置 errno。
- wait()
    - 作用：在父进程中“回收”已经退出的子进程，获得其退出状态，避免僵尸进程。
    - wait() 等待任意子进程退出；waitpid(pid, …) 可精确等待某个或一类子进程（支持 WNOHANG 等选项）。

## 三者之间的关系（典型流程）

1. 父进程调用 fork() → 得到一个子进程（几乎是当前进程的拷贝，采用 COW）
2. 子进程通常马上调用 exec() → 将自身替换为另一个可执行程序（开始执行新程序的 main）
3. 父进程随后调用 wait()/waitpid() → 阻塞或非阻塞地等待子进程结束并回收其退出状态

## 最小示例代码

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        // 子进程：用 ls -l 替换自身
        char *argv[] = { "ls", "-l", NULL };
        execvp(argv[0], argv);  // 成功不返回
        perror("execvp");       // 只有失败才会到这
        _exit(127);
    } else {
        // 父进程：等待子进程退出
        int status = 0;
        pid_t w = waitpid(pid, &status, 0);
        if (w == -1) {
            perror("waitpid");
            return 1;
        }

        if (WIFEXITED(status)) {
            printf("child %d exited with code %d\n", w, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("child %d killed by signal %d\n", w, WTERMSIG(status));
        } else {
            printf("child %d ended with other status\n", w);
        }
    }
    return 0;
}
```

## 总结

- fork 负责“复制进程”
- exec 负责“装载新程序”
- wait 负责“回收子进程” 
- 三者串起来就是 UNIX 进程创建与程序加载的标准范式
- fork()+exec()的方式实现了运行新程序的同时保留父进程持续控制的能力

## 其他API

还有很多与进程交互的系统调用，如kill()调用可以控制进程睡眠和终止

还有很多有用的命令行工具，阅读man手册以了解
