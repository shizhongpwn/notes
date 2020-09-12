# socket编程学习三-socket-API

~~~c
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
~~~

服务端尽可能使用SO_REUSEADDR,在绑定之前尽可能调用setsockopt来设置SO_REUSEADDR套接字选项。该选项可以使得server不必等待TIME_WAIT状态消失就可以重启服务器(对于TIME_WAIT状态会在后面续有叙述).

可以在bind之前添加代码(完整代码请参照博文最后):

~~~c
    int on = 1;
    if (setsockopt(listenfd,SOL_SOCKET,SO_REUSEADDR,
                   &on,sizeof(on)) == -1)
        err_exit("setsockopt SO_REUSEADDR error");
~~~

用以支持地址复用。

> 其实这一个步骤的作用在于实现套接字的设置

## process-per-connection

~~~c++
/** 示例:echo server改进, 多进程模型(client并未更改)**/
void echo(int clientfd);
int main()
{
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1)
        err_exit("socket error");
    int on = 1;
    if (setsockopt(listenfd,SOL_SOCKET,SO_REUSEADDR,
                   &on,sizeof(on)) == -1)
        err_exit("setsockopt SO_REUSEADDR error");
 
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8001);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    if (bind(listenfd, (const struct sockaddr *)&addr, sizeof(addr)) == -1)
        err_exit("bind error");
    if (listen(listenfd, SOMAXCONN) == -1)
        err_exit("listen error");
 
    struct sockaddr_in clientAddr;
    //谨记: 此处一定要初始化
    socklen_t addrLen = sizeof(clientAddr);
    while (true)
    {
        int clientfd = accept(listenfd, (struct sockaddr *)&clientAddr, &addrLen);
        if (clientfd == -1)
            err_exit("accept error");
        //打印客户IP地址与端口号
        cout << "Client information: " << inet_ntoa(clientAddr.sin_addr)
             << ", " << ntohs(clientAddr.sin_port) << endl;
 
        pid_t pid = fork();
        if (pid == -1)
            err_exit("fork error");
        else if (pid > 0)
            close(clientfd);
        //子进程处理链接
        else if (pid == 0)
        {
            close(listenfd);
            echo(clientfd);
            //子进程一定要exit, 否则的话, 该子进程也会回到accept处
            exit(EXIT_SUCCESS);
        }
    }
    close(listenfd);
}
void echo(int clientfd)
{
    char buf[512] = {0};
    int readBytes;
    while ((readBytes = read(clientfd, buf, sizeof(buf))) > 0)
    {
        cout << buf;
        if (write(clientfd, buf, readBytes) == -1)
            err_exit("write socket error");
        memset(buf, 0, sizeof(buf));
    }
    if (readBytes == 0)
    {
        cerr << "client connect closed..." << endl;
        close(clientfd);
    }
    else if (readBytes == -1)
        err_exit("read socket error");
}
~~~

这个最大的特点就是使用了子进程处理的方式，不过要注意以下几点：

* fork被调用一次，但是他有两个返回值：
  * 1）在父进程中，fork返回新创建子进程的进程ID；
      2）在子进程中，fork返回0；
      3）如果出现错误，fork返回一个负值；
* 当时的一个疑问就是listen函数的作用，应该是listen是使得套接字转变为被动套接字等待连接，是可以连接上的，accept只不过是从连接队列里面拿出来一个而已，所以只有close(listen)之后才会停止连接。

