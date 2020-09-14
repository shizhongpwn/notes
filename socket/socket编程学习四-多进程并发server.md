# socket编程学习四-多进程并发server

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

## p2p聊天程序设计与实现

server端与client都有两个进程:

  (1)父进程负责从socket中读取数据将其写至终端, 由于父进程使用的是read系统调用的阻塞版本, 因此如果socket中没有数据的话, 父进程会一直阻塞; 如果read返回0, 表示对端连接关闭, 则父进程会发送SIGUSR1信号给子进程, 通知其退出;

  (2)子进程负责从键盘读取数据将其写入socket, 如果键盘没有数据的话, 则fgets调用会一直阻塞;

~~~c
//serever端代码与说明
int main()
{
    int listenfd = socket(AF_INET, SOCK_STREAM, 0); //创建socket
    if (listenfd == -1)
        err_exit("socket error");
    int on = 1;
    if (setsockopt(listenfd,SOL_SOCKET,SO_REUSEADDR,
                   &on,sizeof(on)) == -1) //设置socket，SO_REUSEADDR选项可以使得server不必等待TIME_WAIT状态消失就可以重启服务器
        err_exit("setsockopt SO_REUSEADDR error");
 
    struct sockaddr_in addr;
    addr.sin_family = AF_INET; //这个就是本机
    addr.sin_port = htons(8001);
    addr.sin_addr.s_addr = htonl(INADDR_ANY); //协议和端口，IP
    if (bind(listenfd, (const struct sockaddr *)&addr, sizeof(addr)) == -1) //绑定本机
        err_exit("bind error");
    if (listen(listenfd, SOMAXCONN) == -1) //进入监听模式
        err_exit("listen error");
 
    struct sockaddr_in clientAddr;
    socklen_t addrLen = sizeof(clientAddr);
    int clientfd = accept(listenfd, (struct sockaddr *)&clientAddr, &addrLen); //开始从队列里面拿出连接
    if (clientfd == -1)
        err_exit("accept error");
    close(listenfd);
    //打印客户IP地址与端口号
    cout << "Client information: " << inet_ntoa(clientAddr.sin_addr)
         << ", " << ntohs(clientAddr.sin_port) << endl;
 
    char buf[512] = {0};
    pid_t pid = fork(); //多线程
    if (pid == -1)
        err_exit("fork error");
    //父进程: socket -> terminal
    else if (pid > 0) //父进程执行
    {
        int readBytes;
        while ((readBytes = read(clientfd, buf, sizeof(buf))) > 0) //从socket读取
        {
            cout << buf;
            memset(buf, 0, sizeof(buf));
        }
        if (readBytes == 0)
            cout << "client connect closed...\nserver exiting..." << endl;
        else if (readBytes == -1)
            err_exit("read socket error");
        //通知子进程退出
        kill(pid, SIGUSR1);
    }
    //子进程: keyboard -> socket
    else if (pid == 0) //进入子进程，从键盘读入数据给socket
    {
        signal(SIGUSR1, sigHandler);
        while (fgets(buf, sizeof(buf), stdin) != NULL)
        {
            if (write(clientfd, buf, strlen(buf)) == -1)
                err_exit("write socket error");
            memset(buf, 0, sizeof(buf));
        }
    }
    close(clientfd);
    exit(EXIT_SUCCESS);
}
~~~

### client端代码

~~~c
//client端代码与说明
int main()
{
    int sockfd = socket(AF_INET, SOCK_STREAM, 0); //创建socket
    if (sockfd == -1)
        err_exit("socket error");
 
    //填写服务器端口号与IP地址
    struct sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET; //tcp协议
    serverAddr.sin_port = htons(8001); //端口
    serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1"); //ip
    if (connect(sockfd, (const struct sockaddr *)&serverAddr, sizeof(serverAddr)) == -1) //连接目标服务器
        err_exit("connect error");
 
    char buf[512] = {0};
    pid_t pid = fork(); //多线程
    if (pid == -1)
        err_exit("fork error");
    //父进程: socket -> terminal
    else if (pid > 0) //父进程
    {
        int readBytes;
        while ((readBytes = read(sockfd, buf, sizeof(buf))) > 0) //从socket读入
        {
            cout << buf;
            memset(buf, 0, sizeof(buf));
        }
        if (readBytes == 0)
            cout << "server connect closed...\nclient exiting..." << endl;
        else if (readBytes == -1)
            err_exit("read socket error");
        kill(pid, SIGUSR1);
    }
    //子进程: keyboard -> socket
    else if (pid == 0)
    {
        signal(SIGUSR1, sigHandler);
        while (fgets(buf, sizeof(buf), stdin) != NULL) //从键盘读入
        {
            if (write(sockfd, buf, strlen(buf)) == -1) //写入socket
                err_exit("write socket error");
            memset(buf, 0, sizeof(buf));
        }
    }
    close(sockfd);
    exit(EXIT_SUCCESS);
}
~~~

