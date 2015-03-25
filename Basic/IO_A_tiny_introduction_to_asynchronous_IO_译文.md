#A tiny introduction to asynchronous IO 
#(异步IO简介)

##说明
本文是[A tiny introduction to asynchronous IO](http://www.wangafu.net/~nickm/libevent-book/01_intro.html)的译文。<br/>

***
***
***
大部分初学的程序员开始都是使用阻塞IO调用（blocking IO calls）。这种IO是同步的（synchronous），调用发起后，不会立即返回，除非操作完成或者超时。例如，在一个TCP连接上发起一个“connect()”调用，操作系统会向连接另一端的主机发送一个SYN包。操作系统不会将控制权交回你的应用，除非它收到SYN ACK包，或者因为超时而放弃。<br/>
下面是一个真实而简单的使用阻塞网络调用的客户端程序：打开到www.google.com的连接，发送一个HTTP请求，然后将响应数据打印到标准输出中。<br/>
**示例：一个简单的阻塞式HTTP客户端**<br/>
	
	/* For sockaddr_in */
	#include <netinet/in.h>
	/* For socket functions */
	#include <sys/socket.h>
	/* For gethostbyname */
	#include <netdb.h>
	
	#include <unistd.h>
	#include <string.h>
	#include <stdio.h>
	
	int main(int c, char **v)
	{
	    const char query[] =
	        "GET / HTTP/1.0\r\n"
	        "Host: www.google.com\r\n"
	        "\r\n";
	    const char hostname[] = "www.google.com";
	    struct sockaddr_in sin;
	    struct hostent *h;
	    const char *cp;
	    int fd;
	    ssize_t n_written, remaining;
	    char buf[1024];
	
	    /* Look up the IP address for the hostname.   Watch out; this isn't
	       threadsafe on most platforms. */
	    h = gethostbyname(hostname);
	    if (!h) {
	        fprintf(stderr, "Couldn't lookup %s: %s", hostname, hstrerror(h_errno));
	        return 1;
	    }
	    if (h->h_addrtype != AF_INET) {
	        fprintf(stderr, "No ipv6 support, sorry.");
	        return 1;
	    }
	
	    /* Allocate a new socket */
	    fd = socket(AF_INET, SOCK_STREAM, 0);
	    if (fd < 0) {
	        perror("socket");
	        return 1;
	    }
	
	    /* Connect to the remote host. */
	    sin.sin_family = AF_INET;
	    sin.sin_port = htons(80);
	    sin.sin_addr = *(struct in_addr*)h->h_addr;
	    if (connect(fd, (struct sockaddr*) &sin, sizeof(sin))) {
	        perror("connect");
	        close(fd);
	        return 1;
	    }
	
	    /* Write the query. */
	    /* XXX Can send succeed partially? */
	    cp = query;
	    remaining = strlen(query);
	    while (remaining) {
	      n_written = send(fd, cp, remaining, 0);
	      if (n_written <= 0) {
	        perror("send");
	        return 1;
	      }
	      remaining -= n_written;
	      cp += n_written;
	    }
	
	    /* Get an answer back. */
	    while (1) {
	        ssize_t result = recv(fd, buf, sizeof(buf), 0);
	        if (result == 0) {
	            break;
	        } else if (result < 0) {
	            perror("recv");
	            close(fd);
	            return 1;
	        }
	        fwrite(buf, 1, result, stdout);
	    }
	
	    close(fd);
	    return 0;
	}
上述代码中所有的网络调用都是阻塞的：“gethostbyname”不会返回，除非成功解析www.google.com，或者失败；“connect”不会返回，除非建立连接；“recv”不会返回，除非收到数据或者关闭信号；“send”不会返回，除非将输出拷贝到内核的写缓冲区中。<br/>
到目前为止，阻塞IO还是可行的。如果你的程序不需要同时执行其他操作，阻塞IO可以很好的完成工作。但是，假设你需要编写一个程序同时处理多个连接呢？更具体的，假设你需要从两个连接中读取数据，但是你又不知道哪一个连接中先有数据呢？可不能像下面这样写，因为如果数据先到达fd[2]，这个程序根本不会尝试从fd[2]读取数据，除非从fd[0]和fd[1]完成读取数据。<br/>
**不好的示例**<br/>

	/* This won't work. */
	char buf[1024];
	int i, n;
	while (i_still_want_to_read()) {
	    for (i=0; i<n_sockets; ++i) {
	        n = recv(fd[i], buf, sizeof(buf), 0);
	        if (n==0)
	            handle_close(fd[i]);
	        else if (n<0)
	            handle_error(fd[i], errno);
	        else
	            handle_input(fd[i], buf, n);
	    }
	}

有时人们使用多线程或者多进程的服务器来解决这个问题。一种非常简单的实现多线程的方法是使用一个专门的进程（或者线程）来处理每一个连接。由于每一个连接都是在一个独立的进程中，所以在一个连接上的阻塞IO调用不会导致其他连接的进程阻塞。<br/>
下面是另外一个示例程序。这个程序实现了一个简单的服务器，在40713端口监听TCP连接，从输入方读取数据，一次一行，然后使用ROT13混淆后回写。（译者注：ROT13混淆是指将字母表前半部分转换为后半部分，后半部分转换为前半部分，逐个映射，例如A映射为N，n映射为a。）程序使用Unix的fork()函数为每一个新连接创建一个处理进程。<br/>
**示例：ROT13服务器（创建进程）**<br/>
	
	/* For sockaddr_in */
	#include <netinet/in.h>
	/* For socket functions */
	#include <sys/socket.h>
	
	#include <unistd.h>
	#include <string.h>
	#include <stdio.h>
	#include <stdlib.h>
	
	#define MAX_LINE 16384
	
	char
	rot13_char(char c)
	{
	    /* We don't want to use isalpha here; setting the locale would change
	     * which characters are considered alphabetical. */
	    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
	        return c + 13;
	    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
	        return c - 13;
	    else
	        return c;
	}
	
	void
	child(int fd)
	{
	    char outbuf[MAX_LINE+1];
	    size_t outbuf_used = 0;
	    ssize_t result;
	
	    while (1) {
	        char ch;
	        result = recv(fd, &ch, 1, 0);
	        if (result == 0) {
	            break;
	        } else if (result == -1) {
	            perror("read");
	            break;
	        }
	
	        /* We do this test to keep the user from overflowing the buffer. */
	        if (outbuf_used < sizeof(outbuf)) {
	            outbuf[outbuf_used++] = rot13_char(ch);
	        }
	
	        if (ch == '\n') {
	            send(fd, outbuf, outbuf_used, 0);
	            outbuf_used = 0;
	            continue;
	        }
	    }
	}
	
	void
	run(void)
	{
	    int listener;
	    struct sockaddr_in sin;
	
	    sin.sin_family = AF_INET;
	    sin.sin_addr.s_addr = 0;
	    sin.sin_port = htons(40713);
	
	    listener = socket(AF_INET, SOCK_STREAM, 0);
	
	#ifndef WIN32
	    {
	        int one = 1;
	        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	    }
	#endif
	
	    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
	        perror("bind");
	        return;
	    }
	
	    if (listen(listener, 16)<0) {
	        perror("listen");
	        return;
	    }
	
	
	
	    while (1) {
	        struct sockaddr_storage ss;
	        socklen_t slen = sizeof(ss);
	        int fd = accept(listener, (struct sockaddr*)&ss, &slen);
	        if (fd < 0) {
	            perror("accept");
	        } else {
	            if (fork() == 0) {
	                child(fd);
	                exit(0);
	            }
	        }
	    }
	}
	
	int
	main(int c, char **v)
	{
	    run();
	    return 0;
	}
那么，这是否是一次处理多个连接的最佳方案呢？我是否可以停止写作这本书，然后去做别的工作呢？不是的。首先，进程的创建（即使是线程的创建）在一些平台上是非常重量级的。因此，你可能会考虑到使用线程池而不是创建新进程。但是，线程无法从根本上解决扩展性问题。如果你的程序需要一次处理成千上万的连接，一个CPU上运行成千上万的线程并不比数个线程更高效。<br/>
如果多线程不能解决多连接的问题，那么什么可以呢？在Unix平台上，你可以使你的sockets变成非阻塞的。Unix下的实现方式是：<br/>
	fcntl(fd, F_SETFL, O_NONBLOCK);
<br/>
其中fd是socket的文件描述符[1]。将fd对应的socket设置为非阻塞之后，向fd发起的网络调用要么立即执行完成，要么返回一个特殊的错误码，提示“无法完成该操作，请重试”。所以，两个socket的示例程序可以改写成这样：<br/>
**不好的示例：频繁的轮询所有socket**<br/>

	/* This will work, but the performance will be unforgivably bad. */
	int i, n;
	char buf[1024];
	for (i=0; i < n_sockets; ++i)
	    fcntl(fd[i], F_SETFL, O_NONBLOCK);
	
	while (i_still_want_to_read()) {
	    for (i=0; i < n_sockets; ++i) {
	        n = recv(fd[i], buf, sizeof(buf), 0);
	        if (n == 0) {
	            handle_close(fd[i]);
	        } else if (n < 0) {
	            if (errno == EAGAIN)
	                 ; /* The kernel didn't have any data for us to read. */
	            else
	                 handle_error(fd[i], errno);
	         } else {
	            handle_input(fd[i], buf, n);
	         }
	    }
	}
现在我们已经使用了非阻塞的socket，上面的代码也可以运行，只是有点差强人意。因为性能太差了。两个原因：一，如果任一连接上都没有数据可读，那么程序将陷入死循环，耗尽所有的CPU资源；二，如果需要处理的连接数不止一两个，那么你需要为每一个都发起一个内核调用，以确认是否有数据。这样看来，我们需要一种方式告诉内核：“一直等待，直到其中有一个socket的数据已经准备好，并且告诉我们是哪些准备好了”。<br/>
现在人们依然在使用的最早的解决方案是select()。select()方法有3个文件描述符集合（数组形式）作为参数：一个读集合，一个写集合，一个异常处理集合。select()方法会一直等待直到任一集合中的socket准备好，然后修改这个集合以包含这些准备好的socket。下面是使用select的示例：<br/>
**示例：使用select**<br/>
	
	/* If you only have a couple dozen fds, this version won't be awful */
	fd_set readset;
	int i, n;
	char buf[1024];
	
	while (i_still_want_to_read()) {
	    int maxfd = -1;
	    FD_ZERO(&readset);
	
	    /* Add all of the interesting fds to readset */
	    for (i=0; i < n_sockets; ++i) {
	         if (fd[i]>maxfd) maxfd = fd[i];
	         FD_SET(fd[i], &readset);
	    }
	
	    /* Wait until one or more fds are ready to read */
	    select(maxfd+1, &readset, NULL, NULL, NULL);
	
	    /* Process all of the fds that are still set in readset */
	    for (i=0; i < n_sockets; ++i) {
	        if (FD_ISSET(fd[i], &readset)) {
	            n = recv(fd[i], buf, sizeof(buf), 0);
	            if (n == 0) {
	                handle_close(fd[i]);
	            } else if (n < 0) {
	                if (errno == EAGAIN)
	                     ; /* The kernel didn't have any data for us to read. */
	                else
	                     handle_error(fd[i], errno);
	             } else {
	                handle_input(fd[i], buf, n);
	             }
	        }
	    }
	}

下面是使用select()重新实现ROT13服务器的代码：<br/>
**示例：基于select的ROT13服务器**<br/>
	
	/* For sockaddr_in */
	#include <netinet/in.h>
	/* For socket functions */
	#include <sys/socket.h>
	/* For fcntl */
	#include <fcntl.h>
	/* for select */
	#include <sys/select.h>
	
	#include <assert.h>
	#include <unistd.h>
	#include <string.h>
	#include <stdlib.h>
	#include <stdio.h>
	#include <errno.h>
	
	#define MAX_LINE 16384
	
	char
	rot13_char(char c)
	{
	    /* We don't want to use isalpha here; setting the locale would change
	     * which characters are considered alphabetical. */
	    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
	        return c + 13;
	    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
	        return c - 13;
	    else
	        return c;
	}
	
	struct fd_state {
	    char buffer[MAX_LINE];
	    size_t buffer_used;
	
	    int writing;
	    size_t n_written;
	    size_t write_upto;
	};
	
	struct fd_state *
	alloc_fd_state(void)
	{
	    struct fd_state *state = malloc(sizeof(struct fd_state));
	    if (!state)
	        return NULL;
	    state->buffer_used = state->n_written = state->writing =
	        state->write_upto = 0;
	    return state;
	}
	
	void
	free_fd_state(struct fd_state *state)
	{
	    free(state);
	}
	
	void
	make_nonblocking(int fd)
	{
	    fcntl(fd, F_SETFL, O_NONBLOCK);
	}
	
	int
	do_read(int fd, struct fd_state *state)
	{
	    char buf[1024];
	    int i;
	    ssize_t result;
	    while (1) {
	        result = recv(fd, buf, sizeof(buf), 0);
	        if (result <= 0)
	            break;
	
	        for (i=0; i < result; ++i)  {
	            if (state->buffer_used < sizeof(state->buffer))
	                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
	            if (buf[i] == '\n') {
	                state->writing = 1;
	                state->write_upto = state->buffer_used;
	            }
	        }
	    }
	
	    if (result == 0) {
	        return 1;
	    } else if (result < 0) {
	        if (errno == EAGAIN)
	            return 0;
	        return -1;
	    }
	
	    return 0;
	}
	
	int
	do_write(int fd, struct fd_state *state)
	{
	    while (state->n_written < state->write_upto) {
	        ssize_t result = send(fd, state->buffer + state->n_written,
	                              state->write_upto - state->n_written, 0);
	        if (result < 0) {
	            if (errno == EAGAIN)
	                return 0;
	            return -1;
	        }
	        assert(result != 0);
	
	        state->n_written += result;
	    }
	
	    if (state->n_written == state->buffer_used)
	        state->n_written = state->write_upto = state->buffer_used = 0;
	
	    state->writing = 0;
	
	    return 0;
	}
	
	void
	run(void)
	{
	    int listener;
	    struct fd_state *state[FD_SETSIZE];
	    struct sockaddr_in sin;
	    int i, maxfd;
	    fd_set readset, writeset, exset;
	
	    sin.sin_family = AF_INET;
	    sin.sin_addr.s_addr = 0;
	    sin.sin_port = htons(40713);
	
	    for (i = 0; i < FD_SETSIZE; ++i)
	        state[i] = NULL;
	
	    listener = socket(AF_INET, SOCK_STREAM, 0);
	    make_nonblocking(listener);
	
	#ifndef WIN32
	    {
	        int one = 1;
	        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	    }
	#endif
	
	    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
	        perror("bind");
	        return;
	    }
	
	    if (listen(listener, 16)<0) {
	        perror("listen");
	        return;
	    }
	
	    FD_ZERO(&readset);
	    FD_ZERO(&writeset);
	    FD_ZERO(&exset);
	
	    while (1) {
	        maxfd = listener;
	
	        FD_ZERO(&readset);
	        FD_ZERO(&writeset);
	        FD_ZERO(&exset);
	
	        FD_SET(listener, &readset);
	
	        for (i=0; i < FD_SETSIZE; ++i) {
	            if (state[i]) {
	                if (i > maxfd)
	                    maxfd = i;
	                FD_SET(i, &readset);
	                if (state[i]->writing) {
	                    FD_SET(i, &writeset);
	                }
	            }
	        }
	
	        if (select(maxfd+1, &readset, &writeset, &exset, NULL) < 0) {
	            perror("select");
	            return;
	        }
	
	        if (FD_ISSET(listener, &readset)) {
	            struct sockaddr_storage ss;
	            socklen_t slen = sizeof(ss);
	            int fd = accept(listener, (struct sockaddr*)&ss, &slen);
	            if (fd < 0) {
	                perror("accept");
	            } else if (fd > FD_SETSIZE) {
	                close(fd);
	            } else {
	                make_nonblocking(fd);
	                state[fd] = alloc_fd_state();
	                assert(state[fd]);/*XXX*/
	            }
	        }
	
	        for (i=0; i < maxfd+1; ++i) {
	            int r = 0;
	            if (i == listener)
	                continue;
	
	            if (FD_ISSET(i, &readset)) {
	                r = do_read(i, state[i]);
	            }
	            if (r == 0 && FD_ISSET(i, &writeset)) {
	                r = do_write(i, state[i]);
	            }
	            if (r) {
	                free_fd_state(state[i]);
	                state[i] = NULL;
	                close(i);
	            }
	        }
	    }
	}
	
	int
	main(int c, char **v)
	{
	    setvbuf(stdout, NULL, _IONBF, 0);
	
	    run();
	    return 0;
	}

但是，其实这依然不是最佳方案。因为生成和读取select()中bit数组的时间是与传入select()方法中fd的最大值成正比的,随着socket数量增大，select()将无法扩展[2]。<br/>
不同的操作系统提供了不同的函数以替换select。例如poll()，epoll()，kqueue()，evports，和/dev/poll。所有这些函数都比select()的性能要好。除了poll()之外，其他函数能够达到O(1)的性能，在添加socket，删除socket和通知应用某个socket已经准备好时。<br/>
不幸的是，所有上述函数都不是标准接口。Linux使用epoll()，BSD（包括Darwin）使用kqueue()，Solaris使用evports和/dev/poll，等等。任一操作系统都不支持其他操作系统的函数。因此，如果你想编写一个跨平台的高性能应用，你需要一个抽象层，封装所有这些接口，在使用时选择最高效的那一个。<br/>
这就是Libevent库的底层API所完成的工作。它提供一个一致性的接口，来替代select()操作，同时根据运行平台的不同选择最高效的底层实现。<br/>
下面是异步ROT13服务器的另外一个版本。这一次，我们使用Libevent 2，而不是select()。我们没有使用fd_sets，而是通过event_base结构体来关联事件，无论底层的实现是select()，poll()，epoll()，还是kqueue()等。<br/>
**示例：基于Libevent的ROT13服务器**<br/>
	
	/* For sockaddr_in */
	#include <netinet/in.h>
	/* For socket functions */
	#include <sys/socket.h>
	/* For fcntl */
	#include <fcntl.h>
	
	#include <event2/event.h>
	
	#include <assert.h>
	#include <unistd.h>
	#include <string.h>
	#include <stdlib.h>
	#include <stdio.h>
	#include <errno.h>
	
	#define MAX_LINE 16384
	
	void do_read(evutil_socket_t fd, short events, void *arg);
	void do_write(evutil_socket_t fd, short events, void *arg);
	
	char
	rot13_char(char c)
	{
	    /* We don't want to use isalpha here; setting the locale would change
	     * which characters are considered alphabetical. */
	    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
	        return c + 13;
	    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
	        return c - 13;
	    else
	        return c;
	}
	
	struct fd_state {
	    char buffer[MAX_LINE];
	    size_t buffer_used;
	
	    size_t n_written;
	    size_t write_upto;
	
	    struct event *read_event;
	    struct event *write_event;
	};
	
	struct fd_state *
	alloc_fd_state(struct event_base *base, evutil_socket_t fd)
	{
	    struct fd_state *state = malloc(sizeof(struct fd_state));
	    if (!state)
	        return NULL;
	    state->read_event = event_new(base, fd, EV_READ|EV_PERSIST, do_read, state);
	    if (!state->read_event) {
	        free(state);
	        return NULL;
	    }
	    state->write_event =
	        event_new(base, fd, EV_WRITE|EV_PERSIST, do_write, state);
	
	    if (!state->write_event) {
	        event_free(state->read_event);
	        free(state);
	        return NULL;
	    }
	
	    state->buffer_used = state->n_written = state->write_upto = 0;
	
	    assert(state->write_event);
	    return state;
	}
	
	void
	free_fd_state(struct fd_state *state)
	{
	    event_free(state->read_event);
	    event_free(state->write_event);
	    free(state);
	}
	
	void
	do_read(evutil_socket_t fd, short events, void *arg)
	{
	    struct fd_state *state = arg;
	    char buf[1024];
	    int i;
	    ssize_t result;
	    while (1) {
	        assert(state->write_event);
	        result = recv(fd, buf, sizeof(buf), 0);
	        if (result <= 0)
	            break;
	
	        for (i=0; i < result; ++i)  {
	            if (state->buffer_used < sizeof(state->buffer))
	                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
	            if (buf[i] == '\n') {
	                assert(state->write_event);
	                event_add(state->write_event, NULL);
	                state->write_upto = state->buffer_used;
	            }
	        }
	    }
	
	    if (result == 0) {
	        free_fd_state(state);
	    } else if (result < 0) {
	        if (errno == EAGAIN) // XXXX use evutil macro
	            return;
	        perror("recv");
	        free_fd_state(state);
	    }
	}
	
	void
	do_write(evutil_socket_t fd, short events, void *arg)
	{
	    struct fd_state *state = arg;
	
	    while (state->n_written < state->write_upto) {
	        ssize_t result = send(fd, state->buffer + state->n_written,
	                              state->write_upto - state->n_written, 0);
	        if (result < 0) {
	            if (errno == EAGAIN) // XXX use evutil macro
	                return;
	            free_fd_state(state);
	            return;
	        }
	        assert(result != 0);
	
	        state->n_written += result;
	    }
	
	    if (state->n_written == state->buffer_used)
	        state->n_written = state->write_upto = state->buffer_used = 1;
	
	    event_del(state->write_event);
	}
	
	void
	do_accept(evutil_socket_t listener, short event, void *arg)
	{
	    struct event_base *base = arg;
	    struct sockaddr_storage ss;
	    socklen_t slen = sizeof(ss);
	    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
	    if (fd < 0) { // XXXX eagain??
	        perror("accept");
	    } else if (fd > FD_SETSIZE) {
	        close(fd); // XXX replace all closes with EVUTIL_CLOSESOCKET */
	    } else {
	        struct fd_state *state;
	        evutil_make_socket_nonblocking(fd);
	        state = alloc_fd_state(base, fd);
	        assert(state); /*XXX err*/
	        assert(state->write_event);
	        event_add(state->read_event, NULL);
	    }
	}
	
	void
	run(void)
	{
	    evutil_socket_t listener;
	    struct sockaddr_in sin;
	    struct event_base *base;
	    struct event *listener_event;
	
	    base = event_base_new();
	    if (!base)
	        return; /*XXXerr*/
	
	    sin.sin_family = AF_INET;
	    sin.sin_addr.s_addr = 0;
	    sin.sin_port = htons(40713);
	
	    listener = socket(AF_INET, SOCK_STREAM, 0);
	    evutil_make_socket_nonblocking(listener);
	
	#ifndef WIN32
	    {
	        int one = 1;
	        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	    }
	#endif
	
	    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
	        perror("bind");
	        return;
	    }
	
	    if (listen(listener, 16)<0) {
	        perror("listen");
	        return;
	    }
	
	    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
	    /*XXX check it */
	    event_add(listener_event, NULL);
	
	    event_base_dispatch(base);
	}
	
	int
	main(int c, char **v)
	{
	    setvbuf(stdout, NULL, _IONBF, 0);
	
	    run();
	    return 0;
	}

（附注：我们使用类型evutil_socket_t替换int来定义socket。在设置socket为非阻塞时，我们调用evutil_make_socket_nonblocking，而不是fcntl(O_NONBLOCK)。所有这些都是为了使得我们的代码兼容Win32的网络API。）<br/>
##关于易用性（和Windows平台）
你或许已经发现，我们的代码更高效，但是也更复杂。之前我们不需要为每一个连接管理一个缓冲区，我们只需要在堆栈上为每一个进程分配一个缓冲区；我们不需要显式跟踪任意一个读写的socket，因为这是隐含在代码中的；我们不需要维护一个结构体，以跟踪已经完成的操作，我们只需要使用循环和变量即可。<br/>
而且，如果你熟悉Windows平台的网络编程，你会发现上述场景中Libevent并不能带来最优的性能。在Windows平台，高效异步IO不是使用select()类型的函数，而是使用IOCP（IO Completion Ports）API。不像所有其他的高效网络API，在socket准备好时，IOCP并不会通知应用程序。相反，应用程序执行网络操作，IOCP在操作完成后才会通知应用程序。<br/>
幸运的是，Libevent 2 “bufferevents”接口解决了上述问题，现在程序更简洁，而且提供了在Windows和Unix平台上都非常高效的实现。<br/>
下面是使用bufferevents API的程序，也是我们ROT13服务器的最后一个实现。<br/>
**示例：一个更简洁的基于Libevent的ROT13服务器**
	
	/* For sockaddr_in */
	#include <netinet/in.h>
	/* For socket functions */
	#include <sys/socket.h>
	/* For fcntl */
	#include <fcntl.h>
	
	#include <event2/event.h>
	#include <event2/buffer.h>
	#include <event2/bufferevent.h>
	
	#include <assert.h>
	#include <unistd.h>
	#include <string.h>
	#include <stdlib.h>
	#include <stdio.h>
	#include <errno.h>
	
	#define MAX_LINE 16384
	
	void do_read(evutil_socket_t fd, short events, void *arg);
	void do_write(evutil_socket_t fd, short events, void *arg);
	
	char
	rot13_char(char c)
	{
	    /* We don't want to use isalpha here; setting the locale would change
	     * which characters are considered alphabetical. */
	    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
	        return c + 13;
	    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
	        return c - 13;
	    else
	        return c;
	}
	
	void
	readcb(struct bufferevent *bev, void *ctx)
	{
	    struct evbuffer *input, *output;
	    char *line;
	    size_t n;
	    int i;
	    input = bufferevent_get_input(bev);
	    output = bufferevent_get_output(bev);
	
	    while ((line = evbuffer_readln(input, &n, EVBUFFER_EOL_LF))) {
	        for (i = 0; i < n; ++i)
	            line[i] = rot13_char(line[i]);
	        evbuffer_add(output, line, n);
	        evbuffer_add(output, "\n", 1);
	        free(line);
	    }
	
	    if (evbuffer_get_length(input) >= MAX_LINE) {
	        /* Too long; just process what there is and go on so that the buffer
	         * doesn't grow infinitely long. */
	        char buf[1024];
	        while (evbuffer_get_length(input)) {
	            int n = evbuffer_remove(input, buf, sizeof(buf));
	            for (i = 0; i < n; ++i)
	                buf[i] = rot13_char(buf[i]);
	            evbuffer_add(output, buf, n);
	        }
	        evbuffer_add(output, "\n", 1);
	    }
	}
	
	void
	errorcb(struct bufferevent *bev, short error, void *ctx)
	{
	    if (error & BEV_EVENT_EOF) {
	        /* connection has been closed, do any clean up here */
	        /* ... */
	    } else if (error & BEV_EVENT_ERROR) {
	        /* check errno to see what error occurred */
	        /* ... */
	    } else if (error & BEV_EVENT_TIMEOUT) {
	        /* must be a timeout event handle, handle it */
	        /* ... */
	    }
	    bufferevent_free(bev);
	}
	
	void
	do_accept(evutil_socket_t listener, short event, void *arg)
	{
	    struct event_base *base = arg;
	    struct sockaddr_storage ss;
	    socklen_t slen = sizeof(ss);
	    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
	    if (fd < 0) {
	        perror("accept");
	    } else if (fd > FD_SETSIZE) {
	        close(fd);
	    } else {
	        struct bufferevent *bev;
	        evutil_make_socket_nonblocking(fd);
	        bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
	        bufferevent_setcb(bev, readcb, NULL, errorcb, NULL);
	        bufferevent_setwatermark(bev, EV_READ, 0, MAX_LINE);
	        bufferevent_enable(bev, EV_READ|EV_WRITE);
	    }
	}
	
	void
	run(void)
	{
	    evutil_socket_t listener;
	    struct sockaddr_in sin;
	    struct event_base *base;
	    struct event *listener_event;
	
	    base = event_base_new();
	    if (!base)
	        return; /*XXXerr*/
	
	    sin.sin_family = AF_INET;
	    sin.sin_addr.s_addr = 0;
	    sin.sin_port = htons(40713);
	
	    listener = socket(AF_INET, SOCK_STREAM, 0);
	    evutil_make_socket_nonblocking(listener);
	
	#ifndef WIN32
	    {
	        int one = 1;
	        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
	    }
	#endif
	
	    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
	        perror("bind");
	        return;
	    }
	
	    if (listen(listener, 16)<0) {
	        perror("listen");
	        return;
	    }
	
	    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
	    /*XXX check it */
	    event_add(listener_event, NULL);
	
	    event_base_dispatch(base);
	}
	
	int main(int c, char **v)
	{
	    setvbuf(stdout, NULL, _IONBF, 0);
	    run();
	    return 0;
	}

##关于性能
TODO<br/>

**脚注**<br/>
1. 文件描述符：当你打开一个socket时，内核分配给该socket的一个数字。使用这个数字向相应的socket发起调用。<br/>
2. 在用户空间，生成和读取bit数组的时间与你传入select()方法的文件描述符数量成正比。但是在内核中，读取bit数组的时间与数组中fd的最大值成正比，即近似于程序使用的所有fd的总数，不论传入select()方法的fd数量有多大。<br/>
<br/>
2015-3-25
