# 【译文】epoll() 3步搞定

> 原文：[epoll() Tutorial – epoll() In 3 Easy Steps!](https://suchprogramming.com/epoll-in-3-easy-steps/) 
>
> 作者：[Kenneth Wilke](https://suchprogramming.com/author/admin/)

并不久远之前，设置单个Web服务器以支持10,000个并发连接还是一项伟大的壮举。有许多因素使开发这样的Web服务器成为可能，例如[nginx](https://www.nginx.com/)，它比以前的服务器可以处理更多的连接，效率更高。最大的因素之一是用于监视文件描述符的常量时间polling（[O(1)](https://rob-bell.net/2009/06/a-beginners-guide-to-big-o-notation/)）机制，被大多数操作系统所采用。

在[No Starch Press](https://nostarch.com/)的[《Linux编程接口》](https://www.amazon.com/gp/product/1593272200/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1593272200&linkCode=as2&tag=suchprogrammi-20&linkId=3457186e0287dc4295e1de52996ff66f)第63.4.5节中，提供了一个观察表，该表描述了几个最常用的轮询方法检查不同数量的文件描述符所花费的时间。

![](./img/poll-times.png)

如上所示，epoll的性能优势非常不错，（不同数量产生的）影响甚至只和10个描述符一样。随着描述符数量的增加，与[epoll()](https://en.wikipedia.org/wiki/Epoll)相比，使用常规[poll()](https://man7.org/linux/man-pages/man2/poll.2.html)或[select()](https://man7.org/linux/man-pages/man2/select.2.html)变得非常没有吸引力。

 本教程将介绍在Linux 2.6.27+上使用epoll()的一些基础知识。  

**必备知识**

本教程假定您熟悉Linux，C的语法以及文件描述符在类UNIX系统中的使用。

## 开始

为本教程创建一个新的工作目录，这是我们正在使用的Makefile。

```makefile
all: epoll_example
 
epoll_example: epoll_example.c
  gcc -Wall -Werror -o $@ epoll_example.c
 
clean:
  @rm -v epoll_example
```

 在整篇文章中，我将使用以下头文件所描述的功能

```c++
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close(), read()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
#include <string.h>    // for strncmp
```

## Step 1: 创建epoll文件描述符

首先，我先完成创建和关闭epoll实例的过程。

```c++
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close()
#include <sys/epoll.h> // for epoll_create1()
 
int main()
{
  int epoll_fd = epoll_create1(0);
 
  if(epoll_fd == -1)
  {
    fprintf(stderr, "Failed to create epoll file descriptor\n");
    return 1;
  }
 
  if(close(epoll_fd))
  {
    fprintf(stderr, "Failed to close epoll file descriptor\n");
    return 1;
  }
  return 0;
}
```

运行此命令应该可以工作，并且不显示任何输出，如果确实出现错误，则说明您可能正在运行一个非常老的Linux内核，或者您的系统真的需要帮助了。  

第一个示例使用[epoll_create1()](https://linux.die.net/man/2/epoll_create1)创建了一个文件描述符，这是强大的内核提供给我们的新epoll实例。尽管现在它还不能做任何事情，但我们仍应确保在程序终止之前将其清理干净。由于它与其他Linux文件描述符一样，所以我们可以使用[close()](https://linux.die.net/man/2/close)。

**水平触发和边缘触发的事件通知**

[水平触发和边缘触发](https://www.quora.com/What-are-the-key-differences-between-edge-triggered-and-level-triggered-interrupts)是从电气工程学借来的术语。当我们使用epoll时，它们的区别很重要。在边缘触发模式下，我们仅在监视文件描述符的状态更改时才接收事件；而在水平触发模式下，我们将持续接收事件，直到相应的文件描述符不再处于就绪状态为止。一般来讲，水平触发是默认设置，更易于使用，也是本教程将使用的，但是要了解边缘触发模式也是可用。

## Step 2: 为epoll添加待观测文件描述符

接下来要做的就是告诉epoll要监视的文件描述符以及要监视的事件类型。在此示例中，我将使用Linux中我最喜欢的文件描述符之一，古老的文件描述符 0（也称为标准输入）。

```c++
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
 
int main()
{
  struct epoll_event event;
  int epoll_fd = epoll_create1(0);
 
  if(epoll_fd == -1)
  {
    fprintf(stderr, "Failed to create epoll file descriptor\n");
    return 1;
  }
 
  event.events = EPOLLIN;
  event.data.fd = 0;
 
  if(epoll_ctl(epoll_fd, EPOLL_CTL_ADD, 0, &event))
  {
    fprintf(stderr, "Failed to add file descriptor to epoll\n");
    close(epoll_fd);
    return 1;
  }
 
  if(close(epoll_fd))
  {
    fprintf(stderr, "Failed to close epoll file descriptor\n");
    return 1;
  }
  return 0;

```

这里我添加了一个epoll_event结构的实例，并使用[epoll_ctl()](https://linux.die.net/man/2/epoll_ctl)将文件描述符0添加到了我们的epoll实例epoll_fd中。事件结构作为我们传入的最后一个参数，让epoll知道我们仅观察输入事件**EPOLLIN**，并提供一些用户定义的数据，这些数据将随事件返回。

## Step 3: 收获

没错！就要到了。现在就是epoll的神奇时刻。 

```c++
#define MAX_EVENTS 5
#define READ_SIZE 10
#include <stdio.h>     // for fprintf()
#include <unistd.h>    // for close(), read()
#include <sys/epoll.h> // for epoll_create1(), epoll_ctl(), struct epoll_event
#include <string.h>    // for strncmp
 
int main()
{
  int running = 1, event_count, i;
  size_t bytes_read;
  char read_buffer[READ_SIZE + 1];
  struct epoll_event event, events[MAX_EVENTS];
  int epoll_fd = epoll_create1(0);
 
  if(epoll_fd == -1)
  {
    fprintf(stderr, "Failed to create epoll file descriptor\n");
    return 1;
  }
 
  event.events = EPOLLIN;
  event.data.fd = 0;
 
  if(epoll_ctl(epoll_fd, EPOLL_CTL_ADD, 0, &event))
  {
    fprintf(stderr, "Failed to add file descriptor to epoll\n");
    close(epoll_fd);
    return 1;
  }
 
  while(running)
  {
    printf("\nPolling for input...\n");
    event_count = epoll_wait(epoll_fd, events, MAX_EVENTS, 30000);
    printf("%d ready events\n", event_count);
    for(i = 0; i < event_count; i++)
    {
      printf("Reading file descriptor '%d' -- ", events[i].data.fd);
      bytes_read = read(events[i].data.fd, read_buffer, READ_SIZE);
      printf("%zd bytes read.\n", bytes_read);
      read_buffer[bytes_read] = '\0';
      printf("Read '%s'\n", read_buffer);
 
      if(!strncmp(read_buffer, "stop\n", 5))
        running = 0;
    }
  }
 
  if(close(epoll_fd))
  {
    fprintf(stderr, "Failed to close epoll file descriptor\n");
    return 1;
  }
  return 0;
}
```

终于，我们要开张了！  

我在这里添加了一些新变量来支持和表达我在做什么。我还添加了一个while循环，该循环将持续从正在监视的文件描述符中读取数据，直到其中一个数据说“stop”为止。我使用[epoll_wait()](https://linux.die.net/man/2/epoll_wait)来等待epoll实例上事件的发生，结果将存储在事件数组中，最多MAX_EVENTS，超时时间为30秒。 epoll_wait()的返回值表示事件数组中有多少个事件数据被填充。除此之外，它还打印出所得到的内容，并执行一些基本的逻辑来完成所有的事情！

示例的执行如下：

```bash
$ ./epoll_example 

Polling for input..
```

*hello!*

```bash
1 ready events
Reading file descriptor '0' -- 7 bytes read.
Read 'hello!
'

Polling for input...
```

*this is too long for the buffer we made*

```bash
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read 'this is to'

Polling for input...
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read 'o long for'

Polling for input...
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read ' the buffe'

Polling for input...
1 ready events
Reading file descriptor '0' -- 10 bytes read.
Read 'r we made
'

Polling for input...
```

*stop*

```bash
1 ready events
Reading file descriptor '0' -- 5 bytes read.
Read 'stop
'
```

首先，我给了一个适合缓冲区的短字符串，它可以正常工作，并继续迭代循环。第二个输入对于缓冲区来说太长了，这正是水平触发帮到我们的地方；事件会持续产生，直到它读取了缓冲区中剩余的所有内容，在边缘触发模式下，我们将只收到1次通知，并且应用程序按原样进行，直到将更多内容写入正在监视的文件描述符中。

我希望这些可以帮助您了解如何使用epoll()。如果您有任何问题，疑问或反馈，不胜感激！