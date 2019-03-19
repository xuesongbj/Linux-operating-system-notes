# Epoll
Epoll是Linux下多路复用IO接口select/poll的增强版本,它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率,因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合,另一点原因就是获取事件的时候,它无须遍历整个被侦听的描述符集,只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

目前Epoll是Linux大规模并发网络程序中的热门首选模型。

Epool除了提供select/poll那种IO事件的水平触发(Level Triggered)外,还提供了边缘触发(Edge Triggered),这就使得用户空间程序有可能缓存IO状态,减少Epoll_wait／epoll_pwait的调用,提高应用程序效率。


## 查看最大Socket描述符和文件描述符

* Socket文件描述符

```
cat /proc/sys/fs/file-max
```

* 设置最大打开文件描述符限制

```
vim /etc/security/limits.conf
 
 *      soft    nofile     63336
 *      hard    nofile     100000
```

## Epoll API

### epoll_create

创建一个epoll句柄,参数size用来告诉内核监听的文件描述符个数,跟内存大小有关。

```
// size: 告诉内核监听的数目
int epoll_create(int size)
```

### epoll_ctl

控制epoll监控的文件描述符的事件:注册、修改、删除.

```
#include <sys/epoll.h>
// epfd:为epoll_creat的句柄
// op:表示动作,用3个宏来表示: 
// EPOLL_CTL_ADD(注册新的fd到epfd)
// EPOLL_CTL_MOD(修改已经注册的fd的监听事件)
// EPOLL_CTL_DEL(从epfd删除一个fd);
//
// event:告诉内核需要监听的事件
//
// EPOLLIN :表示对应的文件描述符可以读(包括对端SOCKET正常关闭)
// EPOLLOUT:表示对应的文件描述符可以写
// EPOLLPRI:表示对应的文件描述符有紧急的数据可读(这里应该表示有带外数据到来)
// EPOLLERR:表示对应的文件描述符发生错误// EPOLLHUP:表示对应的文件描述符被挂断;// EPOLLET: 将EPOLL设为边缘触发(Edge Triggered)模式,这是相对于水平触发(Level Triggered)来说的 要再次把这个socket加入到EPOLL队列里
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)	struct epoll_event {		__uint32_t events; /* Epoll events */		epoll_data_t data; /* User data variable */	};
```

### epoll_wait
等待所监控文件描述符上有事件的产生,类似于select()调用。

```
// events:用来从内核得到事件的集合, maxevents:告之内核这个events有多大,
// 这个maxevents的值不能大于创建epoll_create()时的size, timeout:是超时时间// -1:阻塞 
// 0:立即返回,非阻塞// >0:指定微秒
// 返回值:成功返回有多少文件描述符就绪,时间到时返回0,出错返回-1
#include <sys/epoll.h>int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout) 
```

## Demo

### server

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <errno.h>
#include "wrap.h"
 
#define MAXLINE 80 
#define SERV_PORT 8000
#define OPEN_MAX 1024

int main(int argc, char *argv[]) {	int i, j, maxi, listenfd, connfd, sockfd;
	int nready, efd, res;	ssize_t n;	char buf[MAXLINE], str[INET_ADDRSTRLEN];
	socklen_t clilen;	int client[OPEN_MAX];	struct sockaddr_in cliaddr, servaddr; struct epoll_event tep, ep[OPEN_MAX];	listenfd = Socket(AF_INET, SOCK_STREAM, 0);		bzero(&servaddr,sizeof(servaddr));	servaddr.sin_family = AF_INET;	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);	servaddr.sin_port = htons(SERV_PORT);
	Bind(listenfd, (struct sockaddr *) &servaddr, sizeof(servaddr));	Listen(listenfd, 20);
	for (i = 0; i < OPEN_MAX; i++)
		client[i] = -1;
	maxi = -1;
	
	efd = epoll_create(OPEN_MAX);
	if (efd == -1)
		perr_exit("epoll_create")
	
	tep.events = EPOLLIN;
	tep.data.fd = listenfd;
	res = epoll_ctl(efd, EPOLL_CTL_ADD, listenfd, &tep);
	if (res == -1)
		perr_exit("epoll_ctl")
	
	for (;;) {
		nready = epoll_wait(efd, ep, OPEN_MAX, -1);   /* 阻塞监听 */
		if(nready == -1)
			perr_exit("epoll_wait")
		
		for ( i = 0; i < nready; i++) {
			if (!(ep[i].events & EPOLLIN))
				continue;
				
			if (ep[i].data.fd == listenfd) {
				clilen = sizeof(cliaddr);
				connfd = Accept(listenfd, (struct sockaddr *)&cliaddr, &clilen);
				printf("received from %s at PORT %d \n", inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)), ntohs(cliaddr.sin_port));
				
				for (j = 0; j < OPEN_MAX; j++) {
					if (client[j] < 0) {
						client[j] = connfd;
						break;
					}
				}
				
				if (j == OPEN_MAX)
					perr_exit("too many clients.");
				tep.events = EPOLLIN;
				tep.data.fd = connfd;
				res = epoll_ctl(efd, EPOLL_CTL_ADD, CONNFD, &tep);
				if (res == -1)
					perr_exit("epoll_ctl");
			}
			else {
				sockfd = ep[i].data.fd;
				n = Read(sockfd, buf, MAXLINE);
				if (n == 0) {
					for (j = 0; j <= maxi; j++) {
						if (client[j] == sockfd) {
							client[j] = -1;
							break;
						}
					}
					
					res ＝ epoll_ctl(efd, EPOLL_CTL_DEL, sockfd, NULL);
					if (res == -1)
						perr_exit("epoll_ctl");
					Close(sockfd);
					printf("client[%d] closed connection\n", j);
				}
			}
		}
	}
	close(listenfd);
	close(efd);
	return 0;
}
```

### client

```
/* client.c */
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include "wrap.h"
 
#define MAXLINE 80
#define SERV_PORT 8000

int main(int argc, char *argv)
{
	struct sockaddr_in servaddr;
	char buf[MAXLINE];
	int sockfd, n;
	
	sockfd = Socket(AF_INET, SOCK_STREAM, 0);
	
	bzero(&serveraddr, sizeof(servaddr));
	
	servaddr.sin_family = AF_INET;
	inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);
	servaddr.sin_port = htons(SERV_PORT);
	
	Connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
	
	while (fgets(buf, MAXLINE, stdin) != NULL) {
		Write(sockfd, buf, strlen(buf));
		n = Read(sockfd, buf, MAXLINE);
		if (n == 0)
			printf("the other side has been closed.\n");
		else
			Write(STDOUT_FILENO, buf, n);
	}
	Close(sockfd);
	return 0;
}
```