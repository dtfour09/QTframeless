![关于管道通信的一些理解](https://picx.zhimg.com/v2-8785f05ec3381c5ae6adbce57be72e28_720w.jpg?source=172ae18b)

# 关于管道通信的一些理解

[![GGBond](https://pic1.zhimg.com/v2-9c36c2e0e7bb3eaaf5a6da4d02339690_l.jpg?source=32738c0c)

管道是一种进程之间的通信方式，初学者的疑问往往不是“管道是如何实现进程之间的通信的？”而是“进程之间为什么要通信？”

进程之间通信的目的：

1. 数据传输：一个进程需要将它的数据发送给另一个进程
2. 资源共享：多个进程之间共享同样的资源
3. 通知事件：一个进程需要向另一个或一组进程发送消息，通知它发生了某种事件
4. 进程控制：有些进程希望完全控制另一个进程的执行，此时控制进程希望能够拦截另一个进程的所有陷入和异常，并能够及时知道它的状态改变

管道就像是一个特斯拉阀门，可以让数据从一边写入，一边写出，反过来就不行。那么想要让数据从另一边写入该怎么呢？——再创建一个管道就可以了，这就是管道通信。

管道分为无名管道和命名管道：

**无名管道PIPE**



```c
#include <unistd.h>
int pipe(int pipefd[2]);
```

pipe fd[2]表示一个文件描述符数组，其中fd[0]表示读端，fd[1]表示写端，函数调用成功返回0，失败返回-1。

还是不知道怎么用？一段代码实现一个小功能：从键盘读取数据写入管道，从管道中读取数据写到屏幕

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

int main()
{
    int fd[2];
    int ret=pipe(fd);
    char buf[100]={0};

    if(ret<0)
    {
        perror("pipe");
        return 1;
    }
    printf("pipe OK\n");
    int len;
    while(fgets(buf,100,stdin))
    {
        len=strlen(buf);
        //write into pipe
        ssize_t write_size=write(fd[1],buf,len);
        if(write_size<0)
        {
            perror("write");
            return 1;
        }
        memset(buf,0x00,sizeof(buf));

        //read from pipe
        ssize_t read_size=read(fd[0],buf,100);
        if(read_size < 0)
        {
            perror("read");
            return 1;
        }

        //write to pipe
        if(write(1,buf,len)!=len)
        {
            perror("write");
            return 1;
        }
    }
    return 0;
}
```

匿名管道是用于有亲缘关系的进程间通信的，那么我们fork创建出的子进程是如何来共享管道的呢？

父进程在fork出子进程后，子进程拥有了和父进程一样的代码，当然也拥有和父进程一样的文件描述符，在fork出子进程后，父进程关闭管道的读端fd[0]，子进程关闭管道的写端fd[1]，从而来实现父子进程之间的管道通信，即从子进程向管道内写数据，父进程从管道内读数据。

只能用于具有亲缘关系的进程；通常管道由一个进程创建，然后该进程调用fork，父子进程之间就可用该管道。

**无名管道的特点：**

1. 管道面向字节流（分多次读取管道的内容）
2. 管道的生命周期是随进程。进程退出，管道在内核中所对应的资源就释放
3. 内核会对管道操作进行同步与互斥
4. 管道只能往单向方向通信，若要实现双向通信必须创建两个管道

**命名管道FIFO**

命名管道相较于匿名管道的不同，就是能够适用于任意进程间的通信。不同的是，匿名管道在用pipe函数创建的同时也打开了，而命名管道在mkfifo创建了之后，还需用open函数打开该命名管道。**只有了解了无名管道，命名管道才会更加得心应手。**

不知道怎么用？一段代码……不，这次要两段代码实现简单的聊天功能：

server:发送者（先运行）

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdlib.h>

int main()
{
    umask(0);
    int ret=mkfifo("./myfifo",0666);
    if(ret < 0)
    {
        perror("mkfifo");
    }
    printf("mkfifo ok!\n");

    int rfd = open("myfifo",O_RDONLY);
    if(rfd < 0)
    {
        perror("open");
    }
    printf("open ok!\n");

    char buf[1024];
    while(1)
    {
        buf[0]=0;
        printf("Please wait...\n");
        ssize_t s = read(rfd , buf, sizeof(buf)-1);
        if(s > 0)
        {
            buf[s -1] =0;
            printf("client say %s\n",buf);
        }
        else if(s == 0)
        {
            printf("read done!\n");
            return 0;
        }
        else
        {
            perror("read");
        }
    }
    close(rfd);
    return 0;
}
```

receiver：接收者（后运行）

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    int wfd = open("myfifo",O_WRONLY);
    if(wfd < 0)
    {
        perror("open");
    }
    printf("open ok!\n");

    char buf[1024];
    while(1)
    {
        buf[0]=0;
        printf("Please Enter...\n");
        ssize_t s = read(0 , buf, sizeof(buf)-1);
        if(s > 0)
        {
            buf[s] =0;
            write(wfd,buf,sizeof(buf)-1);
        }
        else
        {
            perror("read");
        }
    }
    close(wfd);
    return 0;
}
```
