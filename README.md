# 基于nginx的简单webserver框架

[原文链接]https://yunf194.github.io/2022/03/01/202231-nginx%E6%95%B4%E4%BD%93%E6%A1%86%E6%9E%B6%E8%A7%A3%E6%9E%90/



## master进程创建监听端口并且初始化

1.对于创建的每个要监听的端口都要创建1个socket，ipv4，任意地址，所有网卡设定。

2.`setsockopt(isock,SOL_SOCKET, SO_REUSEADDR,(const void *) &reuseaddr, sizeof(reuseaddr))`允许重用本地地址,主要是解决TIME_WAIT这个状态导致bind()失败的问题

3.`setnonblocking(isock)`设置isock为非阻塞，这样在后续accept的时候就不再会去阻塞住了，

4.对isock进行bind(),listen()

5.将各个监听的isock（目前为2个）放入监听套接字队列CSocekt::vector<lpngx_listening_t> m_ListenSocketList; 

注意：这是在主进程中创建监听端口(主进程执行这个函数)，一旦后续fork()出来四个子进程，五个进程都在监听80和443两个端口。

```cpp
//监听端口【支持多个端口】，这里遵从nginx的函数命名
//在创建worker进程之前就要执行这个函数；
bool CSocekt::ngx_open_listening_sockets()
{    
    int                isock;                //socket
    struct sockaddr_in serv_addr;            //服务器的地址结构体
    int                iport;                //端口
    char               strinfo[100];         //临时字符串 
   
    //初始化相关
    memset(&serv_addr,0,sizeof(serv_addr));  //先初始化一下
    serv_addr.sin_family = AF_INET;                //选择协议族为IPV4
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY); //监听本地所有的IP地址；INADDR_ANY表示的是一个服务器上所有的网卡（服务器可能不止一个网卡）多个本地ip地址都进行绑定端口号，进行侦听。

    //中途用到一些配置信息
    CConfig *p_config = CConfig::GetInstance();
    for(int i = 0; i < m_ListenPortCount; i++) //要监听这么多个端口，暂时为2，循环2次
    {        
        //参数1：AF_INET：使用ipv4协议，一般就这么写
        //参数2：SOCK_STREAM：使用TCP，表示可靠连接【相对还有一个UDP套接字，表示不可靠连接】
        //参数3：给0，固定用法，就这么记
        isock = socket(AF_INET,SOCK_STREAM,0); //系统函数，成功返回非负描述符，出错返回-1
        if(isock == -1)
        {
            ngx_log_stderr(errno,"CSocekt::Initialize()中socket()失败,i=%d.",i);
            //其实这里直接退出，那如果以往有成功创建的socket呢？就没得到释放吧，当然走到这里表示程序不正常，应该整个退出，也没必要释放了 
            return false;
        }

        //setsockopt（）:设置一些套接字参数选项；
        //参数2：是表示级别，和参数3配套使用，也就是说，参数3如果确定了，参数2就确定了;
        //参数3：允许重用本地地址
        //设置 SO_REUSEADDR，主要是解决TIME_WAIT这个状态导致bind()失败的问题
        int reuseaddr = 1;  //1:打开对应的设置项
        if(setsockopt(isock,SOL_SOCKET, SO_REUSEADDR,(const void *) &reuseaddr, sizeof(reuseaddr)) == -1)
        {
            ngx_log_stderr(errno,"CSocekt::Initialize()中setsockopt(SO_REUSEADDR)失败,i=%d.",i);
            close(isock); //设置不了重用本地地址，那么我还是关闭这个连接并且return false吧                                                  
            return false;
        }

        //为处理惊群问题使用reuseport
        
        int reuseport = 1;
        if (setsockopt(isock, SOL_SOCKET, SO_REUSEPORT,(const void *) &reuseport, sizeof(int))== -1) //端口复用需要内核支持
        {
            //失败就失败吧，失败顶多是惊群，但程序依旧可以正常运行，所以仅仅提示一下即可
            ngx_log_stderr(errno,"CSocekt::Initialize()中setsockopt(SO_REUSEPORT)失败",i);
        }        

        //为了让监听套接字accept()快一点将ESTABLISHED状态队列的socket快点取走，设置该socket为非阻塞，
        //非阻塞不卡住就算没有客户端连入accept也会立即返回，但是返回一个错误码。充分利用时间片。
        //如果是阻塞的，accept会一直阻塞直到有连接到来，这样卡住（休眠），等待一个事情（三次握手成功）发生了，才会继续往下走
        if(setnonblocking(isock) == false)
        {                
            ngx_log_stderr(errno,"CSocekt::Initialize()中setnonblocking()失败,i=%d.",i);
            close(isock);
            return false;
        }

        //设置本服务器要监听的地址和端口，这样客户端才能连接到该地址和端口并发送数据        
        strinfo[0] = 0;
        sprintf(strinfo,"ListenPort%d",i);//取ListenPort0,ListenPort1....
        iport = p_config->GetIntDefault(strinfo,10000);
        serv_addr.sin_port = htons((in_port_t)iport);   //in_port_t其实就是uint16_t

        //绑定服务器地址结构体
        if(bind(isock, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1)
        {
            ngx_log_stderr(errno,"CSocekt::Initialize()中bind()失败,i=%d.",i);
            close(isock);
            return false;
        }
        
        //开始监听
        if(listen(isock,NGX_LISTEN_BACKLOG) == -1)
        {
            ngx_log_stderr(errno,"CSocekt::Initialize()中listen()失败,i=%d.",i);
            close(isock);
            return false;
        }

        //可以，放到列表里来
        /*struct ngx_listening_s  //和监听端口有关的结构
        {
            int                       port;        //监听的端口号
            int                       fd;          //套接字句柄socket
            lpngx_connection_t        connection;  //连接池中的一个连接，注意这是个指针 
        };*/
        lpngx_listening_t p_listensocketitem = new ngx_listening_t; //千万不要写错，注意前边类型是指针，后边类型是一个结构体
        memset(p_listensocketitem,0,sizeof(ngx_listening_t));      //注意后边用的是 ngx_listening_t而不是lpngx_listening_t
        p_listensocketitem->port = iport;                          //记录下所监听的端口号
        p_listensocketitem->fd   = isock;                          //套接字木柄保存下来   
        ngx_log_error_core(NGX_LOG_INFO,0,"监听%d端口成功!",iport); //显示一些信息到日志中
        m_ListenSocketList.push_back(p_listensocketitem);          //加入到队列中
        //具体绑定到连接池的连接后续绑定

    } //end for(int i = 0; i < m_ListenPortCount; i++)    
    if(m_ListenSocketList.size() <= 0)  //不可能一个端口都不监听吧
        return false;
    return true;
}
```

### ngx_epoll_init

创建监听端口是在父进程中（ngx_open_listening_sockets）进行的，那么完整的初始化监听socket（包括创建连接池并且将每个监听socket放入到连接池的连接和放入epoll红黑树开始监听事件）是在子进程

1.在这里创建一个epoll对象，一定要判断返回值，这是一个好习惯

2.创建连接池

3.每个监听socket增加一个 连接池中的连接，同时连接池内的连接（只有是监听socket的连接才可以）也可以通过`lpngx_connection_t::p_Conn->listening = (*pos);`找到监听socket对象。

4.连接池内的连接用`p_Conn->rhandler = &CSocekt::ngx_event_accept;`将读事件的相关处理方法设置为`ngx_event_accept()`函数。

5.监听socket上增加(EPOLL_CTL_ADD)监听的事件，读事件和TCP连接关闭即对应`EPOLLIN|EPOLLRDHUP`事件，

```cpp
//(1)epoll功能初始化，子进程中进行 ，本函数被ngx_worker_process_init()所调用
int CSocekt::ngx_epoll_init()
{
    //(1)很多内核版本不处理epoll_create的参数，只要该参数>0即可
    //创建一个epoll对象，创建了一个红黑树，还创建了一个双向链表
    m_epollhandle = epoll_create(m_worker_connections);   //直接以epoll连接的最大项数为参数，肯定是>0的； 
    if (m_epollhandle == -1) 
    {
        ngx_log_stderr(errno,"CSocekt::ngx_epoll_init()中epoll_create()失败.");
        exit(2); //这是致命问题了，直接退，资源由系统释放吧，这里不刻意释放了，比较麻烦
    }

    //(2)创建连接池【数组】、创建出来，这个东西后续用于处理所有客户端的连接
    //子进程创建以后，会继承父进程的全局变量，但是继承的是父进程刚开始全局变量的值。
    //但是子进程创建以后，子进程修改了变量，或者父进程修改了全局变量的值，父子进程就互相都不影响了。
    initconnection();
    
    //(3)遍历所有监听socket【监听端口】，我们为每个监听socket增加一个 连接池中的连接【说白了就是让一个socket和一个内存绑定，以方便记录该sokcet相关的数据、状态等等】
    std::vector<lpngx_listening_t>::iterator pos;	
	for(pos = m_ListenSocketList.begin(); pos != m_ListenSocketList.end(); ++pos)
    {
        lpngx_connection_t p_Conn = ngx_get_connection((*pos)->fd); //从连接池中获取一个空闲连接对象
        if (p_Conn == NULL)
        {
            //这是致命问题，刚开始怎么可能连接池就为空呢？
            ngx_log_stderr(errno,"CSocekt::ngx_epoll_init()中ngx_get_connection()失败.");
            exit(2); //这是致命问题了，直接退，资源由系统释放吧，这里不刻意释放了，比较麻烦
        }
        
        p_Conn->listening = (*pos);   //连接对象 和监听对象关联，方便通过连接对象找监听对象
        (*pos)->connection = p_Conn;  //监听对象 和连接对象关联，方便通过监听对象找连接对象

        //rev->accept = 1; //监听端口必须设置accept标志为1  ，这个是否有必要，再研究

        //对监听端口的读事件设置处理方法，因为监听端口是用来等对方连接的发送三路握手的，所以监听端口关心的就是读事件
        p_Conn->rhandler = &CSocekt::ngx_event_accept;

        //往监听socket上增加监听事件，从而开始让监听端口履行其职责【如果不加这行，虽然端口能连上，但不会触发ngx_epoll_process_events()里边的epoll_wait()往下走】
        if(ngx_epoll_oper_event(
                                (*pos)->fd,         //socekt句柄
                                EPOLL_CTL_ADD,      //事件类型，这里是增加
                                EPOLLIN|EPOLLRDHUP, //标志，这里代表要增加的标志,EPOLLIN：可读，EPOLLRDHUP：TCP连接的远端关闭或者半关闭
                                0,                  //对于事件类型为增加的，不需要这个参数
                                p_Conn              //连接池中的连接 
                                ) == -1) 
        {
            exit(2); //有问题，直接退出，日志 已经写过了
        }
    } //end for 
    return 1;
}
```



### ngx_close_connection直接关闭连接

```cpp
//用户连入，我们accept4()时，得到的socket在处理中产生失败，则资源用这个函数释放【因为这里涉及到好几个要释放的资源，所以写成函数】
//我们把ngx_close_accepted_connection()函数改名为让名字更通用，并从文件ngx_socket_accept.cxx迁移到本文件中，并改造其中代码，注意顺序
void CSocekt::ngx_close_connection(lpngx_connection_t pConn)
{    
    //pConn->fd = -1; //官方nginx这么写，这么写有意义；    不要这个东西，回收时不要轻易东连接里边的内容
    ngx_free_connection(pConn); 
    if(pConn->fd != -1)
    {
        close(pConn->fd);
        pConn->fd = -1;
    }    
    return;
}
```



### ngx_epoll_oper_event对epoll事件的具体操作

- 三次握手进来增加读事件ADD
- 监听端口有ADD

注意：原来的理解中，绑定ev.data.ptr这个事，只在EPOLL_CTL_ADD的时候做一次即可，但是发现EPOLL_CTL_MOD**似乎会破坏掉ev.data.ptr**，因此不管是EPOLL_CTL_ADD，还是EPOLL_CTL_MOD，ev.data.ptr都要去重新绑定！！！

```cpp
//对epoll事件的具体操作
//返回值：成功返回1，失败返回-1；
int CSocekt::ngx_epoll_oper_event(
                        int                fd,               //句柄，一个socket
                        uint32_t           eventtype,        //事件类型，一般是EPOLL_CTL_ADD，EPOLL_CTL_MOD，EPOLL_CTL_DEL ，说白了就是操作epoll红黑树的节点(增加，修改，删除)
                        uint32_t           flag,             //标志，具体含义取决于eventtype
                        int                bcaction,         //补充动作，用于补充flag标记的不足  :  0：增加   1：去掉 2：完全覆盖 ,eventtype是EPOLL_CTL_MOD时这个参数就有用
                        lpngx_connection_t pConn             //pConn：一个指针【其实是一个连接】，EPOLL_CTL_ADD时增加到红黑树中去，将来epoll_wait时能取出来用
                        )
{
    struct epoll_event ev;    
    memset(&ev, 0, sizeof(ev));

    if(eventtype == EPOLL_CTL_ADD) //往红黑树中增加节点；
    {
        //红黑树从无到有增加节点
        //ev.data.ptr = (void *)pConn;
        ev.events = flag;      //既然是增加节点，则不管原来是啥标记
        pConn->events = flag;  //这个连接本身也记录这个标记
    }
    else if(eventtype == EPOLL_CTL_MOD)
    {
        //节点已经在红黑树中，修改节点的事件信息
        ev.events = pConn->events;  //先把标记恢复回来
        if(bcaction == 0)
        {
            //增加某个标记            
            ev.events |= flag;
        }
        else if(bcaction == 1)
        {
            //去掉某个标记
            ev.events &= ~flag;
        }
        else
        {
            //完全覆盖某个标记            
            ev.events = flag;      //完全覆盖            
        }
        pConn->events = ev.events; //记录该标记
    }
    else
    {
        //删除红黑树中节点，目前没这个需求【socket关闭这项会自动从红黑树移除】，所以将来再扩展
        return  1;  //先直接返回1表示成功
    } 

    //原来的理解中，绑定ptr这个事，只在EPOLL_CTL_ADD的时候做一次即可，但是发现EPOLL_CTL_MOD似乎会破坏掉.data.ptr，因此不管是EPOLL_CTL_ADD，还是EPOLL_CTL_MOD，都给进去
    //找了下内核源码SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,		struct epoll_event __user *, event)，感觉真的会覆盖掉：
       //copy_from_user(&epds, event, sizeof(struct epoll_event)))，感觉这个内核处理这个事情太粗暴了
    ev.data.ptr = (void *)pConn;

    if(epoll_ctl(m_epollhandle,eventtype,fd,&ev) == -1)
    {
        ngx_log_stderr(errno,"CSocekt::ngx_epoll_oper_event()中epoll_ctl(%d,%ud,%ud,%d)失败。",fd,eventtype,flag,bcaction);    
        return -1;
    }
    return 1;
}
```

## worker子进程处理网络事件（读、写）和定时器事件

### 事件驱动模型ngx_epoll_process_events

子进程初始化完监听端口和设置好子进程标题之后，执行for死循环，死循环内不断调用ngx_epoll_process_events。

“事件驱动”，无非就是通过获取事件，通过获取到的事件并根据这个事件来调用适当的函数从而让整个程序干活，无非就是这点事;

1.事件驱动我们第一步一定是获取事件，如何获取事件，调用epoll_wait()。

2.一定要严密的判断，epoll_wait返回事件的数目，而事件会返回到参数m_events里，先判断events数目执行对应的判断。

3.然后for循环里不断地通过m_events[i].data.ptr把发生了事件的**连接池中的连接**取出来并且`revents = m_events[i].events;`取出**这个连接的事件类型**

4.判断事件类型，
	对于读事件，`(this->* (p_Conn->rhandler) )(p_Conn);`函数指针
	对于监听套接字的连接会调用`CSocekt::ngx_event_accept(c)`，这在子进程创建时进行初始化ngx_epoll_init函数中就已经将连接池内的监听套接字连接的函数指针指定到ngx_event_accept上
	如果是已经连入，发送数据到这里，则这里执行的应该是 `CSocekt::ngx_read_request_handler()`
5.判断事件类型，对于写事件

![image-20220308120234266](../images/202231-nginx整体框架解析/image-20220308120234266.png)

注意我这个ngx_epoll_process_events中epoll_wait相当于事件收集器，各个事件对应的处理函数都属于事件处理器，用来消费事件。因此每个处理函数不能够被阻塞，而且应该尽快执行完成，否则整个for死循环中的ngx_epoll_process_events卡住了，下一次epoll_wait函数积累的事件越来越多整个程序就会崩盘了。

```cpp
//开始获取发生的事件消息
//参数unsigned int timer：epoll_wait()阻塞的时长，单位是毫秒；
//返回值，1：正常返回  ,0：有问题返回，一般不管是正常还是问题返回，都应该保持进程继续运行
//本函数被ngx_process_events_and_timers()调用，而ngx_process_events_and_timers()是在子进程的死循环中被反复调用
int CSocekt::ngx_epoll_process_events(int timer) 
{   
    //等待事件，事件会返回到m_events里，最多返回NGX_MAX_EVENTS个事件【因为我只提供了这些内存】；
    //如果两次调用epoll_wait()的事件间隔比较长，则可能在epoll的双向链表中，积累了多个事件，所以调用epoll_wait，可能取到多个事件
    //阻塞timer这么长时间除非：a)阻塞时间到达 b)阻塞期间收到事件【比如新用户连入】会立刻返回c)调用时有事件也会立刻返回d)如果来个信号，比如你用kill -1 pid测试
    //如果timer为-1则一直阻塞，如果timer为0则立即返回，即便没有任何事件
    //返回值：有错误发生返回-1，错误在errno中，比如你发个信号过来，就返回-1，错误信息是(4: Interrupted system call)
    //       如果你等待的是一段时间，并且超时了，则返回0；
    //       如果返回>0则表示成功捕获到这么多个事件【返回值里】
    int events = epoll_wait(m_epollhandle,m_events,NGX_MAX_EVENTS,timer);//一直阻塞在这里
    
    if(events == -1)//一定要严密地判断
    {
        //有错误发生，发送某个信号给本进程就可以导致这个条件成立，而且错误码根据观察是4；
        //#define EINTR  4，EINTR错误的产生：当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。
               //例如：在socket服务器端，设置了信号捕获机制，有子进程，当在父进程阻塞于慢系统调用时由父进程捕获到了一个有效信号时，内核会致使accept返回一个EINTR错误(被中断的系统调用)。
        if(errno == EINTR) 
        {
            //信号所致，直接返回，一般认为这不是毛病，但还是打印下日志记录一下，因为一般也不会人为给worker进程发送消息
            ngx_log_error_core(NGX_LOG_INFO,errno,"CSocekt::ngx_epoll_process_events()中epoll_wait()失败!"); 
            return 1;  //正常返回
        }
        else
        {
            //这被认为应该是有问题，记录日志
            ngx_log_error_core(NGX_LOG_ALERT,errno,"CSocekt::ngx_epoll_process_events()中epoll_wait()失败!"); 
            return 0;  //非正常返回 
        }
    }

    if(events == 0) //超时，但没事件来
    {
        if(timer != -1)
        {
            //要求epoll_wait阻塞一定的时间而不是一直阻塞，这属于阻塞到时间了，则正常返回
            return 1;
        }
        //无限等待【所以不存在超时】，但却没返回任何事件，这应该不正常有问题        
        ngx_log_error_core(NGX_LOG_ALERT,0,"CSocekt::ngx_epoll_process_events()中epoll_wait()没超时却没返回任何事件!"); 
        return 0; //非正常返回 
    }

    //会惊群，一个telnet上来，4个worker进程都会被惊动，都执行下边这个
    ngx_log_stderr(0,"惊群测试:events=%d,进程id=%d",events,ngx_pid); 
    //ngx_log_stderr(0,"----------------------------------------"); 

    //走到这里，就是属于有事件收到了
    lpngx_connection_t p_Conn
    uint32_t           revents;
    for(int i = 0; i < events; ++i)    //遍历本次epoll_wait返回的所有事件，注意events才是返回的实际事件数量
    {
        p_Conn = (lpngx_connection_t)(m_events[i].data.ptr);           //ngx_epoll_add_event()给进去的，这里能取出来

        //能走到这里，我们认为这些事件都没过期，就正常开始处理
        revents = m_events[i].events;//取出事件类型
        
        /*
        if(revents & (EPOLLERR|EPOLLHUP)) //例如对方close掉套接字，这里会感应到【换句话说：如果发生了错误或者客户端断连】
        {
            //这加上读写标记，方便后续代码处理，至于怎么处理，后续再说，这里也是参照nginx官方代码引入的这段代码；
            //官方说法：if the error events were returned, add EPOLLIN and EPOLLOUT，to handle the events at least in one active handler
            //我认为官方也是经过反复思考才加上着东西的，先放这里放着吧； 
            revents |= EPOLLIN|EPOLLOUT;   //EPOLLIN：表示对应的链接上有数据可以读出（TCP链接的远端主动关闭连接，也相当于可读事件，因为本服务器小处理发送来的FIN包）
                                           //EPOLLOUT：表示对应的连接上可以写入数据发送【写准备好】            
        } */

        if(revents & EPOLLIN)  //如果是读事件
        {
            //ngx_log_stderr(errno,"数据来了来了来了 ~~~~~~~~~~~~~.");
            //一个客户端新连入，这个会成立，
            //已连接发送数据来，这个也成立；
            //c->r_ready = 1;                         //标记可以读；【从连接池拿出一个连接时这个连接的所有成员都是0】            
            (this->* (p_Conn->rhandler) )(p_Conn);    //注意括号的运用来正确设置优先级，防止编译出错；【如果是个新客户连入
                                                      //如果新连接进入，这里执行的应该是CSocekt::ngx_event_accept(c)】            
                                                      //如果是已经连入，发送数据到这里，则这里执行的应该是 CSocekt::ngx_read_request_handler()     
                                     
        }
        
        if(revents & EPOLLOUT) //如果是写事件【对方关闭连接也触发这个，再研究。。。。。。】，注意上边的 if(revents & (EPOLLERR|EPOLLHUP))  revents |= EPOLLIN|EPOLLOUT; 读写标记都给加上了
        {
            //ngx_log_stderr(errno,"22222222222222222222.");
            if(revents & (EPOLLERR | EPOLLHUP | EPOLLRDHUP)) //客户端关闭，如果服务器端挂着一个写通知事件，则这里个条件是可能成立的
            {
                //EPOLLERR：对应的连接发生错误                     8     = 1000 
                //EPOLLHUP：对应的连接被挂起                       16    = 0001 0000
                //EPOLLRDHUP：表示TCP连接的远端关闭或者半关闭连接   8192   = 0010  0000   0000   0000
                //我想打印一下日志看一下是否会出现这种情况
                //8221 = ‭0010 0000 0001 1101‬  ：包括 EPOLLRDHUP ，EPOLLHUP， EPOLLERR
                //ngx_log_stderr(errno,"CSocekt::ngx_epoll_process_events()中revents&EPOLLOUT成立并且revents & (EPOLLERR|EPOLLHUP|EPOLLRDHUP)成立,event=%ud。",revents); 

                //我们只有投递了 写事件，但对端断开时，程序流程才走到这里，投递了写事件意味着 iThrowsendCount标记肯定被+1了，这里我们减回
                --p_Conn->iThrowsendCount;                 
            }
            else
            {
                (this->* (p_Conn->whandler) )(p_Conn);   //如果有数据没有发送完毕，由系统驱动来发送，则这里执行的应该是 CSocekt::ngx_write_request_handler()
            }            
        }
    } //end for(int i = 0; i < events; ++i)     
    return 1;
}
```

### 处理三次握手连入事件ngx_event_accept

三次握手进来了，触发了epoll的读事件，前来调用此函数。accept是从已完成连接established队列取出该socket。

- 1.用accept4或者accept返回得到socket注意设置成非阻塞。如果设置非阻塞失败那么必须要回收连接池的连接并且关闭socket。
- 2.给新连接分配一个连接池内的连接ngx_get_connection。
- 3.连接池内的连接设置这个连接的处理函数
  newc->rhandler = &CSocekt::ngx_read_request_handler;  //设置已建立连接的socket当客户端发来数据来时的读处理函数
  newc->whandler = &CSocekt::ngx_write_request_handler; //设置已建立连接的socket的写处理函数
- 4.客户端应该主动发送第一次的数据，这里将读事件加入epoll监控`ngx_epoll_oper_event`，这样当客户端发送数据来时，ngx_read_request_handler()被ngx_epoll_process_events()调用

```cpp
//建立新连接专用函数，当新连接进入时，本函数会被ngx_epoll_process_events()所调用
void CSocekt::ngx_event_accept(lpngx_connection_t oldc)
{
    //因为listen套接字上用的不是ET【边缘触发】，而是LT【水平触发】，意味着客户端连入如果我要不处理，这个函数会被多次调用，所以，我这里这里可以不必多次accept()，可以只执行一次accept()
         //这也可以避免本函数被卡太久，注意，本函数应该尽快返回，以免阻塞程序运行；
    struct sockaddr    mysockaddr;        //远端服务器的socket地址
    socklen_t          socklen;
    int                err;
    int                level;
    int                s;
    static int         use_accept4 = 1;   //我们先认为能够使用accept4()函数
    lpngx_connection_t newc;              //代表连接池中的一个连接【注意这是指针】
    
    //ngx_log_stderr(0,"这是几个\n"); 这里会惊群，也就是说，epoll技术本身有惊群的问题

    socklen = sizeof(mysockaddr);
    do   //用do，跳到while后边去方便
    {     
        if(use_accept4)
        {
            //以为listen套接字是非阻塞的，所以即便已完成连接队列为空，accept4()也不会卡在这里；
            s = accept4(oldc->fd, &mysockaddr, &socklen, SOCK_NONBLOCK); //从内核获取一个用户端连接，最后一个参数SOCK_NONBLOCK表示返回一个非阻塞的socket，节省一次ioctl【设置为非阻塞】调用
        }
        else
        {
            //以为listen套接字是非阻塞的，所以即便已完成连接队列为空，accept()也不会卡在这里；
            s = accept(oldc->fd, &mysockaddr, &socklen);
        }

        //惊群，有时候不一定完全惊动所有4个worker进程，可能只惊动其中2个等等，其中一个成功其余的accept4()都会返回-1；错误 (11: Resource temporarily unavailable【资源暂时不可用】) 
        //所以参考资料：https://blog.csdn.net/russell_tao/article/details/7204260
        //其实，在linux2.6内核上，accept系统调用已经不存在惊群了（至少我在2.6.18内核版本上已经不存在）。大家可以写个简单的程序试下，在父进程中bind,listen，然后fork出子进程，
               //所有的子进程都accept这个监听句柄。这样，当新连接过来时，大家会发现，仅有一个子进程返回新建的连接，其他子进程继续休眠在accept调用上，没有被唤醒。
        //ngx_log_stderr(0,"测试惊群问题，看惊动几个worker进程%d\n",s); 【我的结论是：accept4可以认为基本解决惊群问题，但似乎并没有完全解决，有时候还会惊动其他的worker进程】


        if(s == -1)
        {
            err = errno;

            //对accept、send和recv而言，事件未发生时errno通常被设置成EAGAIN（意为“再来一次”）或者EWOULDBLOCK（意为“期待阻塞”）
            if(err == EAGAIN) //accept()没准备好，这个EAGAIN错误EWOULDBLOCK是一样的
            {
                //除非你用一个循环不断的accept()取走所有的连接，不然一般不会有这个错误【我们这里只取一个连接，也就是accept()一次】
                return ;
            } 
            level = NGX_LOG_ALERT;
            if (err == ECONNABORTED)  //ECONNRESET错误则发生在对方意外关闭套接字后【您的主机中的软件放弃了一个已建立的连接--由于超时或者其它失败而中止接连(用户插拔网线就可能有这个错误出现)】
            {
                //该错误被描述为“software caused connection abort”，即“软件引起的连接中止”。原因在于当服务和客户进程在完成用于 TCP 连接的“三次握手”后，
                    //客户 TCP 却发送了一个 RST （复位）分节，在服务进程看来，就在该连接已由 TCP 排队，等着服务进程调用 accept 的时候 RST 却到达了。
                    //POSIX 规定此时的 errno 值必须 ECONNABORTED。源自 Berkeley 的实现完全在内核中处理中止的连接，服务进程将永远不知道该中止的发生。
                        //服务器进程一般可以忽略该错误，直接再次调用accept。
                level = NGX_LOG_ERR;
            } 
            else if (err == EMFILE || err == ENFILE) //EMFILE:进程的fd已用尽【已达到系统所允许单一进程所能打开的文件/套接字总数】。可参考：https://blog.csdn.net/sdn_prc/article/details/28661661   以及 https://bbs.csdn.net/topics/390592927
                                                        //ulimit -n ,看看文件描述符限制,如果是1024的话，需要改大;  打开的文件句柄数过多 ,把系统的fd软限制和硬限制都抬高.
                                                    //ENFILE这个errno的存在，表明一定存在system-wide的resource limits，而不仅仅有process-specific的resource limits。按照常识，process-specific的resource limits，一定受限于system-wide的resource limits。
            {
                level = NGX_LOG_CRIT;
            }
            //ngx_log_error_core(level,errno,"CSocekt::ngx_event_accept()中accept4()失败!");

            if(use_accept4 && err == ENOSYS) //accept4()函数没实现，坑爹？
            {
                use_accept4 = 0;  //标记不使用accept4()函数，改用accept()函数
                continue;         //回去重新用accept()函数搞
            }

            if (err == ECONNABORTED)  //对方关闭套接字
            {
                //这个错误因为可以忽略，所以不用干啥
                //do nothing
            }
            
            if (err == EMFILE || err == ENFILE) 
            {
                //do nothing，这个官方做法是先把读事件从listen socket上移除，然后再弄个定时器，定时器到了则继续执行该函数，但是定时器到了有个标记，会把读事件增加到listen socket上去；
                //我这里目前先不处理吧【因为上边已经写这个日志了】；
            }            
            return;
        }  //end if(s == -1)

        //走到这里的，表示accept4()/accept()成功了        
        if(m_onlineUserCount >= m_worker_connections)  //用户连接数过多，要关闭该用户socket，因为现在也没分配连接，所以直接关闭即可
        {
            //ngx_log_stderr(0,"超出系统允许的最大连入用户数(最大允许连入数%d)，关闭连入请求(%d)。",m_worker_connections,s);  
            close(s);
            return ;
        }
        //如果某些恶意用户连上来发了1条数据就断，不断连接，会导致频繁调用ngx_get_connection()使用我们短时间内产生大量连接，危及本服务器安全
        if(m_connectionList.size() > (m_worker_connections * 5))
        {
            //比如你允许同时最大2048个连接，但连接池却有了 2048*5这么大的容量，这肯定是表示短时间内 产生大量连接/断开，因为我们的延迟回收机制，这里连接还在垃圾池里没有被回收
            if(m_freeconnectionList.size() < m_worker_connections)
            {
                //整个连接池这么大了，而空闲连接却这么少了，所以我认为是  短时间内 产生大量连接，发一个包后就断开，我们不可能让这种情况持续发生，所以必须断开新入用户的连接
                //一直到m_freeconnectionList变得足够大【连接池中连接被回收的足够多】
                close(s);
                return ;   
            }
        }

        //ngx_log_stderr(errno,"accept4成功s=%d",s); //s这里就是 一个句柄了
        newc = ngx_get_connection(s); //这是针对新连入用户的连接，和监听套接字 所对应的连接是两个不同的东西，不要搞混
        if(newc == NULL)
        {
            //连接池中连接不够用，那么就得把这个socekt直接关闭并返回了，因为在ngx_get_connection()中已经写日志了，所以这里不需要写日志了
            if(close(s) == -1)
            {
                ngx_log_error_core(NGX_LOG_ALERT,errno,"CSocekt::ngx_event_accept()中close(%d)失败!",s);                
            }
            return;
        }
        //...........将来这里会判断是否连接超过最大允许连接数，现在，这里可以不处理

        //成功的拿到了连接池中的一个连接
        memcpy(&newc->s_sockaddr,&mysockaddr,socklen);  //拷贝客户端地址到连接对象【要转成字符串ip地址参考函数ngx_sock_ntop()】
        //{
        //    //测试将收到的地址弄成字符串，格式形如"192.168.1.126:40904"或者"192.168.1.126"
        //    u_char ipaddr[100]; memset(ipaddr,0,sizeof(ipaddr));
        //    ngx_sock_ntop(&newc->s_sockaddr,1,ipaddr,sizeof(ipaddr)-10); //宽度给小点
        //    ngx_log_stderr(0,"ip信息为%s\n",ipaddr);
        //}

        if(!use_accept4)
        {
            //如果不是用accept4()取得的socket，那么就要设置为非阻塞【因为用accept4()的已经被accept4()设置为非阻塞了】
            if(setnonblocking(s) == false)
            {
                //设置非阻塞居然失败
                ngx_close_connection(newc); //关闭socket,这种可以立即回收这个连接，无需延迟，因为其上还没有数据收发，谈不到业务逻辑因此无需延迟；
                return; //直接返回
            }
        }

        newc->listening = oldc->listening;                    //连接对象 和监听对象关联，方便通过连接对象找监听对象【关联到监听端口】
        //newc->w_ready = 1;                                    //标记可以写，新连接写事件肯定是ready的；【从连接池拿出一个连接时这个连接的所有成员都是0】            
        
        newc->rhandler = &CSocekt::ngx_read_request_handler;  //设置数据来时的读处理函数，其实官方nginx中是ngx_http_wait_request_handler()
        newc->whandler = &CSocekt::ngx_write_request_handler; //设置数据发送时的写处理函数。

        //客户端应该主动发送第一次的数据，这里将读事件加入epoll监控，这样当客户端发送数据来时，会触发ngx_wait_request_handler()被ngx_epoll_process_events()调用        
        if(ngx_epoll_oper_event(
                                s,                  //socekt句柄
                                EPOLL_CTL_ADD,      //事件类型，这里是增加
                                EPOLLIN|EPOLLRDHUP, //标志，这里代表要增加的标志,EPOLLIN：可读，EPOLLRDHUP：TCP连接的远端关闭或者半关闭 ，如果边缘触发模式可以增加 EPOLLET
                                0,                  //对于事件类型为增加的，不需要这个参数
                                newc                //连接池中的连接
                                ) == -1)         
        {
            //增加事件失败，失败日志在ngx_epoll_add_event中写过了，因此这里不多写啥；
            ngx_close_connection(newc);//关闭socket,这种可以立即回收这个连接，无需延迟，因为其上还没有数据收发，谈不到业务逻辑因此无需延迟；
            return; //直接返回
        }


        if(m_ifkickTimeCount == 1)
        {
            AddToTimerQueue(newc);
        }
        ++m_onlineUserCount;  //连入用户数量+1        
        break;  //一般就是循环一次就跳出去
    } while (1);   

    return;
}
```

### 处理TCP连接客户端发来的数据ngx_read_request_handler

引入一个消息头【结构】`STRUC_MSG_HEADER`，用来记录一些额外信息，**可以用于处理过时包**
服务器 收包时，  收到： 包头+包体  ，我再额外附加一个消息头 ===》  消息头 + 包头 + 包体

```h
//消息头，引入的目的是当收到数据包时，额外记录一些内容以备将来使用
typedef struct _STRUC_MSG_HEADER
{
	lpngx_connection_t pConn;         //记录对应的链接，注意这是个指针
	uint64_t           iCurrsequence; //收到数据包时记录对应连接的序号，将来能用于比较是否连接已经作废用
	//......其他以后扩展	
}STRUC_MSG_HEADER,*LPSTRUC_MSG_HEADER;
```

对于连接池的每个连接都是有一个序号iCurrsequence，连接初始化的时候++iCurrsequence，连接释放的时候++iCurrsequence，因此当收到一个包的时候，将连接连接池的序号iCurrsequence记录在包的消息头中，当处理完这个包后想要发回给客户端的时候可以比较一下包的iCurrsequence与连接池的iCurrsequence是否一致，如果不一致说明连接已经断开了，丢弃即可。

因此我们需要new一块新内存专门用来存取`消息头 + 包头 + 包体`

```cpp
//来数据时候的处理，当连接上有数据来的时候，本函数会被ngx_epoll_process_events()所调用  ,官方的类似函数为ngx_http_wait_request_handler();
void CSocekt::ngx_read_request_handler(lpngx_connection_t pConn)
{  
    bool isflood = false; //是否flood攻击；

    //收包，注意我们用的第二个和第三个参数，我们用的始终是这两个参数，因此我们必须保证 c->precvbuf指向正确的收包位置，保证c->irecvlen指向正确的收包宽度
    ssize_t reco = recvproc(pConn,pConn->precvbuf,pConn->irecvlen); 
    if(reco <= 0)  
    {
        return;//该处理的上边这个recvproc()函数处理过了，这里<=0是直接return        
    }

    //走到这里，说明成功收到了一些字节（>0），就要开始判断收到了多少数据了     
    if(pConn->curStat == _PKG_HD_INIT) //连接建立起来时肯定是这个状态，因为在ngx_get_connection()中已经把curStat成员赋值成_PKG_HD_INIT了
    {        
        if(reco == m_iLenPkgHeader)//正好收到完整包头，这里拆解包头
        {   
            ngx_wait_request_handler_proc_p1(pConn,isflood); //那就调用专门针对包头处理完整的函数去处理把。
        }
        else
		{
			//收到的包头不完整--我们不能预料每个包的长度，也不能预料各种拆包/粘包情况，所以收到不完整包头【也算是缺包】是很可能的；
            pConn->curStat        = _PKG_HD_RECVING;                 //接收包头中，包头不完整，继续接收包头中	
            pConn->precvbuf       = pConn->precvbuf + reco;              //注意收后续包的内存往后走
            pConn->irecvlen       = pConn->irecvlen - reco;              //要收的内容当然要减少，以确保只收到完整的包头先
        } //end  if(reco == m_iLenPkgHeader)
    } 
    else if(pConn->curStat == _PKG_HD_RECVING) //接收包头中，包头不完整，继续接收中，这个条件才会成立
    {
        if(pConn->irecvlen == reco) //要求收到的宽度和我实际收到的宽度相等
        {
            //包头收完整了
            ngx_wait_request_handler_proc_p1(pConn,isflood); //那就调用专门针对包头处理完整的函数去处理把。
        }
        else
		{
			//包头还是没收完整，继续收包头
            //pConn->curStat        = _PKG_HD_RECVING;                 //没必要
            pConn->precvbuf       = pConn->precvbuf + reco;              //注意收后续包的内存往后走
            pConn->irecvlen       = pConn->irecvlen - reco;              //要收的内容当然要减少，以确保只收到完整的包头先
        }
    }
    else if(pConn->curStat == _PKG_BD_INIT) 
    {
        //包头刚好收完，准备接收包体
        if(reco == pConn->irecvlen)
        {
            //收到的宽度等于要收的宽度，包体也收完整了
            if(m_floodAkEnable == 1) 
            {
                //Flood攻击检测是否开启
                isflood = TestFlood(pConn);
            }
            ngx_wait_request_handler_proc_plast(pConn,isflood);
        }
        else
		{
			//收到的宽度小于要收的宽度
			pConn->curStat = _PKG_BD_RECVING;					
			pConn->precvbuf = pConn->precvbuf + reco;
			pConn->irecvlen = pConn->irecvlen - reco;
		}
    }
    else if(pConn->curStat == _PKG_BD_RECVING) 
    {
        //接收包体中，包体不完整，继续接收中
        if(pConn->irecvlen == reco)
        {
            //包体收完整了
            if(m_floodAkEnable == 1) 
            {
                //Flood攻击检测是否开启
                isflood = TestFlood(pConn);
            }
            ngx_wait_request_handler_proc_plast(pConn,isflood);
        }
        else
        {
            //包体没收完整，继续收
            pConn->precvbuf = pConn->precvbuf + reco;
			pConn->irecvlen = pConn->irecvlen - reco;
        }
    }  //end if(c->curStat == _PKG_HD_INIT)

    if(isflood == true)
    {
        //客户端flood服务器，则直接把客户端踢掉
        //ngx_log_stderr(errno,"发现客户端flood，干掉该客户端!");
        zdClosesocketProc(pConn);
    }

    return;
}
```

注意这个函数调用了`recvproc(pConn,pConn->precvbuf,pConn->irecvlen);`最多只能收pConn->irecvlen个字节

注意采用了状态机，非常的稳健！！

#### recvproc

四次挥手关闭连接，RST强制关闭连接，正常发包都能检测。

**伪代码**

```c
recvproc(lpngx_connection_t pConn,char *buff,ssize_t buflen)
    n = recv(c->fd,buff,buflen,0);
    对返回值n判断，如果n的值异常，根据errno写相应日志或关闭socket，然后直接return -1;
        if n==0  zdClosesocketProc(pConn);return -1;//正常四次挥手，关闭连接
        if n<0
            if(errno == EAGAIN || errno == EWOULDBLOCK)ngx_log_stderr;return -1;//没有收到数据，不关闭连接
            if(errno == EINTR)ngx_log_stderr;return -1;//不算错误，不关闭连接

			//以下这种明显错误的必须关闭连接
            if(errno == ECONNRESET)do nothing//客户端没有四次挥手正常关闭socket连接，却关闭了整个运行程序，发RST包
            else ngx_log_stderr;//不知道什么错误，直接打印出来
            zdClosesocketProc(c);//关闭socket连接
            return -1;
	
	//以下是没有出现错误的
    n>0,返回实际收到的字节数
```

如果recv()有问题，这个函数会把该释放的释放好，该处理的处理好

**特别注意：**所有从10行开始走下来的错误，都认为异常：意味着我们要关闭客户端套接字要回收连接池中连接；

一定要有严密的判断

```cpp
//接收数据专用函数--引入这个函数是为了方便，如果断线，错误之类的，这里直接 释放连接池中连接，然后直接关闭socket，以免在其他函数中还要重复的干这些事
//参数c：连接池中相关连接
//参数buff：接收数据的缓冲区
//参数buflen：要接收的数据大小
//返回值：返回-1，则是有问题发生并且在这里把问题处理完毕了，调用本函数的调用者一般是可以直接return
//        返回>0，则是表示实际收到的字节数
ssize_t CSocekt::recvproc(lpngx_connection_t pConn,char *buff,ssize_t buflen)  //ssize_t是有符号整型，在32位机器上等同与int，在64位机器上等同与long int，size_t就是无符号型的ssize_t
{
    ssize_t n;
    
    n = recv(pConn->fd, buff, buflen, 0); //recv()系统函数， 最后一个参数flag，一般为0；     
    if(n == 0)
    {
        //客户端关闭【应该是正常完成了4次挥手】，我这边就直接回收连接，关闭socket即可 
        zdClosesocketProc(pConn);        
        return -1;
    }
    //客户端没断，走这里 
    if(n < 0) //这被认为有错误发生
    {
        //EAGAIN和EWOULDBLOCK[【这个应该常用在hp上】应该是一样的值，表示没收到数据，一般来讲，在ET模式下会出现这个错误，因为ET模式下是不停的recv肯定有一个时刻收到这个errno，但LT模式下一般是来事件才收，所以不该出现这个返回值
        if(errno == EAGAIN || errno == EWOULDBLOCK)
        {
            //我认为LT模式不该出现这个errno，而且这个其实也不是错误，所以不当做错误处理
            ngx_log_stderr(errno,"CSocekt::recvproc()中errno == EAGAIN || errno == EWOULDBLOCK成立，出乎我意料！");//epoll为LT模式不应该出现这个返回值，所以直接打印出来瞧瞧
            return -1; //不当做错误处理，只是简单返回
        }
        //EINTR错误的产生：当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。
        //例如：在socket服务器端，设置了信号捕获机制，有子进程，当在父进程阻塞于慢系统调用时由父进程捕获到了一个有效信号时，内核会致使accept返回一个EINTR错误(被中断的系统调用)。
        if(errno == EINTR)  //这个不算错误，是我参考官方nginx，官方nginx这个就不算错误；
        {
            //我认为LT模式不该出现这个errno，而且这个其实也不是错误，所以不当做错误处理
            ngx_log_stderr(errno,"CSocekt::recvproc()中errno == EINTR成立，出乎我意料！");//epoll为LT模式不应该出现这个返回值，所以直接打印出来瞧瞧
            return -1; //不当做错误处理，只是简单返回
        }

        //所有从这里走下来的错误，都认为异常：意味着我们要关闭客户端套接字要回收连接池中连接；

        //errno参考：http://dhfapiran1.360drm.com        
        if(errno == ECONNRESET)  //#define ECONNRESET 104 /* Connection reset by peer */
        {
            //如果客户端没有正常关闭socket连接，却关闭了整个运行程序【真是够粗暴无理的，应该是直接给服务器发送rst包而不是4次挥手包完成连接断开】，那么会产生这个错误            
            //10054(WSAECONNRESET)--远程程序正在连接的时候关闭会产生这个错误--远程主机强迫关闭了一个现有的连接
            //算常规错误吧【普通信息型】，日志都不用打印，没啥意思，太普通的错误
            //do nothing

            //....一些大家遇到的很普通的错误信息，也可以往这里增加各种，代码要慢慢完善，一步到位，不可能，很多服务器程序经过很多年的完善才比较圆满；
        }
        else
        {
            //能走到这里的，都表示错误，我打印一下日志，希望知道一下是啥错误，我准备打印到屏幕上
            if(errno == EBADF)  // #define EBADF   9 /* Bad file descriptor */
            {
                //因为多线程，偶尔会干掉socket，所以不排除产生这个错误的可能性
            }
            else
            {
                ngx_log_stderr(errno,"CSocekt::recvproc()中发生错误，我打印出来看看是啥错误！");  //正式运营时可以考虑这些日志打印去掉
            }
        } 
        
        //ngx_log_stderr(0,"连接被客户端 非 正常关闭！");

        //这种真正的错误就要，直接关闭套接字，释放连接池中连接了
        //ngx_close_connection(pConn);
        //inRecyConnectQueue(pConn);
        zdClosesocketProc(pConn);
        return -1;
    }

    //能走到这里的，就认为收到了有效数据
    return n; //返回收到的字节数
}
```

#### ngx_wait_request_handler_proc_p1

包头收完整后的处理，我们称为包处理阶段1

- 1.判断包是否合法，若不合法则状态机恢复为_PKG_HD_INIT，并且连接池的缓冲区头指针重新指定为最开始的
- 2.那么`char *pTmpBuffer  = (char *)p_memory->AllocMemory(m_iLenMsgHeader + e_pkgLen,false);`新分配一段内存并且将连接池的成员指针`pConn->precvMemPointer = pTmpBuffer;`指向这块内存（消息头加整个包长度的内存 ）
- 3.填写消息头的内容，把连接池中连接序号记录到消息头里来，连接池的连接指针也记录到消息头里
- 4.再填写包头内容，把包体内容拷贝到pTmpBuffer中
- 5.如果该报文只有包头无包体，调用ngx_wait_request_handler_proc_plast()收完整包后的处理函数。
- 6.如果该报文还需要继续收包体，修改状态机为_PKG_BD_INIT，并且连接池的缓冲区头指针指向包体。

```cpp
//包头收完整后的处理，我们称为包处理阶段1【p1】：写成函数，方便复用
//注意参数isflood是个引用
void CSocekt::ngx_wait_request_handler_proc_p1(lpngx_connection_t pConn,bool &isflood)
{    
    CMemory *p_memory = CMemory::GetInstance();		

    LPCOMM_PKG_HEADER pPkgHeader;
    pPkgHeader = (LPCOMM_PKG_HEADER)pConn->dataHeadInfo; //正好收到包头时，包头信息肯定是在dataHeadInfo里；

    unsigned short e_pkgLen; 
    e_pkgLen = ntohs(pPkgHeader->pkgLen);  //注意这里网络序转本机序，所有传输到网络上的2字节数据，都要用htons()转成网络序，所有从网络上收到的2字节数据，都要用ntohs()转成本机序
                                                //ntohs/htons的目的就是保证不同操作系统数据之间收发的正确性，【不管客户端/服务器是什么操作系统，发送的数字是多少，收到的就是多少】                                               
    //恶意包或者错误包的判断
    if(e_pkgLen < m_iLenPkgHeader) 
    {
        //伪造包/或者包错误，否则整个包长怎么可能比包头还小？（整个包长是包头+包体，就算包体为0字节，那么至少e_pkgLen == m_iLenPkgHeader）
        //报文总长度 < 包头长度，认定非法用户，废包
        //状态和接收位置都复原，这些值都有必要，因为有可能在其他状态比如_PKG_HD_RECVING状态调用这个函数；
        pConn->curStat = _PKG_HD_INIT;      
        pConn->precvbuf = pConn->dataHeadInfo;
        pConn->irecvlen = m_iLenPkgHeader;
    }
    else if(e_pkgLen > (_PKG_MAX_LENGTH-1000))   //客户端发来包居然说包长度 > 29000?肯定是恶意包
    {
        //恶意包，太大，认定非法用户，废包【包头中说这个包总长度这么大，这不行】
        //状态和接收位置都复原，这些值都有必要，因为有可能在其他状态比如_PKG_HD_RECVING状态调用这个函数；
        pConn->curStat = _PKG_HD_INIT;
        pConn->precvbuf = pConn->dataHeadInfo;
        pConn->irecvlen = m_iLenPkgHeader;
    }
    else
    {
        //合法的包头，继续处理
        //我现在要分配内存开始收包体，因为包体长度并不是固定的，所以内存肯定要new出来；
        char *pTmpBuffer  = (char *)p_memory->AllocMemory(m_iLenMsgHeader + e_pkgLen,false); //分配内存【长度是 消息头长度  + 包头长度 + 包体长度】，最后参数先给false，表示内存不需要memset;        
        pConn->precvMemPointer = pTmpBuffer;  //内存开始指针

        //a)先填写消息头内容
        LPSTRUC_MSG_HEADER ptmpMsgHeader = (LPSTRUC_MSG_HEADER)pTmpBuffer;
        ptmpMsgHeader->pConn = pConn;
        ptmpMsgHeader->iCurrsequence = pConn->iCurrsequence; //收到包时的连接池中连接序号记录到消息头里来，以备将来用；
        //b)再填写包头内容
        pTmpBuffer += m_iLenMsgHeader;                 //往后跳，跳过消息头，指向包头
        memcpy(pTmpBuffer,pPkgHeader,m_iLenPkgHeader); //直接把收到的包头拷贝进来
        if(e_pkgLen == m_iLenPkgHeader)
        {
            //该报文只有包头无包体【我们允许一个包只有包头，没有包体】
            //这相当于收完整了，则直接入消息队列待后续业务逻辑线程去处理吧
            if(m_floodAkEnable == 1) 
            {
                //Flood攻击检测是否开启
                isflood = TestFlood(pConn);
            }
            ngx_wait_request_handler_proc_plast(pConn,isflood);
        } 
        else
        {
            //开始收包体，注意我的写法
            pConn->curStat = _PKG_BD_INIT;                   //当前状态发生改变，包头刚好收完，准备接收包体	   
            pConn->precvbuf = pTmpBuffer + m_iLenPkgHeader;  //pTmpBuffer指向包头，这里 + m_iLenPkgHeader后指向包体 weizhi
            pConn->irecvlen = e_pkgLen - m_iLenPkgHeader;    //e_pkgLen是整个包【包头+包体】大小，-m_iLenPkgHeader【包头】  = 包体
        }                       
    }  //end if(e_pkgLen < m_iLenPkgHeader) 

    return;
}
```

#### ngx_wait_request_handler_proc_plast

如果包没问题那么就**入消息队列并触发线程处理消息**，注意进入
并且要将**当前连接的状态机**恢复为_PKG_HD_INIT，并且连接池的缓冲区头指针重新指定为最开始的，且将precvMemPointer指针置为NULL（原先指向消息头加整个包的内存）

```cpp
//收到一个完整包后的处理【plast表示最后阶段】，放到一个函数中，方便调用
//注意参数isflood是个引用
void CSocekt::ngx_wait_request_handler_proc_plast(lpngx_connection_t pConn,bool &isflood)
{
    if(isflood == false)
    {
        g_threadpool.inMsgRecvQueueAndSignal(pConn->precvMemPointer); //入消息队列并触发线程处理消息
    }
    else
    {
        //对于有攻击倾向的恶人，先把他的包丢掉
        CMemory *p_memory = CMemory::GetInstance();
        p_memory->FreeMemory(pConn->precvMemPointer); //直接释放掉内存，根本不往消息队列入
    }

    pConn->precvMemPointer = NULL;
    pConn->curStat         = _PKG_HD_INIT;     //收包状态机的状态恢复为原始态，为收下一个包做准备                    
    pConn->precvbuf        = pConn->dataHeadInfo;  //设置好收包的位置
    pConn->irecvlen        = m_iLenPkgHeader;  //设置好要接收数据的大小
    return;
}
```

#### inMsgRecvQueueAndSignal

注意传入的参数是消息头+包头+包体

将这一个完整消息放入**线程池中的线程**的接收数据消息队列里去

并且调用Call**激发线程来干活**

```cpp
//收到一个完整消息后，入消息队列，并触发线程池中线程来处理该消息
void CThreadPool::inMsgRecvQueueAndSignal(char *buf)
{
    //互斥
    int err = pthread_mutex_lock(&m_pthreadMutex);     
    if(err != 0)
    {
        ngx_log_stderr(err,"CThreadPool::inMsgRecvQueueAndSignal()pthread_mutex_lock()失败，返回的错误码为%d!",err);
    }
        
    m_MsgRecvQueue.push_back(buf);	         //入消息队列
    ++m_iRecvMsgQueueCount;                  //收消息队列数字+1，个人认为用变量更方便一点，比 m_MsgRecvQueue.size()高效

    //取消互斥
    err = pthread_mutex_unlock(&m_pthreadMutex);   
    if(err != 0)
    {
        ngx_log_stderr(err,"CThreadPool::inMsgRecvQueueAndSignal()pthread_mutex_unlock()失败，返回的错误码为%d!",err);
    }

    //可以激发一个线程来干活了
    Call();                                  
    return;
}
```

#### Call

pthread_cond_signal唤醒一个等待该条件的线程，也就是可以唤醒卡在`pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex)`的线程

```cpp
while ( (pThreadPoolObj->m_MsgRecvQueue.size() == 0) && m_shutdown == false) {
    if(pThread->ifrunning == false)            
        pThread->ifrunning = true; 
    pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); 
}
```

如果拿得到消息即`m_MsgRecvQueue.size()`大小不为0，那么其中1个线程就可以拿到锁并且退出while循环，退出while循环去取消息并且调用相应的消息处理函数。

```cpp
//来任务了，调一个线程池中的线程下来干活
void CThreadPool::Call()
{
    int err = pthread_cond_signal(&m_pthreadCond); //唤醒一个等待该条件的线程，也就是可以唤醒卡在pthread_cond_wait()的线程
    if(err != 0 )
    {
        //这是有问题啊，要打印日志啊
        ngx_log_stderr(err,"CThreadPool::Call()中pthread_cond_signal()失败，返回的错误码为%d!",err);
    }
    
    //(1)如果当前的工作线程全部都忙，则要报警
    //bool ifallthreadbusy = false;
    if(m_iThreadNum == m_iRunningThreadNum) //线程池中线程总量，跟当前正在干活的线程数量一样，说明所有线程都忙碌起来，线程不够用了
    {        
        //线程不够用了
        //ifallthreadbusy = true;
        time_t currtime = time(NULL);
        if(currtime - m_iLastEmgTime > 10) //最少间隔10秒钟才报一次线程池中线程不够用的问题；
        {
            //两次报告之间的间隔必须超过10秒，不然如果一直出现当前工作线程全忙，但频繁报告日志也够烦的
            m_iLastEmgTime = currtime;  //更新时间
            //写日志，通知这种紧急情况给用户，用户要考虑增加线程池中线程数量了
            ngx_log_stderr(0,"CThreadPool::Call()中发现线程池中当前空闲线程数量为0，要考虑扩容线程池了!");
        }
    } //end if 
    return;
}
```

### 处理TCP连接发送数据

**什么叫socekt可写？**
每一个tcp连接(socket)，都会有一个接收缓冲区 和 一个发送缓冲；
发送缓冲区缺省大小一般10几k，接收缓冲区大概几十k，setsocketopt()来设置；

send()【Windows端】,write()【linux】发送数据时，实际上这两个函数是把数据放到了发送缓冲区，之后这两个函数返回了；【这两个函数执行成功不代表客户端已经成功收到数据，只有在客户端用了recv()或read()收到数据后，告诉服务器已经确认收到了数据，这样服务器才会把发送缓冲区中这些数据清空】

如果服务器端的发送 缓冲区满了，那么服务器再调用send(),write()发送数据的时候，那么send(),write()函数就会返回一个EAGAIN；；EAGAIN不是一个错误，只是示意发送缓冲区已经满了，迟一些再调用send(),write()来发送数据吧；

针对 当socket可写的时候【发送缓冲区没满】，会不停的触发socket可写事件 ,我们提出两种解决方案【面试可能考试】；

两种解决方案，来自网络,意义在于我们可以通过这种解决方案来指导我们写代码；

- 第一种最普遍的解决方案:

需要向socket写数据的时候把socket写事件通知加入到epoll中，等待可写事件，当可写事件来时操作系统会通知咱们；此时咱们可以调用wirte/send函数发送数据，当发送数据完毕后，把socket的写事件通知从红黑树中移除；
缺点：即使发送很少的数据，也需要把事件通知加入到epoll，写完毕后，有需要把写事件通知从红黑树干掉,对效率有一定的影响【有一定的操作代价】

- 改进方案：

开始不把socket写事件通知加入到epoll,当我需要写数据的时候，**直接调用write/send发送数据**，如果能全部发送完那么最好；

但如果**发送缓冲区满了send()返回了EAGIN**无法一次性全部发送完，此时，我再把写事件通知加入到epoll，此时，就变成了在epoll驱动下写数据

当发送缓冲区可写（即有空间了）的时候，ngx_epoll_process_events函数中epoll_wait会通知有可写事件，这个时候我们调用ngx_write_request_handler来处理可写事件，当全部数据发送完毕后，再把写事件通知从epoll中干掉；

优点：数据不多的时候，可以避免epoll的写事件的增加/删除，提高了程序的执行效率；

#### msgSend待发送消息入到发消息队列

主要的逻辑：

1.CSocket::m_MsgSendQueue.push_back(psendbuf);将消息放入待发送消息队列中

2.sem_post(&m_semEventSendQueue)让ServerSendQueueThread()流程走下来干活

**同步原理：**

`CSocekt::Initialize_subproc`函数中初始化调用`sem_init(&m_semEventSendQueue,0,0)`对此信号量进行初始化为0。

sem_wait()：测试指定信号量的值，如果该值>0，那么将该值**减1**然后该函数立即返回；如果该值 等于0，那么该线程将投入睡眠中，一直到该值>0，这个时候  那么将该值**减1**然后该函数立即返回；  

semd_post()：能够将指定信号量值**加1**，即便当前没有其他线程在等待该信号量值也没关系；这也就保证了信号量不会丢失

```cpp
//将一个待发送消息入到发消息队列中
void CSocekt::msgSend(char *psendbuf) 
{
    CMemory *p_memory = CMemory::GetInstance();

    CLock lock(&m_sendMessageQueueMutex);  //互斥量

    //发送消息队列过大也可能给服务器带来风险
    if(m_iSendMsgQueueCount > 50000)
    {
        //发送队列过大，比如客户端恶意不接受数据，就会导致这个队列越来越大
        //那么可以考虑为了服务器安全，干掉一些数据的发送，虽然有可能导致客户端出现问题，但总比服务器不稳定要好很多
        m_iDiscardSendPkgCount++;
        p_memory->FreeMemory(psendbuf);
		return;
    }
    
    //总体数据并无风险，不会导致服务器崩溃，要看看个体数据，找一下恶意者了    
    LPSTRUC_MSG_HEADER pMsgHeader = (LPSTRUC_MSG_HEADER)psendbuf;
	lpngx_connection_t p_Conn = pMsgHeader->pConn;
    if(p_Conn->iSendCount > 400)
    {
        //该用户收消息太慢【或者干脆不收消息】，累积的该用户的发送队列中有的数据条目数过大，认为是恶意用户，直接切断
        ngx_log_stderr(0,"CSocekt::msgSend()中发现某用户%d积压了大量待发送数据包，切断与他的连接！",p_Conn->fd);      
        m_iDiscardSendPkgCount++;
        p_memory->FreeMemory(psendbuf);
        zdClosesocketProc(p_Conn); //直接关闭
		return;
    }

    ++p_Conn->iSendCount; //发送队列中有的数据条目数+1；
    m_MsgSendQueue.push_back(psendbuf);     
    ++m_iSendMsgQueueCount;   //原子操作

    //将信号量的值+1,这样其他卡在sem_wait的就可以走下去
    if(sem_post(&m_semEventSendQueue)==-1)  //让ServerSendQueueThread()流程走下来干活
    {
        ngx_log_stderr(0,"CSocekt::msgSend()中sem_post(&m_semEventSendQueue)失败.");      
    }
    return;
}
```

#### ServerSendQueueThread处理发送消息队列的线程*

pthread_mutex_lock(&pSocketObj->m_sendMessageQueueMutex); //因为我们要操作发送消息对列

发送消息，如果发送缓冲区满了，则需要通过epoll事件来驱动消息的继续发送，所以如果发送缓冲区满，则用这个连接池成员变量`ngx_connection_s::atomic<int>iThrowsendCount;`标记是依靠epoll来驱动的

- 1.sendsize>0且成功发送了整个包的所有数据，一下就发送出去这很顺利，那么把堆里面的存放消息的那块内存释放掉即可
- 2.sendsize>0但是没有全部发送完毕(EAGAIN)，数据只发出去了一部分，但肯定是因为 发送缓冲区满了，EPOLL_MOD给当前连接增加一个监听可写事件
- 3.sendsize == 0对方断开了，对方断开这个事件属于可读事件，会再recvproc函数中处理
- 4.sendsize == -1表明 errno == EAGAIN ，服务器的发送缓冲区满了，那么和第2种清空一样，需要通过EPOLL_MOD给当前连接增加一个监听可写事件
- 5.sendsize == -2，一般我认为都是对端断开的错误，对端断开需要把堆里面的存放消息的那块内存释放掉即可，操作系统会帮我们将这个socket连接从红黑树中移除掉

```cpp
//处理发送消息队列的线程
void* CSocekt::ServerSendQueueThread(void* threadData)
{    
    ThreadItem *pThread = static_cast<ThreadItem*>(threadData);
    CSocekt *pSocketObj = pThread->_pThis;
    int err;
    std::list <char *>::iterator pos,pos2,posend;
    
    char *pMsgBuf;	
    LPSTRUC_MSG_HEADER	pMsgHeader;
	LPCOMM_PKG_HEADER   pPkgHeader;
    lpngx_connection_t  p_Conn;
    unsigned short      itmp;
    ssize_t             sendsize;  

    CMemory *p_memory = CMemory::GetInstance();
    
    while(g_stopEvent == 0) //不退出
    {
        //如果信号量值>0，则 -1(减1) 并走下去，否则卡这里卡着【为了让信号量值+1，可以在其他线程调用sem_post达到，实际上在CSocekt::msgSend()调用sem_post就达到了让这里sem_wait走下去的目的】
        //******如果被某个信号中断，sem_wait也可能过早的返回，错误为EINTR；
        //整个程序退出之前，也要sem_post()一下，确保如果本线程卡在sem_wait()，也能走下去从而让本线程成功返回
        if(sem_wait(&pSocketObj->m_semEventSendQueue) == -1)
        {
            //失败？及时报告，其他的也不好干啥
            if(errno != EINTR) //这个我就不算个错误了【当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。】
                ngx_log_stderr(errno,"CSocekt::ServerSendQueueThread()中sem_wait(&pSocketObj->m_semEventSendQueue)失败.");            
        }

        //一般走到这里都表示需要处理数据收发了
        if(g_stopEvent != 0)  //要求整个进程退出
            break;

        if(pSocketObj->m_iSendMsgQueueCount > 0) //原子的 
        {
            err = pthread_mutex_lock(&pSocketObj->m_sendMessageQueueMutex); //因为我们要操作发送消息对列m_MsgSendQueue，所以这里要临界            
            if(err != 0) ngx_log_stderr(err,"CSocekt::ServerSendQueueThread()中pthread_mutex_lock()失败，返回的错误码为%d!",err);

            pos    = pSocketObj->m_MsgSendQueue.begin();
			posend = pSocketObj->m_MsgSendQueue.end();

            while(pos != posend)
            {
                pMsgBuf = (*pos);                          //拿到的每个消息都是 消息头+包头+包体【但要注意，我们是不发送消息头给客户端的】
                pMsgHeader = (LPSTRUC_MSG_HEADER)pMsgBuf;  //指向消息头
                pPkgHeader = (LPCOMM_PKG_HEADER)(pMsgBuf+pSocketObj->m_iLenMsgHeader);	//指向包头
                p_Conn = pMsgHeader->pConn;

                //包过期，因为如果 这个连接被回收，比如在ngx_close_connection(),inRecyConnectQueue()中都会自增iCurrsequence
                     //而且这里有没必要针对 本连接 来用m_connectionMutex临界 ,只要下面条件成立，肯定是客户端连接已断，要发送的数据肯定不需要发送了
                if(p_Conn->iCurrsequence != pMsgHeader->iCurrsequence) 
                {
                    //本包中保存的序列号与p_Conn【连接池中连接】中实际的序列号已经不同，丢弃此消息，小心处理该消息的删除
                    pos2=pos;
                    pos++;
                    pSocketObj->m_MsgSendQueue.erase(pos2);
                    --pSocketObj->m_iSendMsgQueueCount; //发送消息队列容量少1		
                    p_memory->FreeMemory(pMsgBuf);	
                    continue;
                } //end if

                if(p_Conn->iThrowsendCount > 0) 
                {
                    //靠系统驱动来发送消息，所以这里不能再发送
                    pos++;
                    continue;
                }

                --p_Conn->iSendCount;   //发送队列中有的数据条目数-1；
            
                //走到这里，可以发送消息，一些必须的信息记录，要发送的东西也要从发送队列里干掉
                p_Conn->psendMemPointer = pMsgBuf;      //发送后释放用的，因为这段内存是new出来的
                pos2=pos;
				pos++;
                pSocketObj->m_MsgSendQueue.erase(pos2);
                --pSocketObj->m_iSendMsgQueueCount;      //发送消息队列容量少1	
                p_Conn->psendbuf = (char *)pPkgHeader;   //要发送的数据的缓冲区指针，因为发送数据不一定全部都能发送出去，我们要记录数据发送到了哪里，需要知道下次数据从哪里开始发送
                itmp = ntohs(pPkgHeader->pkgLen);        //包头+包体 长度 ，打包时用了htons【本机序转网络序】，所以这里为了得到该数值，用了个ntohs【网络序转本机序】；
                p_Conn->isendlen = itmp;                 //要发送多少数据，因为发送数据不一定全部都能发送出去，我们需要知道剩余有多少数据还没发送
                                
                //这里是重点，我们采用 epoll水平触发的策略，能走到这里的，都应该是还没有投递 写事件 到epoll中
                    //epoll水平触发发送数据的改进方案：
	                //开始不把socket写事件通知加入到epoll,当我需要写数据的时候，直接调用write/send发送数据；
	                //如果返回了EAGIN【发送缓冲区满了，需要等待可写事件才能继续往缓冲区里写数据】，此时，我再把写事件通知加入到epoll，
	                //此时，就变成了在epoll驱动下写数据，全部数据发送完毕后，再把写事件通知从epoll中干掉；
	                //优点：数据不多的时候，可以避免epoll的写事件的增加/删除，提高了程序的执行效率；                         
                //(1)直接调用write或者send发送数据
                //ngx_log_stderr(errno,"即将发送数据%ud。",p_Conn->isendlen);

                sendsize = pSocketObj->sendproc(p_Conn,p_Conn->psendbuf,p_Conn->isendlen); //注意参数
                if(sendsize > 0)
                {                    
                    if(sendsize == p_Conn->isendlen) //成功发送出去了数据，一下就发送出去这很顺利
                    {
                        //成功发送的和要求发送的数据相等，说明全部发送成功了 发送缓冲区去了【数据全部发完】
                        p_memory->FreeMemory(p_Conn->psendMemPointer);  //释放内存
                        p_Conn->psendMemPointer = NULL;
                        p_Conn->iThrowsendCount = 0;  //这行其实可以没有，因此此时此刻这东西就是=0的                        
                        //ngx_log_stderr(0,"CSocekt::ServerSendQueueThread()中数据发送完毕，很好。"); //做个提示吧，商用时可以干掉
                    }
                    else  //没有全部发送完毕(EAGAIN)，数据只发出去了一部分，但肯定是因为 发送缓冲区满了,那么
                    {                        
                        //发送到了哪里，剩余多少，记录下来，方便下次sendproc()时使用
                        p_Conn->psendbuf = p_Conn->psendbuf + sendsize;
				        p_Conn->isendlen = p_Conn->isendlen - sendsize;	
                        //因为发送缓冲区慢了，所以 现在我要依赖系统通知来发送数据了
                        ++p_Conn->iThrowsendCount;             //标记发送缓冲区满了，需要通过epoll事件来驱动消息的继续发送【原子+1，且不可写成p_Conn->iThrowsendCount = p_Conn->iThrowsendCount +1 ，这种写法不是原子+1】
                        //投递此事件后，我们将依靠epoll驱动调用ngx_write_request_handler()函数发送数据
                        if(pSocketObj->ngx_epoll_oper_event(
                                p_Conn->fd,         //socket句柄
                                EPOLL_CTL_MOD,      //事件类型，这里是增加【因为我们准备增加个写通知】
                                EPOLLOUT,           //标志，这里代表要增加的标志,EPOLLOUT：可写【可写的时候通知我】
                                0,                  //对于事件类型为增加的，EPOLL_CTL_MOD需要这个参数, 0：增加   1：去掉 2：完全覆盖
                                p_Conn              //连接池中的连接
                                ) == -1)
                        {
                            //有这情况发生？这可比较麻烦，不过先do nothing
                            ngx_log_stderr(errno,"CSocekt::ServerSendQueueThread()ngx_epoll_oper_event()失败.");
                        }

                        //ngx_log_stderr(errno,"CSocekt::ServerSendQueueThread()中数据没发送完毕【发送缓冲区满】，整个要发送%d，实际发送了%d。",p_Conn->isendlen,sendsize);

                    } //end if(sendsize > 0)
                    continue;  //继续处理其他消息                    
                }  //end if(sendsize > 0)

                //能走到这里，应该是有点问题的
                else if(sendsize == 0)
                {
                    //发送0个字节，首先因为我发送的内容不是0个字节的；
                    //然后如果发送 缓冲区满则返回的应该是-1，而错误码应该是EAGAIN，所以我综合认为，这种情况我就把这个发送的包丢弃了【按对端关闭了socket处理】
                    //这个打印下日志，我还真想观察观察是否真有这种现象发生
                    //ngx_log_stderr(errno,"CSocekt::ServerSendQueueThread()中sendproc()居然返回0？"); //如果对方关闭连接出现send=0，那么这个日志可能会常出现，商用时就 应该干掉
                    //然后这个包干掉，不发送了
                    p_memory->FreeMemory(p_Conn->psendMemPointer);  //释放内存
                    p_Conn->psendMemPointer = NULL;
                    p_Conn->iThrowsendCount = 0;  //这行其实可以没有，因此此时此刻这东西就是=0的    
                    continue;
                }

                //能走到这里，继续处理问题
                else if(sendsize == -1)
                {
                    //发送缓冲区已经满了【一个字节都没发出去，说明发送 缓冲区当前正好是满的】
                    ++p_Conn->iThrowsendCount; //标记发送缓冲区满了，需要通过epoll事件来驱动消息的继续发送
                    //投递此事件后，我们将依靠epoll驱动调用ngx_write_request_handler()函数发送数据
                    if(pSocketObj->ngx_epoll_oper_event(
                                p_Conn->fd,         //socket句柄
                                EPOLL_CTL_MOD,      //事件类型，这里是增加【因为我们准备增加个写通知】
                                EPOLLOUT,           //标志，这里代表要增加的标志,EPOLLOUT：可写【可写的时候通知我】
                                0,                  //对于事件类型为增加的，EPOLL_CTL_MOD需要这个参数, 0：增加   1：去掉 2：完全覆盖
                                p_Conn              //连接池中的连接
                                ) == -1)
                    {
                        //有这情况发生？这可比较麻烦，不过先do nothing
                        ngx_log_stderr(errno,"CSocekt::ServerSendQueueThread()中ngx_epoll_add_event()_2失败.");
                    }
                    continue;
                }

                else
                {
                    //能走到这里的，应该就是返回值-2了，一般就认为对端断开了，等待recv()来做断开socket以及回收资源
                    p_memory->FreeMemory(p_Conn->psendMemPointer);  //释放内存
                    p_Conn->psendMemPointer = NULL;
                    p_Conn->iThrowsendCount = 0;  //这行其实可以没有，因此此时此刻这东西就是=0的  
                    continue;
                }

            } //end while(pos != posend)

            err = pthread_mutex_unlock(&pSocketObj->m_sendMessageQueueMutex); 
            if(err != 0)  ngx_log_stderr(err,"CSocekt::ServerSendQueueThread()pthread_mutex_unlock()失败，返回的错误码为%d!",err);
            
        } //if(pSocketObj->m_iSendMsgQueueCount > 0)
    } //end while
    
    return (void*)0;
}
```

#### sendproc发送数据专用函数

```cpp
//发送数据专用函数，返回本次发送的字节数
//返回 > 0，成功发送了一些字节
//=0，估计对方断了
//-1，errno == EAGAIN ，本方发送缓冲区满了
//-2，errno != EAGAIN != EWOULDBLOCK != EINTR ，一般我认为都是对端断开的错误
ssize_t CSocekt::sendproc(lpngx_connection_t c,char *buff,ssize_t size)  //ssize_t是有符号整型，在32位机器上等同与int，在64位机器上等同与long int，size_t就是无符号型的ssize_t
{
    //这里参考借鉴了官方nginx函数ngx_unix_send()的写法
    ssize_t   n;

    for ( ;; )
    {
        n = send(c->fd, buff, size, 0); //send()系统函数， 最后一个参数flag，一般为0； 
        if(n > 0) //成功发送了一些数据
        {        
            //发送成功一些数据，但发送了多少，我们这里不关心，也不需要再次send
            //这里有两种情况
            //(1) n == size也就是想发送多少都发送成功了，这表示完全发完毕了
            //(2) n < size 没发送完毕，那肯定是发送缓冲区满了，所以也不必要重试发送，直接返回吧
            return n; //返回本次发送的字节数
        }

        if(n == 0)
        {
            //send()返回0？ 一般recv()返回0表示断开,send()返回0，我这里就直接返回0吧【让调用者处理】；我个人认为send()返回0，要么你发送的字节是0，要么对端可能断开。
            //网上找资料：send=0表示超时，对方主动关闭了连接过程
            //我们写代码要遵循一个原则，连接断开，我们并不在send动作里处理诸如关闭socket这种动作，集中到recv那里处理，否则send,recv都处理都处理连接断开关闭socket则会乱套
            //连接断开epoll会通知并且 recvproc()里会处理，不在这里处理
            return 0;
        }

        if(errno == EAGAIN)  //这东西应该等于EWOULDBLOCK
        {
            //内核缓冲区满，这个不算错误
            return -1;  //表示发送缓冲区满了
        }

        if(errno == EINTR) 
        {
            //这个应该也不算错误 ，收到某个信号导致send产生这个错误？
            //参考官方的写法，打印个日志，其他啥也没干，那就是等下次for循环重新send试一次了
            ngx_log_stderr(errno,"CSocekt::sendproc()中send()失败.");  //打印个日志看看啥时候出这个错误
            //其他不需要做什么，等下次for循环吧            
        }
        else
        {
            //走到这里表示是其他错误码，都表示错误，错误我也不断开socket，我也依然等待recv()来统一处理断开，因为我是多线程，send()也处理断开，recv()也处理断开，很难处理好
            return -2;    
        }
    } //end for
}
```

#### ngx_write_request_handler(epoll通知后就调用这个函数)*

调用sendproc(pConn,pConn->psendbuf,pConn->isendlen);发送数据

- 1.数据只发送了一部分，**return返回**，之后如果发送缓冲区可写epoll还会通知
- 2.成功的发送完了所有的数据，就用EPOLL_MOD参数将写事件通知给去掉，epoll将不会继续监听写事件
- 3.数据发送完毕那么就可以sem_post(&m_semEventSendQueue)试着通知ServerSendQueueThread可以继续发送数据了
- 4.最后清空堆中存放消息的那块内存**return返回**

```cpp
//设置数据发送时的写处理函数,当数据可写时epoll通知我们，我们在 int CSocekt::ngx_epoll_process_events(int timer)  中调用此函数
//能走到这里，数据就是没法送完毕， 要继续发送
void CSocekt::ngx_write_request_handler(lpngx_connection_t pConn)
{      
    CMemory *p_memory = CMemory::GetInstance();
    
    //这些代码的书写可以参照 void* CSocekt::ServerSendQueueThread(void* threadData)
    ssize_t sendsize = sendproc(pConn,pConn->psendbuf,pConn->isendlen);

    if(sendsize > 0 && sendsize != pConn->isendlen)
    {        
        //没有全部发送完毕，数据只发出去了一部分，那么发送到了哪里，剩余多少，继续记录，方便下次sendproc()时使用
        pConn->psendbuf = pConn->psendbuf + sendsize;
		pConn->isendlen = pConn->isendlen - sendsize;	
        return;
    }
    else if(sendsize == -1)
    {
        //这不太可能，可以发送数据时通知我发送数据，我发送时你却通知我发送缓冲区满？
        ngx_log_stderr(errno,"CSocekt::ngx_write_request_handler()时if(sendsize == -1)成立，这很怪异。"); //打印个日志，别的先不干啥
        return;
    }

    if(sendsize > 0 && sendsize == pConn->isendlen) //成功发送完毕，做个通知是可以的；
    {
        //如果是成功的发送完毕数据，则把写事件通知从epoll中干掉吧；其他情况，那就是断线了，等着系统内核把连接从红黑树中干掉即可；
        if(ngx_epoll_oper_event(
                pConn->fd,          //socket句柄
                EPOLL_CTL_MOD,      //事件类型，这里是修改【因为我们准备减去写通知】
                EPOLLOUT,           //标志，这里代表要减去的标志,EPOLLOUT：可写【可写的时候通知我】
                1,                  //对于事件类型为增加的，EPOLL_CTL_MOD需要这个参数, 0：增加   1：去掉 2：完全覆盖
                pConn               //连接池中的连接
                ) == -1)
        {
            //有这情况发生？这可比较麻烦，不过先do nothing
            ngx_log_stderr(errno,"CSocekt::ngx_write_request_handler()中ngx_epoll_oper_event()失败。");
        }    

        //ngx_log_stderr(0,"CSocekt::ngx_write_request_handler()中数据发送完毕，很好。"); //做个提示吧，商用时可以干掉
        
    }

    //能走下来的，要么数据发送完毕了，要么对端断开了，那么执行收尾工作吧；

    //数据发送完毕，或者把需要发送的数据干掉，都说明发送缓冲区可能有地方了，让发送线程往下走判断能否发送新数据
    if(sem_post(&m_semEventSendQueue)==-1)       
        ngx_log_stderr(0,"CSocekt::ngx_write_request_handler()中sem_post(&m_semEventSendQueue)失败.");


    p_memory->FreeMemory(pConn->psendMemPointer);  //释放内存
    pConn->psendMemPointer = NULL;        
    --pConn->iThrowsendCount;  //建议放在最后执行
    return;
}
```

## 连接池

### initconnection初始化连接池

连接池的所有连接都放入m_connectionList，空闲连接都放入m_freeconnectionList，以后要取空闲连接就从空闲连接中取得。注意两个存储连接的链表类型是list<lpngx_connection_t>，存储的是指针，指向具体连接池某一连接的内存。

```cpp
//初始化连接池
void CSocekt::initconnection()
{
    lpngx_connection_t p_Conn;
    CMemory *p_memory = CMemory::GetInstance();   

    int ilenconnpool = sizeof(ngx_connection_t);    
    for(int i = 0; i < m_worker_connections; ++i) //先创建这么多个连接，后续不够再增加
    {
        p_Conn = (lpngx_connection_t)p_memory->AllocMemory(ilenconnpool,true); //清理内存 , 因为这里分配内存new char，无法执行构造函数，所以如下：
        //手工调用构造函数，因为AllocMemory里无法调用构造函数
        p_Conn = new(p_Conn) ngx_connection_t();  //定位new，释放则显式调用p_Conn->~ngx_connection_t();		
        p_Conn->GetOneToUse();
        m_connectionList.push_back(p_Conn);     //所有链接【不管是否空闲】都放在这个list
        m_freeconnectionList.push_back(p_Conn); //空闲连接会放在这个list
    } //end for
    m_free_connection_n = m_total_connection_n = m_connectionList.size(); //开始这两个列表一样大
    return;
}
```

#### GetOneToUse()连接初始化

每次初始化connection都会++iCurrsequence;

```cpp
//分配出去一个连接的时候初始化一些内容,原来内容放在 ngx_get_connection()里，现在放在这里
void ngx_connection_s::GetOneToUse()
{
    ++iCurrsequence;

    fd  = -1;                                         //开始先给-1
    curStat = _PKG_HD_INIT;                           //收包状态处于 初始状态，准备接收数据包头【状态机】
    precvbuf = dataHeadInfo;                          //收包我要先收到这里来，因为我要先收包头，所以收数据的buff直接就是dataHeadInfo
    irecvlen = sizeof(COMM_PKG_HEADER);               //这里指定收数据的长度，这里先要求收包头这么长字节的数据
    
    precvMemPointer   = NULL;                         //既然没new内存，那自然指向的内存地址先给NULL
    iThrowsendCount   = 0;                            //原子的
    psendMemPointer   = NULL;                         //发送数据头指针记录
    events            = 0;                            //epoll事件先给0 
    lastPingTime      = time(NULL);                   //上次ping的时间

    FloodkickLastTime = 0;                            //Flood攻击上次收到包的时间
	FloodAttackCount  = 0;	                          //Flood攻击在该时间内收到包的次数统计
    iSendCount        = 0;                            //发送队列中有的数据条目数，若client只发不收，则可能造成此数过大，依据此数做出踢出处理 
}
```



### ngx_get_connection

多线程操纵链表CLock lock(&m_connectionMutex); 因此需要加锁

从空闲队列取出连接池的连接并且调用GetOneToUse初始化连接，再绑定当前socket的fd。返回连接return p_Conn

没有空闲连接则创建一个新的连接并且要放入总表队列调用GetOneToUse初始化连接，再绑定当前socket的fd。

```cpp
//从连接池中获取一个空闲连接【当一个客户端连接TCP进入，我希望把这个连接和我的 连接池中的 一个连接【对象】绑到一起，后续 我可以通过这个连接，把这个对象拿到，因为对象里边可以记录各种信息】
lpngx_connection_t CSocekt::ngx_get_connection(int isock)
{
    //因为可能有其他线程要访问m_freeconnectionList，m_connectionList【比如可能有专门的释放线程要释放/或者主线程要释放】之类的，所以应该临界一下
    CLock lock(&m_connectionMutex);  

    if(!m_freeconnectionList.empty())
    {
        //有空闲的，自然是从空闲的中摘取
        lpngx_connection_t p_Conn = m_freeconnectionList.front(); //返回第一个元素但不检查元素存在与否
        m_freeconnectionList.pop_front();                         //移除第一个元素但不返回	
        p_Conn->GetOneToUse();
        --m_free_connection_n; 
        p_Conn->fd = isock;
        return p_Conn;
    }

    //走到这里，表示没空闲的连接了，那就考虑重新创建一个连接
    CMemory *p_memory = CMemory::GetInstance();
    lpngx_connection_t p_Conn = (lpngx_connection_t)p_memory->AllocMemory(sizeof(ngx_connection_t),true);
    p_Conn = new(p_Conn) ngx_connection_t();
    p_Conn->GetOneToUse();
    m_connectionList.push_back(p_Conn); //入到总表中来，但不能入到空闲表中来，因为凡是调这个函数的，肯定是要用这个连接的
    ++m_total_connection_n;             
    p_Conn->fd = isock;
    return p_Conn;

    //因为我们要采用延迟释放的手段来释放连接，因此这种 instance就没啥用，这种手段用来处理立即释放才有用。

}
```

### clearconnection

清空整个连接池

```cpp
//最终回收连接池，释放内存
void CSocekt::clearconnection()
{
    lpngx_connection_t p_Conn;
	CMemory *p_memory = CMemory::GetInstance();
	
	while(!m_connectionList.empty())
	{
		p_Conn = m_connectionList.front();
		m_connectionList.pop_front(); 
        p_Conn->~ngx_connection_t();     //手工调用析构函数
		p_memory->FreeMemory(p_Conn);
	}
}
```

### ngx_free_connection()立即回收

用户没有三次握手接入之前我们可以直接立即回收

PutOneToFree中会将此连接的序列号iCurrsequence自加1，以避免过期包的发送。

```cpp
//归还参数pConn所代表的连接到到连接池中，注意参数类型是lpngx_connection_t
void CSocekt::ngx_free_connection(lpngx_connection_t pConn) 
{
    //因为有线程可能要动连接池中连接，所以在合理互斥也是必要的
    CLock lock(&m_connectionMutex);  

    //首先明确一点，连接，所有连接全部都在m_connectionList里；
    pConn->PutOneToFree();

    //扔到空闲连接列表里
    m_freeconnectionList.push_back(pConn);

    //空闲连接数+1
    ++m_free_connection_n;
    return;
}
```

#### PutOneToFree

收到的包不全并且用户退出了，有必要将收到一半的包的内存释放掉

```cpp
//回收回来一个连接的时候做一些事
void ngx_connection_s::PutOneToFree()
{
    ++iCurrsequence;   
    if(precvMemPointer != NULL)//我们曾经给这个连接分配过接收数据的内存，则要释放内存
    {        
        CMemory::GetInstance()->FreeMemory(precvMemPointer);
        precvMemPointer = NULL;        
    }
    if(psendMemPointer != NULL) //如果发送数据的缓冲区里有内容，则要释放内存
    {
        CMemory::GetInstance()->FreeMemory(psendMemPointer);
        psendMemPointer = NULL;
    }
	iThrowsendCount = 0;                              //设置不设置感觉都行         
}   
```

### inRecyConnectQueue延时回收

用户三次握手进来了，但是断了，还是采用延时回收吧，延时回收也会将连接的序列号pConn->iCurrsequence自加1，以避免过期包的发送

放入`CSocekt::list<lpngx_connection_t>m_recyconnectionList`然后ServerRecyConnectionThread线程自会处理

```cpp
//将要回收的连接放到一个队列中来，后续有专门的线程会处理这个队列中的连接的回收
//有些连接，我们不希望马上释放，要隔一段时间后再释放以确保服务器的稳定，所以，我们把这种隔一段时间才释放的连接先放到一个队列中来
void CSocekt::inRecyConnectQueue(lpngx_connection_t pConn)
{
    std::list<lpngx_connection_t>::iterator pos;
    bool iffind = false;
        
    CLock lock(&m_recyconnqueueMutex); //针对连接回收列表的互斥量，因为线程ServerRecyConnectionThread()也有要用到这个回收列表；

    //如下判断防止连接被多次扔到回收站中来
    for(pos = m_recyconnectionList.begin(); pos != m_recyconnectionList.end(); ++pos)
	{
		if((*pos) == pConn)		
		{	
			iffind = true;
			break;			
		}
	}
    if(iffind == true) //找到了，不必再入了
	{
		//我有义务保证这个只入一次嘛
        return;
    }

    pConn->inRecyTime = time(NULL);        //记录回收时间
    ++pConn->iCurrsequence;
    m_recyconnectionList.push_back(pConn); //等待ServerRecyConnectionThread线程自会处理 
    ++m_totol_recyconnection_n;            //待释放连接队列大小+1
    --m_onlineUserCount;                   //连入用户数量-1
    return;
}
```

### ServerRecyConnectionThread处理连接回收的线程

**伪代码**

```cpp
//处理连接回收的线程
void* CSocekt::ServerRecyConnectionThread(void* threadData)
    while(1)
        usleep(200 * 1000);//睡眠200毫秒
		pthread_mutex_lock(&pSocketObj->m_recyconnqueueMutex);//上锁
        for(; pos != posend; ++pos)
            if(进入回收站时间+等待回收时间>当前时间)continue;//未到释放时间
			m_recyconnectionList.erase(pos);//将释放连接容器里的连接释放
			ngx_free_connection(p_Conn);////归还参数pConn所代表的连接到到连接池中
		pthread_mutex_unlock(&pSocketObj->m_recyconnqueueMutex);//解锁
        if(g_stopEvent == 1) //要退出整个程序，那么肯定要先退出这个循环，所有连接硬释放
        做上面的相同行为而且不加时间判断，对所有连接全回收
	endwhile(1)   
```

CSocket的静态成员函数，与线程池无关

```cpp
//处理连接回收的线程
void* CSocekt::ServerRecyConnectionThread(void* threadData)
{
    ThreadItem *pThread = static_cast<ThreadItem*>(threadData);
    CSocekt *pSocketObj = pThread->_pThis;
    
    time_t currtime;
    int err;
    std::list<lpngx_connection_t>::iterator pos,posend;
    lpngx_connection_t p_Conn;
    
    while(1)
    {
        //为简化问题，我们直接每次休息200毫秒
        usleep(200 * 1000);  //单位是微妙,又因为1毫秒=1000微妙，所以 200 *1000 = 200毫秒

        //不管啥情况，先把这个条件成立时该做的动作做了
        if(pSocketObj->m_totol_recyconnection_n > 0)
        {
            currtime = time(NULL);
            err = pthread_mutex_lock(&pSocketObj->m_recyconnqueueMutex);  
            if(err != 0) ngx_log_stderr(err,"CSocekt::ServerRecyConnectionThread()中pthread_mutex_lock()失败，返回的错误码为%d!",err);

lblRRTD:
            pos    = pSocketObj->m_recyconnectionList.begin();
			posend = pSocketObj->m_recyconnectionList.end();
            for(; pos != posend; ++pos)
            {
                p_Conn = (*pos);
                if(
                    ( (p_Conn->inRecyTime + pSocketObj->m_RecyConnectionWaitTime) > currtime)  && (g_stopEvent == 0) //如果不是要整个系统退出，你可以continue，否则就得要强制释放
                    )
                {
                    continue; //没到释放的时间
                }    
                //到释放的时间了: 
                //......这将来可能还要做一些是否能释放的判断[在我们写完发送数据代码之后吧]，先预留位置
                //....

                //我认为，凡是到释放时间的，iThrowsendCount都应该为0；这里我们加点日志判断下
                //if(p_Conn->iThrowsendCount != 0)
                if(p_Conn->iThrowsendCount > 0)
                {
                    //这确实不应该，打印个日志吧；
                    ngx_log_stderr(0,"CSocekt::ServerRecyConnectionThread()中到释放时间却发现p_Conn.iThrowsendCount!=0，这个不该发生");
                    //其他先暂时啥也不敢，路程继续往下走，继续去释放吧。
                }

                //流程走到这里，表示可以释放，那我们就开始释放
                --pSocketObj->m_totol_recyconnection_n;        //待释放连接队列大小-1
                pSocketObj->m_recyconnectionList.erase(pos);   //迭代器已经失效，但pos所指内容在p_Conn里保存着呢
           
                pSocketObj->ngx_free_connection(p_Conn);	   //归还参数pConn所代表的连接到到连接池中
                goto lblRRTD; 
            } //end for
            err = pthread_mutex_unlock(&pSocketObj->m_recyconnqueueMutex); 
            if(err != 0)  ngx_log_stderr(err,"CSocekt::ServerRecyConnectionThread()pthread_mutex_unlock()失败，返回的错误码为%d!",err);
        } //end if

        if(g_stopEvent == 1) //要退出整个程序，那么肯定要先退出这个循环
        {
            if(pSocketObj->m_totol_recyconnection_n > 0)
            {
                //因为要退出，所以就得硬释放了【不管到没到时间，不管有没有其他不 允许释放的需求，都得硬释放】
                err = pthread_mutex_lock(&pSocketObj->m_recyconnqueueMutex);  
                if(err != 0) ngx_log_stderr(err,"CSocekt::ServerRecyConnectionThread()中pthread_mutex_lock2()失败，返回的错误码为%d!",err);

        lblRRTD2:
                pos    = pSocketObj->m_recyconnectionList.begin();
			    posend = pSocketObj->m_recyconnectionList.end();
                for(; pos != posend; ++pos)
                {
                    p_Conn = (*pos);
                    --pSocketObj->m_totol_recyconnection_n;        //待释放连接队列大小-1
                    pSocketObj->m_recyconnectionList.erase(pos);   //迭代器已经失效，但pos所指内容在p_Conn里保存着呢
                    pSocketObj->ngx_free_connection(p_Conn);	   //归还参数pConn所代表的连接到到连接池中
                    goto lblRRTD2; 
                } //end for
                err = pthread_mutex_unlock(&pSocketObj->m_recyconnqueueMutex); 
                if(err != 0)  ngx_log_stderr(err,"CSocekt::ServerRecyConnectionThread()pthread_mutex_unlock2()失败，返回的错误码为%d!",err);
            } //end if
            break; //整个程序要退出了，所以break;
        }  //end if
    } //end while    
    
    return (void*)0;
}
```



## 线程池

### 线程类

```h
//线程池相关类
class CThreadPool
{
public:
    //构造函数
    CThreadPool();               
    
    //析构函数
    ~CThreadPool();                           

public:
    bool Create(int threadNum);                     //创建该线程池中的所有线程
    void StopAll();                                 //使线程池中的所有线程退出

    void inMsgRecvQueueAndSignal(char *buf);        //收到一个完整消息后，入消息队列，并触发线程池中线程来处理该消息
    void Call();                                    //来任务了，调一个线程池中的线程下来干活  
    int  getRecvMsgQueueCount(){return m_iRecvMsgQueueCount;} //获取接收消息队列大小

private:
    static void* ThreadFunc(void *threadData);      //新线程的线程回调函数    
	//char *outMsgRecvQueue();                        //将一个消息出消息队列	，不需要，直接在ThreadFunc()中处理
    void clearMsgRecvQueue();                       //清理接收消息队列

private:
    //定义一个 线程池中的 线程 的结构，以后可能做一些统计之类的 功能扩展，所以引入这么个结构来 代表线程 感觉更方便一些
    struct ThreadItem   
    {
        pthread_t   _Handle;                        //线程句柄
        CThreadPool *_pThis;                        //记录线程池的指针	
        bool        ifrunning;                      //标记是否正式启动起来，启动起来后，才允许调用StopAll()来释放

        //构造函数
        ThreadItem(CThreadPool *pthis):_pThis(pthis),ifrunning(false){}                             
        //析构函数
        ~ThreadItem(){}        
    };

private:
    static pthread_mutex_t     m_pthreadMutex;      //线程同步互斥量/也叫线程同步锁
    static pthread_cond_t      m_pthreadCond;       //线程同步条件变量
    static bool                m_shutdown;          //线程退出标志，false不退出，true退出

    int                        m_iThreadNum;        //要创建的线程数量

    //int                        m_iRunningThreadNum; //线程数, 运行中的线程数	
    std::atomic<int>           m_iRunningThreadNum; //线程数, 运行中的线程数，原子操作
    time_t                     m_iLastEmgTime;      //上次发生线程不够用【紧急事件】的时间,防止日志报的太频繁
    //time_t                     m_iPrintInfoTime;    //打印信息的一个间隔时间，我准备10秒打印出一些信息供参考和调试
    //time_t                     m_iCurrTime;         //当前时间

    std::vector<ThreadItem *>  m_threadVector;      //线程 容器，容器里就是各个线程了 

    //接收消息队列相关
    std::list<char *>          m_MsgRecvQueue;      //接收数据消息队列 
	int                        m_iRecvMsgQueueCount;//收消息队列大小
};
```

### 创建线程池（worker进程中执行）

#### Create()会激发线程入口函数ThreadFunc

```cpp
//创建线程池中的线程，要手工调用，不在构造函数里调用了
//返回值：所有线程都创建成功则返回true，出现错误则返回false
bool CThreadPool::Create(int threadNum)
{    
    ThreadItem *pNew;
    int err;

    m_iThreadNum = threadNum; //保存要创建的线程数量    
    
    for(int i = 0; i < m_iThreadNum; ++i)
    {
        m_threadVector.push_back(pNew = new ThreadItem(this));             //创建 一个新线程对象 并入到线程池容器中         
        err = pthread_create(&pNew->_Handle, NULL, ThreadFunc, pNew);      //创建线程，错误不返回到errno，一般返回错误码
        if(err != 0)
        {
            //创建线程有错
            ngx_log_stderr(err,"CThreadPool::Create()创建线程%d失败，返回的错误码为%d!",i,err);
            return false;
        }
        else
        {
            //创建线程成功
            //ngx_log_stderr(0,"CThreadPool::Create()创建线程%d成功,线程id=%d",pNew->_Handle);
        }        
    } //end for

    //我们必须保证每个线程都启动并运行到pthread_cond_wait()，本函数才返回，只有这样，这几个线程才能进行后续的正常工作 
    std::vector<ThreadItem*>::iterator iter;
lblfor:
    for(iter = m_threadVector.begin(); iter != m_threadVector.end(); iter++)
    {
        if( (*iter)->ifrunning == false) //这个条件保证所有线程完全启动起来，以保证整个线程池中的线程正常工作；
        {
            //这说明有没有启动完全的线程
            usleep(100 * 1000);  //单位是微妙,又因为1毫秒=1000微妙，所以 100 *1000 = 100毫秒
            goto lblfor;
        }
    }
    return true;
}
```

注意`m_threadVector.push_back(pNew = new ThreadItem(this));`这个this指针是线程池CThreadPool的指针，通过这一句指向CthreadPool的指针就传入ThreadItem中去了。

CThreadPool是线程池的管理类，整个服务器只需这一个对象即可

ThreadItem是线程的结构，包含线程句柄，线程各个状态等等，`CthreadPool::vector<ThreadItem *>  m_threadVector;` 就是线程 容器，容器里就是各个线程了

**特别注意：**【很重要】

线程池要求都执行阻塞到这一行`pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex);`，在此之前create()函数不允许返回。因为如果不这样的话，先开的线程可能create()函数已经执行完毕了并且开始执行比如StopAll()函数进行修改甚至关闭了线程池资源了，但是所有线程还没有完全启动，这样会导致线程池异常。 

所以上面的lblfor循环是为了保证所有线程完全启动起来，以保证整个线程池中的线程正常工作。

#### ThreadFunc()线程入口函数       

精华代码：

```cpp
err = pthread_mutex_lock(&m_pthreadMutex);  
while ( (pThreadPoolObj->m_MsgRecvQueue.size() == 0) && m_shutdown == false) {
    if(pThread->ifrunning == false)            
        pThread->ifrunning = true; 
    pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); 
}
```

1. 注意在CThreadPool类中`static void* ThreadFunc(void *threadData); `是一个静态函数，不存在this指针，因此临时定义`CThreadPool *pThreadPoolObj = pThread->_pThis;`
2. 注意m_pthreadCond是一个静态成员`static pthread_cond_t    m_pthreadCond;`   
   `pthread_cond_t CThreadPool::m_pthreadCond = PTHREAD_COND_INITIALIZER;   //初始化`
3. 因此对于m_pthreadCond 而言`pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); `刚开始时初始状态，没有什么东西来激发它，**会卡在这里，而且m_pthreadMutex会被释放掉；**
   第一个线程执行到这一句的时候，m_pthreadMutex会被释放掉，第二个线程得以在while循环中往下执行。如果有100个线程，最终结果是100个线程都会卡在这里并且m_pthreadMutex会被释放掉。这100个线程都在等待m_pthreadCond这个条件。

```cpp
//线程入口函数，当用pthread_create()创建线程后，这个ThreadFunc()函数都会被立即执行；注意这个是静态函数，不带有this参数
void* CThreadPool::ThreadFunc(void* threadData)
{
    //这个是静态成员函数，是不存在this指针的；
    ThreadItem *pThread = static_cast<ThreadItem*>(threadData);
    CThreadPool *pThreadPoolObj = pThread->_pThis;
    
    CMemory *p_memory = CMemory::GetInstance();	    
    int err;

    pthread_t tid = pthread_self(); //获取线程自身id，以方便调试打印信息等    
    while(true)
    {
        //线程用pthread_mutex_lock()函数去锁定指定的mutex变量，若该mutex已经被另外一个线程锁定了，该调用将会阻塞线程直到mutex被解锁。  
        err = pthread_mutex_lock(&m_pthreadMutex);  
        if(err != 0) ngx_log_stderr(err,"CThreadPool::ThreadFunc()中pthread_mutex_lock()失败，返回的错误码为%d!",err);//有问题，要及时报告
        

        //以下这行程序写法技巧十分重要，必须要用while这种写法，
        //因为：pthread_cond_wait()是个值得注意的函数，调用一次pthread_cond_signal()可能会唤醒多个【惊群】【官方描述是 至少一个/pthread_cond_signal 在多处理器上可能同时唤醒多个线程】
        //pthread_cond_wait()函数，如果只有一条消息 唤醒了两个线程干活，那么其中有一个线程拿不到消息，那如果不用while写，就会出问题，所以被惊醒后必须再次用while拿消息，拿到才走下来；
        while ( (pThreadPoolObj->m_MsgRecvQueue.size() == 0) && m_shutdown == false)
        {
            //如果这个pthread_cond_wait被唤醒【被唤醒后程序执行流程往下走的前提是拿到了锁--官方：pthread_cond_wait()返回时，互斥量再次被锁住】，
              //那么会立即再次执行g_socket.outMsgRecvQueue()，如果拿到了一个NULL，则继续在这里wait着();
            if(pThread->ifrunning == false)            
                pThread->ifrunning = true; //标记为true了才允许调用StopAll()：测试中发现如果Create()和StopAll()紧挨着调用，就会导致线程混乱，所以每个线程必须执行到这里，才认为是启动成功了；
            
            //ngx_log_stderr(0,"执行了pthread_cond_wait-------------begin");
            //刚开始执行pthread_cond_wait()的时候，会卡在这里，而且m_pthreadMutex会被释放掉；
            pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); //整个服务器程序刚初始化的时候，所有线程必然是卡在这里等待的；
            //ngx_log_stderr(0,"执行了pthread_cond_wait-------------end");
        }

        //能走下来的，必然是 拿到了真正的 消息队列中的数据   或者 m_shutdown == true
        //走到这里时刻，互斥量肯定是锁着的。。。。。。

        //先判断线程退出这个条件
        if(m_shutdown)
        {   
            pthread_mutex_unlock(&m_pthreadMutex); //解锁互斥量
            break;                     
        }

        //走到这里，可以取得消息进行处理了【消息队列中必然有消息】,注意，目前还是互斥着呢
        char *jobbuf = pThreadPoolObj->m_MsgRecvQueue.front();     //返回第一个元素但不检查元素存在与否
        pThreadPoolObj->m_MsgRecvQueue.pop_front();                //移除第一个元素但不返回	
        --pThreadPoolObj->m_iRecvMsgQueueCount;                    //收消息队列数字-1
               
        //可以解锁互斥量了
        err = pthread_mutex_unlock(&m_pthreadMutex); 
        if(err != 0)  ngx_log_stderr(err,"CThreadPool::ThreadFunc()中pthread_mutex_unlock()失败，返回的错误码为%d!",err);//有问题，要及时报告
        
        //能走到这里的，就是有消息可以处理，开始处理
        ++pThreadPoolObj->m_iRunningThreadNum;    //原子+1【记录正在干活的线程数量增加1】，这比互斥量要快很多

        g_socket.threadRecvProcFunc(jobbuf);     //处理消息队列中来的消息

        p_memory->FreeMemory(jobbuf);              //释放消息内存 
        --pThreadPoolObj->m_iRunningThreadNum;     //原子-1【记录正在干活的线程数量减少1】

    } //end while(true)

    //能走出来表示整个程序要结束啊，怎么判断所有线程都结束？
    return (void*)0;
}
```

### 线程处理消息队列

所有线程都卡在`pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex);`才能初始化线程。

```h
//定义成员函数指针
typedef bool (CLogicSocket::*handler)(  lpngx_connection_t pConn,      //连接池中连接的指针
                                        LPSTRUC_MSG_HEADER pMsgHeader,  //消息头指针
                                        char *pPkgBody,                 //包体指针
                                        unsigned short iBodyLength);    //包体长度

//用来保存 成员函数指针 的这么个数组
static const handler statusHandler[] = 
{
    //数组前5个元素，保留，以备将来增加一些基本服务器功能
    &CLogicSocket::_HandlePing,                             //【0】：心跳包的实现
    NULL,                                                   //【1】：下标从0开始
    NULL,                                                   //【2】：下标从0开始
    NULL,                                                   //【3】：下标从0开始
    NULL,                                                   //【4】：下标从0开始
 
    //开始处理具体的业务逻辑
    &CLogicSocket::_HandleRegister,                         //【5】：实现具体的注册功能
    &CLogicSocket::_HandleLogIn,                            //【6】：实现具体的登录功能
    //......其他待扩展，比如实现攻击功能，实现加血功能等等；


};
#define AUTH_TOTAL_COMMANDS sizeof(statusHandler)/sizeof(handler) //整个命令有多少个，编译时即可知道
```

#### threadRecvProcFunc

**ThreadFunc所有线程都阻塞在pthread_cond_wait处，当有消息来时有线程取到消息后会调用当前函数**

- 1.用强制类型转换取得消息头和包头两个结构体。从包头中取出包的长度（注意要ntohs网络序转本机序）。
- 2.如果没有包体只有包头，crc32校验码应该是0，若不为0丢弃包return。
- 3.如果有包体，拿到包体并且服务器通过包体计算得到的crc32校验码，然后与客户端传来pPkgHeader->crc32校验码比较是否一致，若不一致丢弃且return。
- 4.然后通过消息头取出连接池的连接指针p_Conn和消息头的iCurrsequence，然后比较连接池的此连接的p_Conn->iCurrsequence与消息头的iCurrsequence是否一致，若不一致说明连接已经关闭了。丢弃包直接return。

```cpp
typedef struct _STRUC_MSG_HEADER{//消息头结构体
	lpngx_connection_t pConn;         //记录对应的链接，注意这是个指针
	uint64_t           iCurrsequence; //收到数据包时记录对应连接的序号，将来能用于比较是否连接已经作废用
}STRUC_MSG_HEADER,*LPSTRUC_MSG_HEADER;
```

- 5.判断长度是否大于AUTH_TOTAL_COMMANDS，若大于是恶意包，根本没有这么多命令，丢弃包并且return。
- 6.`(this->*statusHandler[imsgCode])(p_Conn,pMsgHeader,(char *)pPkgBody,pkglen-m_iLenPkgHeader);`根据客户端发来包头内部的消息类型代码（区别每个不同的命令）调用对应的函数实现各个不同消息类型的处理逻辑。

```cpp
///处理收到的数据包，由线程池来调用本函数，本函数是一个单独的线程；
//pMsgBuf：消息头 + 包头 + 包体 ：自解释；
void CLogicSocket::threadRecvProcFunc(char *pMsgBuf)
{          
    LPSTRUC_MSG_HEADER pMsgHeader = (LPSTRUC_MSG_HEADER)pMsgBuf;                  //消息头
    LPCOMM_PKG_HEADER  pPkgHeader = (LPCOMM_PKG_HEADER)(pMsgBuf+m_iLenMsgHeader); //包头
    void  *pPkgBody;                                                              //指向包体的指针
    unsigned short pkglen = ntohs(pPkgHeader->pkgLen);                            //客户端指明的包宽度【包头+包体】

    if(m_iLenPkgHeader == pkglen)
    {
        //没有包体，只有包头
		if(pPkgHeader->crc32 != 0) //只有包头的crc值给0
		{
			return; //crc错，直接丢弃
		}
		pPkgBody = NULL;
    }
    else 
	{
        //有包体，走到这里
		pPkgHeader->crc32 = ntohl(pPkgHeader->crc32);		          //针对4字节的数据，网络序转主机序
		pPkgBody = (void *)(pMsgBuf+m_iLenMsgHeader+m_iLenPkgHeader); //跳过消息头 以及 包头 ，指向包体

        //ngx_log_stderr(0,"CLogicSocket::threadRecvProcFunc()中收到包的crc值为%d!",pPkgHeader->crc32);

		//计算crc值判断包的完整性        
		int calccrc = CCRC32::GetInstance()->Get_CRC((unsigned char *)pPkgBody,pkglen-m_iLenPkgHeader); //计算纯包体的crc值
		if(calccrc != pPkgHeader->crc32) //服务器端根据包体计算crc值，和客户端传递过来的包头中的crc32信息比较
		{
            ngx_log_stderr(0,"CLogicSocket::threadRecvProcFunc()中CRC错误[服务器:%d/客户端:%d]，丢弃数据!",calccrc,pPkgHeader->crc32);    //正式代码中可以干掉这个信息
			return; //crc错，直接丢弃
		}
        else
        {
            //ngx_log_stderr(0,"CLogicSocket::threadRecvProcFunc()中CRC正确[服务器:%d/客户端:%d]，不错!",calccrc,pPkgHeader->crc32);
        }        
	}

    //包crc校验OK才能走到这里    	
    unsigned short imsgCode = ntohs(pPkgHeader->msgCode); //消息代码拿出来
    lpngx_connection_t p_Conn = pMsgHeader->pConn;        //消息头中藏着连接池中连接的指针

    //我们要做一些判断
    //(1)如果从收到客户端发送来的包，到服务器释放一个线程池中的线程处理该包的过程中，客户端断开了，那显然，这种收到的包我们就不必处理了；    
    if(p_Conn->iCurrsequence != pMsgHeader->iCurrsequence)   //该连接池中连接以被其他tcp连接【其他socket】占用，这说明原来的 客户端和本服务器的连接断了，这种包直接丢弃不理
    {
        return; //丢弃不理这种包了【客户端断开了】
    }

    //(2)判断消息码是正确的，防止客户端恶意侵害我们服务器，发送一个不在我们服务器处理范围内的消息码
	if(imsgCode >= AUTH_TOTAL_COMMANDS) //无符号数不可能<0
    {
        ngx_log_stderr(0,"CLogicSocket::threadRecvProcFunc()中imsgCode=%d消息码不对!",imsgCode); //这种有恶意倾向或者错误倾向的包，希望打印出来看看是谁干的
        return; //丢弃不理这种包【恶意包或者错误包】
    }

    //能走到这里的，包没过期，不恶意，那好继续判断是否有相应的处理函数
    //(3)有对应的消息处理函数吗
    if(statusHandler[imsgCode] == NULL) //这种用imsgCode的方式可以使查找要执行的成员函数效率特别高
    {
        ngx_log_stderr(0,"CLogicSocket::threadRecvProcFunc()中imsgCode=%d消息码找不到对应的处理函数!",imsgCode); //这种有恶意倾向或者错误倾向的包，希望打印出来看看是谁干的
        return;  //没有相关的处理函数
    }

    //一切正确，可以放心大胆的处理了
    //(4)调用消息码对应的成员函数来处理
    (this->*statusHandler[imsgCode])(p_Conn,pMsgHeader,(char *)pPkgBody,pkglen-m_iLenPkgHeader);
    return;	
}
```

### 释放线程池

虽然一般不会调用这个函数，而且实在不行直接关闭程序系统帮我们释放资源，但是为了优雅一点自己实现一下。

#### StopAll() 

```cpp
err = pthread_mutex_lock(&m_pthreadMutex);  
while ( (pThreadPoolObj->m_MsgRecvQueue.size() == 0) && m_shutdown == false) {
    if(pThread->ifrunning == false)            
        pThread->ifrunning = true; 
    pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); 
}
```

先将所有线程安全退出后，再将内存释放。“优雅干净”

- 1.首先给个判断m_shutdown避免重复释放，然后m_shutdown置为true表示要关闭线程池了，这是个静态变量`static bool CThreadPool::m_shutdown = false;`
- 2.pthread_cond_broadcast广播会激发ThreadPool的静态成员m_pthreadCond，一旦激发成功pthread_cond_wait(&m_pthreadCond, &m_pthreadMutex); 的线程被唤醒了，并且这个while循环条件不满足，所有线程都去拿锁m_pthreadMutex，其他未拿到锁的线程只能卡死，等待上个线程退出释放锁，最终所有线程return退出。
- 3.for循环pthread_join回收退出的线程资源直至所有线程都退出
- 4.通过m_threadVector容器中的成员指针将之前new出来的ThreadItem内存释放。

```c
//停止所有线程【等待结束线程池中所有线程，该函数返回后，应该是所有线程池中线程都结束了】
void CThreadPool::StopAll() 
{
    //(1)已经调用过，就不要重复调用了
    if(m_shutdown == true)
    {
        return;
    }
    m_shutdown = true;

    //(2)唤醒等待该条件【卡在pthread_cond_wait()的】的所有线程，一定要在改变条件状态以后再给线程发信号
    int err = pthread_cond_broadcast(&m_pthreadCond); 
    if(err != 0)
    {
        //这肯定是有问题，要打印紧急日志
        ngx_log_stderr(err,"CThreadPool::StopAll()中pthread_cond_broadcast()失败，返回的错误码为%d!",err);
        return;
    }

    //(3)等等线程，让线程真返回    
    std::vector<ThreadItem*>::iterator iter;
	for(iter = m_threadVector.begin(); iter != m_threadVector.end(); iter++)
    {
        pthread_join((*iter)->_Handle, NULL); //等待一个线程终止
    }

    //流程走到这里，那么所有的线程池中的线程肯定都返回了；
    pthread_mutex_destroy(&m_pthreadMutex);
    pthread_cond_destroy(&m_pthreadCond);    

    //(4)释放一下new出来的ThreadItem【线程池中的线程】    
	for(iter = m_threadVector.begin(); iter != m_threadVector.end(); iter++)
	{
		if(*iter)
			delete *iter;
	}
	m_threadVector.clear();

    ngx_log_stderr(0,"CThreadPool::StopAll()成功返回，线程池中线程全部正常结束!");
    return;    
}
```

### 

```cpp
//将一个待发送消息入到发消息队列中
void CSocekt::msgSend(char *psendbuf) 
    上锁m_sendMessageQueueMutex
    判断当前TCP连接的发送消息队列m_iSendMsgQueueCount大小
    	过大：丢包并直接return
    判断当前TCP连接的发送消息队列iSendCount大小
    	过大：直接关闭连接并返回
    
    m_MsgSendQueue.push_back(psendbuf);//放入发送消息队列
    sem_post(&m_semEventSendQueue)//信号量加一，让ServerSendQueueThread()流程走下来干活
    
```



```cpp
//处理发送消息队列的线程
void* CSocekt::ServerSendQueueThread(void* threadData)
    while(程序不退出)
        sem_wait(&pSocketObj->m_semEventSendQueue)//本线程卡在这里，等待msgSend函数唤醒
        pthread_mutex_lock(m_sendMessageQueueMutex);//加锁
		while(pos != posend)//遍历消息队列
            p_Conn = pMsgHeader->pConn;//取出当前TCP连接的指针
            判断是否消息过期
                是：从队列中移除并且continue
            发送缓冲区是否满？
                是：continue
           
            sendproc()//终于开始发数据了 
            if(一下子数据全发出去了)释放内存
            else (没有全部发送完毕 EAGAIN )//肯定是因为 发送缓冲区满了
                ngx_epoll_oper_event//依靠epoll驱动调用ngx_write_request_handler()函数发送数据
            else(一个字节都没发)
                ngx_epoll_oper_event//也是依靠epoll驱动
            else //对端断开了
        end while(pos != posend)
        解锁
	end while(程序不退出)
 	return;
```



```cpp

```



```cpp
//设置数据发送时的写处理函数,当数据可写时epoll通知我们，我们在 int CSocekt::ngx_epoll_process_events(int timer)  中调用此函数
//在 int CSocekt::ngx_epoll_process_events(int timer)  中epoll_wait阻塞了
//能走到这里，数据就是没法送完毕，要继续发送
void CSocekt::ngx_write_request_handler(lpngx_connection_t pConn)
    sendsize = sendproc(pConn,pConn->psendbuf,pConn->isendlen);//发消息
	if(sendsize > 0 && sendsize != pConn->isendlen)//没有全部发完
        return;//返回，等待epoll_wait再次唤醒这个函数继续发送
	
	if(sendsize > 0 && sendsize == pConn->isendlen) //成功发送完毕，做个通知是可以的；
		ngx_epoll_oper_event//把写事件通知从epoll中干掉
        
    sem_post(&m_semEventSendQueue)//信号量加一，让ServerSendQueueThread()流程走下来干活
```



```cpp
//来数据时候的处理，当连接上有数据来的时候，本函数会被ngx_epoll_process_events()所调用  ,官方的类似函数为ngx_http_wait_request_handler();
void CSocekt::ngx_read_request_handler(lpngx_connection_t pConn)
    ssize_t reco = recvproc(pConn,pConn->precvbuf,pConn->irecvlen); //收取数据
    if(reco <= 0) return;
    ngx_wait_request_handler_proc_p1(pConn,isflood); //那就调用专门针对包头处理完整的函数去处理把。
    ngx_wait_request_handler_proc_plast(pConn,isflood);//包体处理函数
		g_threadpool.inMsgRecvQueueAndSignal(pConn->precvMemPointer); //入消息队列并触发线程处理消息
	
    
```



```cpp
//收到一个完整消息后，入消息队列，并触发线程池中线程来处理该消息
void CThreadPool::inMsgRecvQueueAndSignal(char *buf)
{
    pthread_mutex_lock(&m_pthreadMutex);上锁
    m_MsgRecvQueue.push_back(buf);	         //入消息队列
    pthread_mutex_unlock(&m_pthreadMutex);解锁
    //可以激发一个线程来干活了
    Call();
    return;
}
```



```cpp
//来任务了，调一个线程池中的线程下来干活
void CThreadPool::Call()
{
    pthread_cond_signal(&m_pthreadCond); //唤醒一个等待该条件的线程，也就是可以唤醒卡在pthread_cond_wait()的线程
    if(m_iThreadNum == m_iRunningThreadNum) //线程池中线程总量，跟当前正在干活的线程数量一样，说明所有线程都忙碌起
            ngx_log_stderr(0,"CThreadPool::Call()中发现线程池中当前空闲线程数量为0，要考虑扩容线程池了!");
    return;
}
```





```cpp
//处理连接回收的线程
void* CSocekt::ServerRecyConnectionThread(void* threadData)
    while(1)
        usleep(200 * 1000);//睡眠200毫秒
		pthread_mutex_lock(&pSocketObj->m_recyconnqueueMutex);上锁
        for(; pos != posend; ++pos)
            if(进入回收站时间+等待回收时间>当前时间)continue;//未到释放时间
			m_recyconnectionList.erase(pos);//将释放连接容器里的连接释放
			connection(p_Conn);////归还参数pConn所代表的连接到到连接池中
		解锁
        if(g_stopEvent == 1) //要退出整个程序，那么肯定要先退出这个循环
        做上面的相同行为而且不加时间判断，对所有连接全回收
	endwhile(1)   
        
```



```cpp
//归还参数pConn所代表的连接到到连接池中，注意参数类型是lpngx_connection_t
void CSocekt::ngx_free_connection(lpngx_connection_t pConn) 
{
    //因为有线程可能要动连接池中连接，所以在合理互斥也是必要的
    CLock lock(&m_connectionMutex);  

    //首先明确一点，连接，所有连接全部都在m_connectionList里；
    pConn->PutOneToFree();

    //扔到空闲连接列表里
    m_freeconnectionList.push_back(pConn);

    //空闲连接数+1
    ++m_free_connection_n;

    return;
}
//
	std::list<lpngx_connection_t>  m_connectionList;                      //连接列表【连接池】
	std::list<lpngx_connection_t>  m_freeconnectionList;                  //空闲连接列表【这里边装的全是空闲的
	std::list<lpngx_connection_t>  m_recyconnectionList;                  //将要释放的连接放这里
```



```cpp
//回收回来一个连接的时候做一些事
void ngx_connection_s::PutOneToFree()
{
    ++iCurrsequence;   
    if(precvMemPointer != NULL)//我们曾经给这个连接分配过接收数据的内存，则要释放内存
    {        
        CMemory::GetInstance()->FreeMemory(precvMemPointer);
        precvMemPointer = NULL;        
    }
    if(psendMemPointer != NULL) //如果发送数据的缓冲区里有内容，则要释放内存
    {
        CMemory::GetInstance()->FreeMemory(psendMemPointer);
        psendMemPointer = NULL;
    }

    iThrowsendCount = 0;                              //设置不设置感觉都行         
}
```

## 业务逻辑

线程池里面地线程都“嗷嗷待哺”地等待客户端发来消息，线程池地线程都会执行threadRecvProcFunc函数，这个函数会根据发来的消息包的不同执行不同的逻辑函数。

### 心跳包

心跳包其实就是 一个普通的数据包；

一般每个几十秒，最长一般也就是1分钟【10秒-60秒之间】，有客户端主动发送给服务器；服务器收到之后，一般会给客户端返回一个心跳包；

三路握手，tcp连接建立之后，才存在发送心跳包的问题—— 如果c不给s发心跳包，服务器会怎样；约定 30秒发送 一次； 服务器可能会在90秒或者100秒内，主动关闭该客户端的socket连接；

作为一个好的客户端程序，如果你发送了心跳包给服务器，但是在90或者100秒之内，你[客户端]没有收到服务器回应的心跳包，那么你就应该主动关闭与服务器端的链接，并且如果业务需要重连，客户端程序在关闭这个连接后还要重新主动再次尝试连接服务器端；客户端程序 也有必要提示使用者 与服务器的连接已经断开；

**为什么引入心跳包？**

常规客户端关闭，服务器端能感知到；但是有一种特殊情况，连接断开c/s都感知不到；

c /s程序运行在不同的两个物理电脑上；tcp已经建立；
拔掉c /s程序的网线； **拔掉网线导致服务器感知不到客户端断开**，这个事实，一定要知道；
为了应对拔网线，导致不知道对方是否断开了tcp连接这种事，这就是我们引入心跳包机制的原因；
超时没有发送来心跳包，那么就会将对端的socket连接close掉，回收资源；这就是心跳包的作用；
其他作用：检测网络延迟等等
这里心跳包主要目的就是检测双方的链接是否断开；

tcp本身keepalive机制；因为检测时间不好控制，所以不适合我们；

因此连接池的每个连接引入一个成员变量lastPingTime记录上一次的ping命令（心跳包）的时间，不断地更新

#### 处理发来的心跳包

##### _HandlePing

```cpp
//接收并处理客户端发送过来的ping包
bool CLogicSocket::_HandlePing(lpngx_connection_t pConn,LPSTRUC_MSG_HEADER pMsgHeader,char *pPkgBody,unsigned short iBodyLength)
{
    //心跳包要求没有包体；
    if(iBodyLength != 0)  //有包体则认为是 非法包
		return false; 

    CLock lock(&pConn->logicPorcMutex); //凡是和本用户有关的访问都考虑用互斥，以免该用户同时发送过来两个命令达到各种作弊目的
    pConn->lastPingTime = time(NULL);   //更新该变量

    //服务器也发送 一个只有包头的数据包给客户端，作为返回的数据
    SendNoBodyPkgToClient(pMsgHeader,_CMD_PING);

    //ngx_log_stderr(0,"成功收到了心跳包并返回结果！");
    return true;
}
```

##### SendNoBodyPkgToClient

```cpp
//发送没有包体的数据包给客户端
void CLogicSocket::SendNoBodyPkgToClient(LPSTRUC_MSG_HEADER pMsgHeader,unsigned short iMsgCode)
{
    CMemory  *p_memory = CMemory::GetInstance();

    char *p_sendbuf = (char *)p_memory->AllocMemory(m_iLenMsgHeader+m_iLenPkgHeader,false);
    char *p_tmpbuf = p_sendbuf;
    
	memcpy(p_tmpbuf,pMsgHeader,m_iLenMsgHeader);
	p_tmpbuf += m_iLenMsgHeader;

    LPCOMM_PKG_HEADER pPkgHeader = (LPCOMM_PKG_HEADER)p_tmpbuf;	  //指向的是我要发送出去的包的包头	
    pPkgHeader->msgCode = htons(iMsgCode);	
    pPkgHeader->pkgLen = htons(m_iLenPkgHeader); 
	pPkgHeader->crc32 = 0;		
    msgSend(p_sendbuf);
    return;
}
```

#### 检测心跳时间

##### 配置文件

20秒；超过20*3 +10 =70秒，仍旧没收到心跳包，那么服务器端就把tcp断开；或者20秒直接断开TCP连接
增加配置Sock_WaitTimeEnable，Sock_MaxWaitTime

```conf
#Sock_WaitTimeEnable：是否开启踢人时钟，1：开启   0：不开启
Sock_WaitTimeEnable = 1
#多少秒检测一次是否 心跳超时，只有当Sock_WaitTimeEnable = 1时，本项才有用
Sock_MaxWaitTime = 20
#当时间到达Sock_MaxWaitTime指定的时间时，直接把客户端踢出去，只有当Sock_WaitTimeEnable = 1时，本项才有用
Sock_TimeOutKick = 0
```

##### CSocekt::AddToTimerQueue

在**ngx_event_accept**（三次握手成功后）调用AddToTimerQueue()添加一个**定时器**。每次进来一个用户，就往时间队列multimap（有序的键/值对，但它可以保存重复的元素）增加一个连接。
每次插入时间队列会按键值 自动排序 小->大。并且将时间队列**头部时间值（最早时间）**保存到m_timer_value_里
`std::multimap<time_t, LPSTRUC_MSG_HEADER>  m_timerQueuemap;`键：时间，值：消息头（消息头存放连接指针和连接序号）。

```cpp
//设置踢出时钟(向multimap表中增加内容)，用户三次握手成功连入，然后我们开启了踢人开关【Sock_WaitTimeEnable = 1】，那么本函数被调用；
void CSocekt::AddToTimerQueue(lpngx_connection_t pConn)
{
    CMemory *p_memory = CMemory::GetInstance();

    time_t futtime = time(NULL);
    futtime += m_iWaitTime;  //20秒之后的时间

    CLock lock(&m_timequeueMutex); //互斥，因为要操作m_timeQueuemap了
    LPSTRUC_MSG_HEADER tmpMsgHeader = (LPSTRUC_MSG_HEADER)p_memory->AllocMemory(m_iLenMsgHeader,false);
    tmpMsgHeader->pConn = pConn;
    tmpMsgHeader->iCurrsequence = pConn->iCurrsequence;
    m_timerQueuemap.insert(std::make_pair(futtime,tmpMsgHeader)); //按键 自动排序 小->大
    m_cur_size_++;  //计时队列尺寸+1
    m_timer_value_ = GetEarliestTime(); //计时队列头部时间值保存到m_timer_value_里
    return;    
}
```

##### CSocekt::ServerSendQueueThread处理时间队列线程

创建一个新线程，专门处理事件队列里心跳包未发送的连接

- 每次取出m_timer_value_最早时间，判断有没有连接是已经过期的（过久未发心跳包）
- 通过GetOverTimeTimer根据给的当前时间，从m_timeQueuemap找到比这个时间更老（更早）的节点【1个】返回去，这些节点都是时间超过了，要处理的节点
- 然后对要处理的时间过期节点，该去检测心跳包是否超时的事宜，是否要踢出这个连接

```cpp
//时间队列监视和处理线程，处理到期不发心跳包的用户踢出的线程
void* CSocekt::ServerTimerQueueMonitorThread(void* threadData)
{
    ThreadItem *pThread = static_cast<ThreadItem*>(threadData);
    CSocekt *pSocketObj = pThread->_pThis;

    time_t absolute_time,cur_time;
    int err;

    while(g_stopEvent == 0) //不退出
    {
        //这里没互斥判断，所以只是个初级判断，目的至少是队列为空时避免系统损耗		
		if(pSocketObj->m_cur_size_ > 0)//队列不为空，有内容
        {
			//时间队列中最近发生事情的时间放到 absolute_time里；
            absolute_time = pSocketObj->m_timer_value_; //这个可是省了个互斥，十分划算
            cur_time = time(NULL);
            if(absolute_time < cur_time)
            {
                //时间到了，可以处理了
                std::list<LPSTRUC_MSG_HEADER> m_lsIdleList; //保存要处理的内容
                LPSTRUC_MSG_HEADER result;

                err = pthread_mutex_lock(&pSocketObj->m_timequeueMutex);  
                if(err != 0) ngx_log_stderr(err,"CSocekt::ServerTimerQueueMonitorThread()中pthread_mutex_lock()失败，返回的错误码为%d!",err);//有问题，要及时报告
                while ((result = pSocketObj->GetOverTimeTimer(cur_time)) != NULL) //一次性的把所有超时节点都拿过来
				{
					m_lsIdleList.push_back(result); 
				}//end while
                err = pthread_mutex_unlock(&pSocketObj->m_timequeueMutex); 
                if(err != 0)  ngx_log_stderr(err,"CSocekt::ServerTimerQueueMonitorThread()pthread_mutex_unlock()失败，返回的错误码为%d!",err);//有问题，要及时报告                
                LPSTRUC_MSG_HEADER tmpmsg;
                while(!m_lsIdleList.empty())
                {
                    tmpmsg = m_lsIdleList.front();
					m_lsIdleList.pop_front(); 
                    pSocketObj->procPingTimeOutChecking(tmpmsg,cur_time); //这里需要检查心跳超时问题
                } //end while(!m_lsIdleList.empty())
            }
        } //end if(pSocketObj->m_cur_size_ > 0)
        
        usleep(500 * 1000); //为简化问题，我们直接每次休息500毫秒
    } //end while

    return (void*)0;
}
```

##### CLogicSocket::procPingTimeOutChecking

```cpp
//心跳包检测时间到，该去检测心跳包是否超时的事宜，本函数是子类函数，实现具体的判断动作
void CLogicSocket::procPingTimeOutChecking(LPSTRUC_MSG_HEADER tmpmsg,time_t cur_time)
{
    CMemory *p_memory = CMemory::GetInstance();

    if(tmpmsg->iCurrsequence == tmpmsg->pConn->iCurrsequence) //此连接没断
    {
        lpngx_connection_t p_Conn = tmpmsg->pConn;

        if(/*m_ifkickTimeCount == 1 && */m_ifTimeOutKick == 1)  //能调用到本函数第一个条件肯定成立，所以第一个条件加不加无所谓，主要是第二个条件
        {
            //到时间直接踢出去的需求
            zdClosesocketProc(p_Conn); 
        }            
        else if( (cur_time - p_Conn->lastPingTime ) > (m_iWaitTime*3+10) ) //超时踢的判断标准就是 每次检查的时间间隔*3，超过这个时间没发送心跳包，就踢【大家可以根据实际情况自由设定】
        {
            //踢出去【如果此时此刻该用户正好断线，则这个socket可能立即被后续上来的连接复用  如果真有人这么倒霉，赶上这个点了，那么可能错踢，错踢就错踢】            
            //ngx_log_stderr(0,"时间到不发心跳包，踢出去!");   //感觉OK
            zdClosesocketProc(p_Conn); 
        }   
             
        p_memory->FreeMemory(tmpmsg);//内存要释放
    }
    else //此连接断了
    {
        p_memory->FreeMemory(tmpmsg);//内存要释放
    }
    return;
}
```

##### CSocekt::DeleteFromTimerQueue

zdClosesocketProc(p_Conn)会调用此函数，主要从时间队列中删除并且释放内存

```cpp
//把指定用户tcp连接从timer表中抠出去
void CSocekt::DeleteFromTimerQueue(lpngx_connection_t pConn)
{
    std::multimap<time_t, LPSTRUC_MSG_HEADER>::iterator pos,posend;
	CMemory *p_memory = CMemory::GetInstance();

    CLock lock(&m_timequeueMutex);

    //因为实际情况可能比较复杂，将来可能还扩充代码等等，所以如下我们遍历整个队列找 一圈，而不是找到一次就拉倒，以免出现什么遗漏
lblMTQM:
	pos    = m_timerQueuemap.begin();
	posend = m_timerQueuemap.end();
	for(; pos != posend; ++pos)	
	{
		if(pos->second->pConn == pConn)
		{			
			p_memory->FreeMemory(pos->second);  //释放内存
			m_timerQueuemap.erase(pos);
			--m_cur_size_; //减去一个元素，必然要把尺寸减少1个;								
			goto lblMTQM;
		}		
	}
	if(m_cur_size_ > 0)
	{
		m_timer_value_ = GetEarliestTime();
	}
    return;    
}
```



## 测试

### 如何把发送缓冲区撑满

（1）每次服务器给客户端发送65K左右的数据，发送到第20次才出现服务器的发送缓冲区满；这时**客户端收了1个包(65K)**，【触发了epoll可写事件，此时执行了 ngx_write_request_handler()】

（2）我又发包，连续成功发送了16次，才又出现发送缓冲区满；我客户端再收包，结果连续**收了16个包**，服务器才又出现ngx_write_request_handler()函数被成功执行，这表示客户端连续收了16次包，服务器的发送缓冲区才倒出地方来；

（3）此后，大概服务器能够连续发送16次才再出现发送缓冲区满，客户端连续收16次，服务器端才出现ngx_write_request_handler()被执行【服务器的发送缓冲区有地方】；

测试结论：

（1）ngx_write_request_handler（）逻辑正确；能够通过此函数把剩余的未成功发送的数据发送出去；

（2）LT模式下，我们发送数据采用的 改进方案 是非常有效的，在很大程度上提高了效率；

（3） 发送缓冲区大概10-几10K,但是我们实际测试的时候，成功的发送出去了1000多k数据才报告发送缓冲区满；
当我们发送端调用send()发送数据时，操作系统底层已经把数据发送到了 该连接的接收端 的**接收缓存**，这个接收缓存大概有几百K，
千万不要认为发送缓冲区只有几十K，所以我们send()几十k就能把发送缓冲区填满；

![image-20220309160014668](../images/202231-nginx整体框架解析/image-20220309160014668.png)

（4）不管怎么说，主要对方不接收数据，发送方的发送缓冲区总有满的时候；当发送缓冲满的时候，我们发送数据就会使用ngx_write_request_handler（）来执行了，所以现在看起来，我们整个的服务器的发送数据的实现代码是正确的；

### 高并发测试

并发数量取决于很多因素：

- (1)采用的开发技术：epoll，支持数十万并发
- (2)这个程序收发数据的频繁程度，以及具体 要处理的业务复杂程度
- (3)服务器实际的物理内存；可用的物理内存数量，会直接决定你能支持的并发连接
- (4)一些其他的tcp/ip配置项

一般，我们日常所写的服务器程序，支持几千甚至1-2万的并发，基本上就差不多了；一个服务器程序，要根据我们具体的物理内存，以及我们具体要实现的业务等等因素，控制能够同时连入的客户端数量；如果你允许客户端无限连入，那么你的服务器肯定会崩溃；

这里我的解决方法是引入一个新变量m_onlineUserCount

void CSocekt::ngx_event_accept(lpngx_connection_t oldc) 连入人数+1
void CSocekt::inRecyConnectQueue(lpngx_connection_t pConn) 连入人数-1

控制连入用户数量的解决思路：如果同时连入的用户数量超过了允许的最大连入数量时，我们就把这个连入的用户直接踢出去；

### 安全问题思考

#### 防范SYN Flood攻击

以游戏服务器为例：

假设我们认为一个合理的客户端一秒钟发送数据包给服务器不超过10个；
如果客户端不停的给服务器发数据包，1秒钟超过了10个数据包 ，那我服务器就认为这个玩家有恶意攻击服务器的倾向；
我们服务器就应该果断的把这个TCP客户端连接关闭，这个也是服务器发现恶意玩家以及保护自身安全的手段；

代码上如何实现 1秒钟超过10个数据包则把客户端踢出去？
增加了TestFlood();

改造了ngx_read_request_handler(),ngx_wait_request_handler_proc_p1()，在每次收到了完整包就可以调用TestFlood()

ngx_wait_request_handler_proc_plast（）判断是否isflood，选择释放内存还是放入消息队列。

```cpp
//测试是否flood攻击成立，成立则返回true，否则返回false
bool CSocekt::TestFlood(lpngx_connection_t pConn)
{
    struct  timeval sCurrTime;   //当前时间结构
	uint64_t        iCurrTime;   //当前时间（单位：毫秒）
	bool  reco      = false;
	
	gettimeofday(&sCurrTime, NULL); //取得当前时间
    iCurrTime =  (sCurrTime.tv_sec * 1000 + sCurrTime.tv_usec / 1000);  //毫秒
	if((iCurrTime - pConn->FloodkickLastTime) < m_floodTimeInterval)   //两次收到包的时间 < 100毫秒
	{
        //发包太频繁记录
		pConn->FloodAttackCount++;
		pConn->FloodkickLastTime = iCurrTime;
	}
	else
	{
        //既然发布不这么频繁，则恢复计数值
		pConn->FloodAttackCount = 0;
		pConn->FloodkickLastTime = iCurrTime;
	}

    //ngx_log_stderr(0,"pConn->FloodAttackCount=%d,m_floodKickCount=%d.",pConn->FloodAttackCount,m_floodKickCount);

	if(pConn->FloodAttackCount >= m_floodKickCount)
	{
		//可以踢此人的标志
		reco = true;
	}
	return reco;
}
```

####  收到太多数据包处理不过来

限速：epoll技术，一个限速的思路；在epoll红黑树节点中，把这个EPOLLIN【可读】通知干掉；系统不会通知，服务器就不会去读，数据一直积累在接收缓冲区里，客户端那边会的发送缓冲区会满，客户端会减慢速度发送甚至停止发送。

数据报太多的话，会在printTDInfo()中做了一个简单提示，大家根据需要自己改造代码；

#### 积压太多数据包发送不出去

见void CSocekt::msgSend(char *psendbuf) ，增加一个判断

```c
//将一个待发送消息入到发消息队列中
void CSocekt::msgSend(char *psendbuf) 
{
    CMemory *p_memory = CMemory::GetInstance();

    CLock lock(&m_sendMessageQueueMutex);  //互斥量

    //发送消息队列过大也可能给服务器带来风险
    if(m_iSendMsgQueueCount > 50000)
    {
        //发送队列过大，比如客户端恶意不接受数据，就会导致这个队列越来越大
        //那么可以考虑为了服务器安全，干掉一些数据的发送，虽然有可能导致客户端出现问题，但总比服务器不稳定要好很多
        m_iDiscardSendPkgCount++;
        p_memory->FreeMemory(psendbuf);
		return;
    }
    
    //总体数据并无风险，不会导致服务器崩溃，要看看个体数据，找一下恶意者了    
    LPSTRUC_MSG_HEADER pMsgHeader = (LPSTRUC_MSG_HEADER)psendbuf;
	lpngx_connection_t p_Conn = pMsgHeader->pConn;
    if(p_Conn->iSendCount > 400)
    {
        //该用户收消息太慢【或者干脆不收消息】，累积的该用户的发送队列中有的数据条目数过大，认为是恶意用户，直接切断
        ngx_log_stderr(0,"CSocekt::msgSend()中发现某用户%d积压了大量待发送数据包，切断与他的连接！",p_Conn->fd);      
        m_iDiscardSendPkgCount++;
        p_memory->FreeMemory(psendbuf);
        zdClosesocketProc(p_Conn); //直接关闭
		return;
    }

    ++p_Conn->iSendCount; //发送队列中有的数据条目数+1；
    m_MsgSendQueue.push_back(psendbuf);     
    ++m_iSendMsgQueueCount;   //原子操作

    //将信号量的值+1,这样其他卡在sem_wait的就可以走下去
    if(sem_post(&m_semEventSendQueue)==-1)  //让ServerSendQueueThread()流程走下来干活
    {
        ngx_log_stderr(0,"CSocekt::msgSend()中sem_post(&m_semEventSendQueue)失败.");      
    }
    return;
}
```

#### 连入安全的进一步完善

void CSocekt::ngx_event_accept(lpngx_connection_t oldc)

```c
        //如果某些恶意用户连上来发了1条数据就断，不断连接，会导致频繁调用ngx_get_connection()使用我们短时间内产生大量连接，危及本服务器安全
        if(m_connectionList.size() > (m_worker_connections * 5))
        {
            //比如你允许同时最大2048个连接，但连接池却有了 2048*5这么大的容量，这肯定是表示短时间内 产生大量连接/断开，因为我们的延迟回收机制，这里连接还在垃圾池里没有被回收
            if(m_freeconnectionList.size() < m_worker_connections)
            {
                //整个连接池这么大了，而空闲连接却这么少了，所以我认为是  短时间内 产生大量连接，发一个包后就断开，我们不可能让这种情况持续发生，所以必须断开新入用户的连接
                //一直到m_freeconnectionList变得足够大【连接池中连接被回收的足够多】
                close(s);
                return ;   
            }
        }
```

### 压力测试

一般要测试很多天，跑的时间长了可能 会暴露下次，跑的时间短了可能还暴露不出来；

```c
#define _CONNECTION_COUNT_          2048      //创建多少个连接，此值越大，当然recv失败的机会越大，返回10060,表示超时，setsockopt里设置了5秒的；
#define _THREAD_COUNT_              100       //准备创建这么多个线程
#define CESHIXIBIAO _CONNECTION_COUNT_ + 2000 //测试下标
#define SERVERIPADDR "192.168.200.129"
#define DEFAULT_PORT 80
#define _RECVTIMEOUT_               1500 //超时等待时间（单位：毫秒）
```

初始化socket后创建100个线程，线程具体执行如下所示：

```c
ScanThread
    socket()//所有线程加起来一共创建2048个线程
    connect()
    FuncsendrecvData()//模拟收发数据
        send()
        recv()
    FunccloseSocket()//模拟断一些socket
        closesocket();
    FunccreateSocket()//模拟把断的socket进行重连
        socket()
        connect(); 
```

建议：

(1)测试收包，简单的逻辑处理，发包；

(2)建议如果有多个物理电脑；客户端单独放在一个电脑；

建议用高性能linux服务器专门运行服务器程序
windows也建议单独用一个电脑来测试；

(3)测试什么？

1. 程序崩溃，这明显不行，肯定要解决
2. 程序运行异常，比如过几个小时，服务器连接不上了；没有回应了，你发过来的包服务器处理不了了；
3. 服务器程序占用的内存才能不断增加，增加到一定程度，可能导致整个服务器崩溃；

top -p 子进程ID：显示进程占用的内存和cpu百分比，用q可以退出；
top -p pid,推荐文章：https://www.cnblogs.com/dragonsuc/p/5512797.html

![image-20220217170617051](../images/202231-nginx整体框架解析/image-20220217170617051.png)

cat /proc/子进程ID/status   ---------其中VmRSS:     7700 kB，占用的实际内存。

![image-20220217170312468](../images/202231-nginx整体框架解析/image-20220217170312468.png)

#### 最大连接是1018

![image-20220217170445780](../images/202231-nginx整体框架解析/image-20220217170445780.png)

日志中报：CSocekt::ngx_event_accept()中accept4()失败
这个跟 用户进程可打开的文件数限制有关； 因为系统为每个tcp连接都要创建一个socekt句柄，每个socket句柄同时也是一个文件句柄；

```bash
fengyun@ubuntu:~/share/nginx$ ulimit -n
1024
```

通过`ulimit -n`可以看到进程允许打开的文件数目限制是1024。而减去标准输入输出，错误输出，日志文件，监听端口等这几个占用的fd后数目是1018。

我们就必须修改linux对当前用户的进程 同时打开的文件数量的限制；

### 惊群

惊群：1个master进程  4个worker进程

一个连接进入，惊动了4个worker进程，但是只有一个worker进程accept();其他三个worker进程被惊动，这就叫惊群；

但是，这三个被惊动的worker进程都做了无用功【操作系统本身的缺陷】；

配置nginx的worker子进程数目为4，然后借助telnet进行连接测试

```c
fengyun@ubuntu:~/share/nginx$ telnet 192.168.200.129 80
```

在ngx_epoll_process_events()加入一个测试代码`ngx_log_stderr(0,"惊群测试:events=%d,进程id=%d",events,ngx_pid); `

可以观察到

![image-20220217193302164](../images/202231-nginx整体框架解析/image-20220217193302164.png)

官方nginx解决惊群的办法：锁，进程之间的锁；谁获得这个锁，谁就往监听端口增加EPOLLIN标记，有了这个标记，客户端连入就能够被服务器感知到；

3.9以上内核版本的linux，在内核中解决了惊群问题；而且性能比官方nginx解决办法效率高很多；
reuseport【复用端口】,是一种套接字的复用机制，允许将多个套接字bind到同一个ip地址/端口上，这样一来，就可以建立多个服务器来接收到同一个端口的连接【多个worker进程能够监听同一个端口】；

但是注意目前master进程中在ngx_open_listening_sockets创建了一个监听套接字，创建了四个worker进程的监听套接字和master套接字是同一个，即使设置了reuseport仍然会产生惊群现象、

[深入浅出 Linux 惊群：现象、原因和解决方案](https://zhuanlan.zhihu.com/p/385410196)在一个 epoll 上睡眠的多个 task，如果在一个 LT 模式下的 fd 的事件上来，会唤醒 epoll 睡眠队列上的所有 task，而 ET 模式下，仅仅唤醒一个 task，这是 epoll"惊群"的根源。

看了这位腾讯 IEG 后台开发工程师的文章，我选择了试着在worker进程中使用ngx_open_listening_sockets，每个worker进程都会创建一个监听套接字listenfd，然后使用reuseport

## 性能优化

从两个方面看下性能优化问题；

软件层面：

1. 充分利用cpu，比如刚才惊群问题；
2. 深入了解tcp/ip协议，通过一些协议参数配置来进一步改善性能；
3. 处理业务逻辑方面，算法方面有些内容，可以提前做好；

硬件层面【花钱搞定】：

1. 高速网卡，增加网络带宽；
2. 专业服务器；数十个核心，马力极其强；
3. 内存：容量大，访问速度快；
4. 主板啊，总线不断升级的；

### 性能优化的实施

#### 绑定cpu、提升进程优先级

- 一个worker进程运行在一个核上；为什么能够提高性能呢？

cpu：缓存；cpu缓存命中率问题；把进程固定到cpu核上，可以大大增加cpu缓存命中率，从而提高程序运行效率；
nginx官方有一个函数worker_cpu_affinity【cpu亲和性】，就是为了把worker进程固定的绑到某个cpu核上；
ngx_set_cpu_affinity,ngx_setaffinity;

- 提升进程优先级,这样这个进程就有机会被分配到更多的cpu时间（时间片【上下文切换】），得到执行的机会就会增多；

setpriority()；

干活时进程 处于R状态，没有连接连入时，进程处于S

pidstat - w - p 3660 1   看某个进程的上下文切换次数[切换频率越低越好]

![image-20220217201040281](../images/202231-nginx整体框架解析/image-20220217201040281.png)

cswch/s：主动切换/秒：你还有运行时间，但是因为你等东西，你把自己挂起来了，让出了自己时间片。

nvcswch/s：被动切换/秒：时间片耗尽了，你必须要切出去；

- 一个服务器程序，一般只放在一个计算机上跑,专用机；

#### TCP / IP协议的配置选项

这些配置选项都有缺省值，通过修改，在某些场合下，对性能可能会有所提升；

若要修改这些配置项，要求做到以下几点：

1. 对这个配置项有明确的理解；
2. 对相关的配置项,记录他的缺省值，做出修改；
3. 要反复不断的亲自测试，亲自验证；是否提升性能，是否有副作用；

#### CP / IP协议的配置选项

1. 绑定cpu、提升进程优先级
2. TCP / IP协议的配置选项
3. TCP/IP协议额外注意的一些算法、概念等

#### 配置最大允许打开的文件句柄数

cat /proc/sys/fs/file-max  ：查看操作系统可以使用的最大句柄数
cat /proc/sys/fs/file-nr   ：查看当前已经分配的，分配了没使用的，文件句柄最大数目

```bash
fengyun@ubuntu:~/share/nginx$ sudo cat /proc/sys/fs/file-max
9223372036854775807
fengyun@ubuntu:~/share/nginx$ sudo cat /proc/sys/fs/file-nr
7872	0	9223372036854775807
```

限制用户使用的最大句柄数
 /etc/security/limit.conf文件；
root soft nofile 60000  :setrlimit(RLIMIT_NOFILE)
root hard nofile 60000

ulimit -n ：查看系统允许的当前用户进程打开的文件数限制
ulimit -HSn 5000   ：临时设置，只对当前session有效；
n:表示我们设置的是文件描述符
推荐文章：https://blog.csdn.net/xyang81/article/details/52779229

### 内存池补充说明

为什么没有用内存池技术：感觉必要性不大
TCMalloc,取代malloc();
库地址：https://github.com/gperftools/gperftools
