---
title: Nilsh：从零开始实现交互式 Shell
date: 2019-04-21 08:09:04
tags:
  - c
  - linux
  - shell
  - syscall
---

最近~~被生活所迫~~尝试制作 Shell —— 注意是 Interactive Shell 而非 Shell Interpreter 。参照《现代操作系统》一书的描述，Shell 的基本原理就是 `fork-and-execute` 。程序的基本框架看起来可能是这样的：

```C
#define TRUE 1

while (TRUE) {
  type_prompt();
  read_command(command, parameters);
  if (fork() == 0)
    execve(command, parameters, 0);
  else waitpid(-1, &status, 0);
}
```

当然实际实现过程中还是遇到了不少其他问题，这篇博客就简单记录下这些问题及其解决 。我的源代码就托管在 [GitHub](https://github.com/queensferryme/nilsh) 上。

<!--more-->

## 代码结构

从上面的代码片段中，你可能已经大概了解了 Shell 的执行流程了 。下面具体阐述一下：

1.  初始化 Shell 环境
2.  打印命令行提示符（如 `$` 等）
3.  读入并解析命令
4.  执行命令（命令分为内置与外部两种）
5.  返回步骤 2 或退出 Shell

其中命令的解析与执行是最重要的两部分 。在我的 [GitHub](https://github.com/queensferryme/nilsh) 上你可以看到我的代码也是依此来划分功能模块的：

-   `execute.c` 模块负责命令的执行
-   `parse.c` 模块负责命令的解析
-   `utils.c` 模块包含了其他辅助功能
-   `main.c` 包含了主函数

环境初始化以及打印提示符细枝末节就不说了，想了解请直接看源码 。

## 命令解析

然后就到了解析命令的时候了 。我的思路比较简单粗暴~~不严谨~~，将来系统学习了形式语言之类的专业课后再来重写解析器（咕）。目前的思路就是将一个简单命令划分为三个部分：命令、参数与 I/O 重定向，而一个复合命令可能由多个简单命令通过管道 `|` 连接起来 。

命令解析流程大概就这么出来了：

1.  将命令按照管道符 `|` 分割开来，如果没有管道符则视为单条简单命令；

2.  将分割出的子串再通过空格或制表符等分割开来，分类为为命令与 I/O 重定向，存储进 `Command`  结构体，结构体声明如下：

    ```C
    #define VAR_MAX_LENGTH 128
    
    typedef struct {
      char* args[VAR_MAX_LENGTH];
      char in[VAR_MAX_LENGTH];
      char out[VAR_MAX_LENGTH];
    } Command;
    ```

    其中 `args[0]` 存储的是命令，`args` 存储命令及其参数， `in` 与 `out` 存储 I/O 重定向位置（默认 `stdin` 与 `stdout`）；

3.  将各简单命令的解析结果 `Command` 存储进 `CommandPool` 结构体中，结构体声明如下：

    ```C
    typedef struct {
      Command list[VAR_MAX_LENGTH];
      size_t count;
    } CommandPool;
    ```

    其中 `count` 记录了简单命令的数目，这些简单命令被依次存储在 `Command` 结构体数组中，执行时通过管道连接；

具体实现请参见源代码，这里就不再赘述 。

## 命令执行

### fork() 执行外部命令

解析完成后就可以开始执行命令了 。首先考虑没有管道的情况，此时简单命令又被分为内置命令与外部命令 。内置命令包括 `cd` `exit` 等，它们无需或无法通过执行外部二进制文件来实现；外部命令则可能是 `/bin` 或者 `/usr/bin` 目录下的二进制可执行文件，这时我们就需要通过 `fork-and-execute` 的方式在子进程中执行之 。

你可以看到我用于执行单条简单命令的 `execute_command` 函数的实现（位于 `execute.c`）：

```C
void execute_command(Command* cmd) {
  if (is_builtin_command(cmd->args[0]))
    execute_builtin_command(cmd);
  else execute_external_command(cmd);
}
```

值得细说的是外部命令的执行，这里我们需要理解 `fork` 函数的作用 。

`fork` 函数的作用就是创建子进程，它的特殊之处就在于**执行一次返回两次** —— 在父进程中返回子进程 pid，在子进程中返回 0 。考虑以下代码：

```C
if (fork() == 0)
  printf("child process");
else printf("parent process");
```

在子进程中 `fork() == 0` 为真，故输出 `child process`；在父进程中则输出 `parent process` 。

还需要注意的是 fork 出的**子进程继承了父进程的上下文**，包括各种变量、函数以及当前执行到哪一条语句 。因此子进程被创建后会从 fork 函数后的第一条指令开始继续执行，而非从头开始 。在上例中子进程被创建后就会立刻开始判断 `fork() == 0` 是否成立，而不是从头开始执行 `main` 函数 。

然后我们再回来看 `execute_external_command` 函数，代码如下：

```C
void execute_external_command(Command* cmd) {
  pid_t pid = fork();
  if (pid == 0) {
    if (cmd->in[0])
      freopen(cmd->in, "r", stdin);
    if (cmd->out[0])
      freopen(cmd->out, "w", stdout);
    if (execvp(cmd->args[0], cmd->args))
      fprintf(stderr, "\033[0;31m%s\033[0m", "no such command\n");
    exit(0);
  }
  else waitpid(pid, NULL, 0);
}
```

在父进程中我们需要使用 `waitpid` 等待子进程执行完毕后再进入下一个读取-解析-执行主循环，而在子进程中我们也需要使用 `exit(0)` 来结束子进程，以免它进入下一个读取-解析-执行主循环 。

### freopen() 实现 I/O 重定向

上述 `execute_external_command` 中还有 `freopen` 函数，它就是用来实现 I/O 重定向的。用法也很简单：

```C
freopen("in.txt", "r", stdin);
freopen("out.txt", "w", stdout);
```

两条语句分别实现了把 `in.txt` 文件以读（read）模式打开，并将标准输入（stdin）重定向到该文件以及把 `out.txt` 文件以写（write）模式打开，并将标准输出（stdout）重定向到该文件 。

### pipe() 实现管道

`pipe` 函数用于创建管道，它接受一个二元数组 `fd` 为参数，并使 `fd[0]` 成为读端、`fd[1]` 成为写端 。这里的 `fd` 是 file descriptor 的缩写，`pipe` 函数其实就是创建了两个文件描述符，使之成为管道 —— 管道的一方向管道中写内容，另一方则从管道中读内容 。考虑以下代码：

```C
#include <stdio.h>
#include <unistd.h>

int main() {
  int fd[2];
  pipe(fd);
  pid_t pid = fork();
  if (pid == 0) {
    dup2(fd[0], 0);
    close(fd[0]);
    close(fd[1]);
    putchar(getchar());
  } else {
    dup2(fd[1], 1);
    close(fd[0]);
    close(fd[1]);
    putchar('A');
  }
  return 0;
}
```

执行结果输出字符 A，表示管道通信成功 。

那么对于具体命令 `ls -l | tail -n 2` ，我们就可以创建一个管道，使 `ls -l` 的输出写到 `fd[1]` 写端；然后 `tail -n 2` 就可以从 `fd[0]` 读端读取 `ls -l` 的输出，分别执行即可 。

对于有 n 条嵌套的管道，逻辑也是类似的，但会更复杂一些 —— 尤其是处理一层层的 `fork` 令人头疼无比 。最后我参考 StackOverflow 上的一个[回答](https://stackoverflow.com/questions/8082932/connecting-n-commands-with-pipes-in-a-shell)完成了这个功能，具体可参加 `execute_piped_command` 函数 。

## 成果展示

[![asciicast](https://asciinema.org/a/241297.svg)](https://asciinema.org/a/241297)

---

**参考文献：**

-   [Tomorrow 的博客：DIY Shell 系列](https://www.tomorrow.wiki/archives/tag/stupidshell)
-   [StackOverflow: Connecting n commands with pipes in a shell?](https://stackoverflow.com/questions/8082932/connecting-n-commands-with-pipes-in-a-shell)
