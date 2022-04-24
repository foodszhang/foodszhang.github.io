# Lab: Xv6 and Unix utilities
## Boot xv6
```bash
git clone git://g.csail.mit.edu/xv6-labs-2021
```
没啥好说的， 装源码和依赖， 就是这个git巨慢， 不知道为啥， 挂了梯子还是慢
## sleep(easy)
属于上来没理解的就先蒙圈的， 不过这里提示做的真的是很好， 把基本可能遇到的问题都说了

问题的关键就是这里并不是让你自己实现一个sleep的功能， 实际是调用操作系统的系统调用， 而这个系统调用实际已经是定义好了签名， 对着签名直接调用就完事了

##pingpong(easy)
这个程序使用了几个常见的系统调用， *pipe*， *fork*， *read*, *write*

关键点：

* pipe是传递一个长度为2的int数组， 第一个元素是读管道， 第二个元素写管道
* fork的返回值如果是0， 代表当前为子进程， （这里一开始我还记混了， 导致程序子进程先退出而无法通过test）
剩下没啥问题

##primes (moderate)
难度开始有所提升， 乍一看不知道如何入手，[这个网页](http://swtch.com/~rsc/thread/) 的描述性文字其实我都没咋看， 那张图很关键
![算法](https://swtch.com/~rsc/thread/sieve.gif)
这玩意实际是多个进程， 每个进程和自己的父进程与子进程通过管道传输数据， 输出每个进程收到的第一个数字， 并过滤掉这个数字的倍数， 并将剩余数字传递给子进程
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int create_child(int fd){
    int pipefd[2];
    int result = pipe(pipefd);
    if(result == -1){
        int s;
        int r = read(fd,&s,4);
        if(r <= 0){
            exit(0);
        }
        //printf("prime %d\n", s);
        printf("prime %d\n", s);
        close(fd);
        exit(0);
    }
    int pid = fork();

    if(pid!=0){
        close(pipefd[0]);
        int first = 0;
        while(1){
            int s;
            int r = read(fd,&s,4);
            if(r <= 0){
                break;
            }
            if(first == 0){
                printf("prime %d\n", s);
                first = s;
                continue;
            }
            if(s % first != 0){
                //printf("prime %d\n", s);
                write(pipefd[1], &s, 4);
            }
        }
        close(pipefd[1]);
        close(fd);
        wait(0);
        
    } else {
        close(pipefd[1]);
        create_child(pipefd[0]);
    }
    exit(0);

}


int main(int argc, char *argv[]){
    int pipefd[2];
    int result = pipe(pipefd);
    if(result == -1){
        exit(0);
    }
    int pid = fork();
    if(pid==0){
        close(pipefd[1]);
        create_child(pipefd[0]);
    } else {
        int s=2;
        close(pipefd[0]);
        while(1){
            write(pipefd[1], &s, 4);
            s += 1;
            if (s> 35){
                break;
            }
        }
        close(pipefd[1]);
        wait(0);
    }
    exit(0);
}
```

关键点：

* 这里系统的fd很容易超， 所以一开始写的有问题的时候， 数字总是不够
* pipe的返回值如果是-1， 代表无法新建新的fd，同时如果read读管道的返回值=0， 说明写管道已经被关闭了
( 这里如果一个进程fork出子进程， fd是会被重复使用的， 也就是单纯父进程关闭fd， fd并不会关闭， 因为实际子进程也有一个保持的状态， 所以操作就得是， 读进程把写管道事先关闭， 写进程把读管道事先关闭， 从而达到父进程关闭写管道后， read读管道返回0。


