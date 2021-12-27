# note
## C++
1. **右值、移动语义、完美转发及配合lambda**  
   采用**右值**和**移动语义**主要是避免进行过多的副本复制，从而节省系统资源，提高效率  

   **右值**：表达式结束时就不存在的临时对象，例如函数返回值，运算表达式等，最简单的查看方法是
   看能否对该值取地址，能则是左指，不能则是右值。  

   **左值引用、右值引用**：`&&`表示右值引用，只能绑定右值。 `&`表示左值引用，只能绑定左值。`const &`例外，左值右值都能绑定。模板类`T &&`也都能绑定（也称为通用引用），进行自动推断。 

   **右值用法**：通常的拷贝复制函数，和赋值运算函数（=号运算，也会隐式调用复制函数），这些值作为临时的参数值传递给函数时（有的编译器会优化，先不考虑编译器优化），一般会先调用构造函数构建对象，然后在会创建临时的副本接收这个对象，然后传递给函数，这样就造成了资源浪费。采用右值就会告诉编译器，不需要再创建副本，直接把对象传递给函数。具体做法就是构建**移动语义复制构造函数**和**移动语义赋值函数**，例如一个类`MyString`有一个私有变量字符串`char *m_data`,则他的移动赋值构造函数如下
   ```
   MyString(MyString&& str) noexcept
       :m_data(str.m_data) {
       str.m_data = nullptr; //不再指向之前的资源了
   }
   ```
   例如当在`push_back(MyString("hello))`时候，`MyString("hello)`作为临时变量，默认的会新建一个临时副本存储它，然后放入内存。而调用**移动语义赋值**则直接把要添加位置的内存指针指向这个临   时变量。可以理解为移动构造函数与拷贝构造不同，它并不是重新分配一块新的空间，将要拷贝的对象复制过来，而是"偷"了过来，将自己的指针指向别人的资源，然后将别人的指针修改为`nullptr`,最后把别人的指   针置空也很重要，不置空的话，这个临时对象析构后则后面继承的指针则无数据可以操控，而且后面这个对象的析构也可能会对对同一个地址释放，则会出现不可知的情况。（用`emplace_back()`等效移动语义）  
   调用`std::move()`可以把左值转换成右值  

   **完美转发**：所谓转发，就是通过一个函数将参数继续转交给另一个函数进行处理，原参数可能是右值，可能是左值，如果还能继续保持参数的原有特征，那么它就是完美的。  
   ```  
   void process(int& i){
    cout << "process(int&):" << i << endl;
   }
   void myforward(int&& i){
      cout << "myforward(int&&):" << i << endl;
      process(i);
   }

   int main(){
   int a = 0;
   myforward(2);  //右值经过forward函数转交给process函数，却称为了一个左值，
   //原因是该右值有了名字  所以是 process(int&):2
   myforward(move(a));  // 同上，在转发的时候右值变成了左值  process(int&):0
   // forward(a) // 错误用法，右值引用不接受左值
   }  
   ```     
   调用模板类和`std::forward()`则可以解决  
   ```  
   template<typename T>
   void myforward(T && t){
      process(std::forward<T>(t));
   }  
   ```  
2. **多线程**:
   1. `unique_lock` `lock_guard`
   2. 条件变量
   3. `future` `promise` `async



   

  
## 计算机网络
   1. **tcp长肥胖管道问题**  
   具有大的带宽时延乘积的网络被称为长肥网络（LongFatNetwork，即LFN），而一个运行在LFN上的TCP连接被称为长肥管道。使用长肥管道会遇到多种问题。
      1. TCP首部中窗口大小为16bit，因此窗口大小最大为65535字节，这就将发送方发送但未被确认的数据的总长度限制到了65536字节。对于LFN管道，这可能会出现所有的数据都还未到达接收方，但是发送方已        受限于窗口大小而不能继续发送的情形，这就极大的降低了网络的吞吐量。扩大窗口选项可以解决这个问题。
      2. 根据TCP的拥塞控制，丢失分组会导致连接进行拥塞控制，即便是由于冗余ACK而进入了快速恢复，也会使得拥塞窗口降低一半，而如果是由于超时进入了慢启动，则拥塞窗口会变为1，无论是哪一种情形，发        送方允许被发送的数据量都大量减小了，这会导致网络吞吐量降低。选择确认（SACK）可以用来部分避免该问题，采用该技术使得接收方可以有选择的对无序到达的报文段进行确认而不是采用累积确认，这样被        确认的报文段就不会超时，也不会有冗余的ACK。
      3. TCP对每个字节数据使用一个32bit无符号的序号来进行标识。TCP定义了最大的报文段生存时间（MSL）来限制报文段在网络中的生存时间。但是在LFN网络上，由于序号空间是有限的，在已经传输了             4294967296个字节以后序号会被重用。如果网络快到在不到一个MSL的时候序号就发生了回绕，网络中就会有两个具有相同序号的不同的报文段，接收方将无法区分它们的顺序。在一个千兆比特网络               （1000Mb/s）中只需要34秒就可以完成4294967296个字节的发送。使用TCP的时间戳选项的PAWS(ProtectionAgainstWrappedSequencenumbers)算法（保护回绕的序号）可以解决该问题。
   2. **七种TCP定时器**  
      1. **建立连接定时器**： TCP连接建立的时，发送SYN后，没有应答重复发送
      2. **重传定时器**： TCP连接建立后传输数据时候，没有收到ack确认
      3. **延迟发送ACK定时器**： 延迟发送ack确认序号，可以让要发送的seq需要一起发送
      4. **坚定计时器**：当收到发送方给定窗口大小为0时启动，如果没有对方没有后续的报文发送则计时器发送探测报文，防止窗口一直是0，无法发送数据
      5. **keep-alive**：当双方无数据交换的时候发送（一般两小时后），更有效率的方法是发送心跳报文
      6. **FIN_WAIT_2**： 当TCP四次挥手释放的时候，发送FIN后收到对方的ACK确认后启动，防止无限等待对方。
      7. **TIME_WAIT**:四次挥手的主动方最后一此挥手后启动，一般2MSL时间，一个作用是防止对方没接受到确认报文导致没收到重传的FIN报文，另一个是让这次释放过程中滞留的报文信息消失，影响以后的连        接。
   3. **半连接/全连接**：  Linux内核协议栈为一个tcp连接管理使用两个队列，一个是半链接队列（用来保存处于SYN_SENT和SYN_RECV状态的请求），一个是全连接队列（accpetd队列）（用来保存处于        established状态，但是应用层没有调用accept取走的请求）。
   4. **DNS，递归/迭代查询；什么时候UDP，什么时候TCP**  
      1. DNS 查询响应报文大于 512 字节时
      2. DNS 主、辅助服务器之间，进行区域传送时: 触发 DNS 区域传送的情况有两种：  
         1. 新上线一台辅助服务器，会从主服务器执行区域传送，进行同步数据。
         2. 辅助服务器会定时(通常是 3 小时)，向主服务器查询，以便了解到主服务器的数据是否发生变动，如果变动，也会触发一次区域传送。  
            区域传送会使用 TCP 协议，一方面是为了保证数据的可靠，另一方面此时传送的数据，也远比一个查询或响应大的多。
      3. **递归查询**: 所谓递归查询就是：如果主机所询问的本地域名服务器不知道被查询的域名的IP地址，那么本地域名服务器就以DNS客户的身份，向其它根域名服务器继续发出查询请求报文(**即替主机继续查询**)，而**不是让主机自己进行下一步查询**。因此，递归查询返回的查询结果或者是所要查询的IP地址，或者是报错，表示无法查询到所需的IP地址.
      4. **迭代查询**：当根域名服务器收到本地域名服务器发出的迭代查询请求报文时，要么给出所要查询的IP地址，要么告诉本地服务器：“你下一步应当向哪一个域名服务器进行查询”。然后让本地服务器进行后续的查询。根域名服务器通常是把自己知道的顶级域名服务器的IP地址告诉本地域名服务器，让本地域名服务器再向顶级域名服务器查询。顶级域名服务器在收到本地域名服务器的查询请求后，要么给出所要查询的IP地址，要么告诉本地服务器下一步应当向哪一个权限域名服务器进行查询。最后，知道了所要解析的IP地址或报错，然后把这个结果返回给发起查询的主机。**返回多次，可能是提示查询的dns服务器地址，也可能是结果/报错**
         

      

## 网络编程
### 信号
1. **signal信号** :signal信号是进程之间相互传递消息的一种方法，信号全称为软中断信号  
信号只是用来通知某进程发生了什么事件，无法给进程传递任何数据，进程对信号的处理方法有三种：   
   * 第一种方法是，忽略某个信号，对该信号不做任何处理，就象未发生过一样。  
   * 第二种是设置中断的处理函数，收到信号后，由该函数来处理。  
   * 第三种方法是，对该信号的处理采用系统的默认操作，大部分的信号的默认操作是终止进程。
2. **常见信号** :下面是常用到的几个信号  
   信号名|信号值|信号动作|发出的原因
   --|:--:|:--:|--:
   SIGINT|2|终止进程|键盘ctrl+c
   SIGKILL|9|强制终止，无法忽略|发出命令kill -9
   SIGSEGV|11|缺省的动作是终止进程并进行内核映像转储（core dump）|无效的内存引用
   SIGTERM|15|终止进程|发出命令kill
   SIGCHLD|20,17,18|忽略信号|子进程结束信号  
3. **可靠信号/不可靠信号**：1-32是不可靠信号， 34-64是可靠信号。  
   不可靠信号主要会出现信号丢失问题：如果多次发送同一个信号可能会出现丢失，只接受到部分信号。  
4. **信号中断**: 多次同一个信号（可靠信号）会阻塞等待，如果发送其他信号，会优先处理新的信号，等处理函数结束再处理原信号。
5. **信号有什么用**： 服务程序运行在后台，如果想让中止它，强行杀掉不是个好办法，因为程序被杀的时候，程序突然死亡，没有释放资源，会影响系统的稳定，用Ctrl+c中止与杀程序是相同的效果。如果能向后台程序发送一个信号，后台程序收到这个信号后，调用一个函数，在函数中编写释放资源的代码，程序就可以有计划的退出，安全而体面。  
6. **信号的用法**：
   * `sighandler_t signal(int signum, sighandler_t handler);` 第一个参数：信号值， 第二个参数处理函数(SIG_IGN忽略)；
   * `int kill(pid_t pid, int sig);`   
   kill函数将参数sig指定的信号给参数pid 指定的进程。参数pid 有几种情况：  
      1. pid>0 将信号传给进程号为pid 的进程。  
      2. pid=0 将信号传给和目前进程相同进程组的所有进程，常用于父进程给子进程发送信号，注意，发送信号者进程也会收到自己发出的信号。
      3. pid=-1 将信号广播传送给系统内所有的进程，例如系统关机时，会向所有的登录窗口广播关机信息。  
   sig：准备发送的信号代码，假如其值为零则没有任何信号送出，但是系统会执行错误检查，通常会利用sig值为零来检验某个进程是否仍在运行。  
   返回值说明： 成功执行时，返回0；失败返回-1，errno被设为以下的某个值.  
   EINVAL：指定的信号码无效（参数 sig 不合法）。  
   EPERM：权限不够无法传送信号给指定进程。  
   ESRCH：参数 pid 所指定的进程或进程组不存在。  
   * 利用`sigse`设置信号集，进行信号的阻塞
   * 更新的`sigeaction` 设置了结构体进行封装，可以携带信号外的其他数据，具体查找资料，自己没用过。
7. **僵尸进程**：一个子进程在调用return或exit(0)结束自己的生命的时候，其实它并没有真正的被销毁，而是留下一个僵尸进程.
   * **僵尸进程的危害**： 僵尸进程是子进程结束时，父进程又没有回收子进程占用的资源。僵尸进程在消失之前会继续占用系统资源。
   * **僵尸进程的解决方法**： 解决僵尸进程的方法有两种
      * 子进程退出之前，会向父进程发送一个信号，父进程调用wait函数等待这个信号，只要等到了，就不会产生僵尸进程。这话说得容易，在并发的服务程序中这是不可能的，因为父进程要做其它的事，例如等待客户端的新连接，不可能去等待子进程的退出信号
      * 另一种方法就是父进程直接忽略子进程的退出信号，具体做法很简单，在主程序中启用以下代码： `signal(SIGCHLD,SIG_IGN); `
      
### I/O多路复用
1. **select** : `int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)返回值：就绪描述符的数目，超时返回0，出错返回-1`
   该函数准许进程指示内核等待多个事件中的任何一个发送，并只在有一个或多个事件发生或经历一段指定的时间后才唤醒  
   select的几大缺点：
   * 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
   * 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
   * select支持的文件描述符数量太小了，默认是1024(每个描述符是通过位图映射的）
   * select是水平触发(LT)，即当此select检测到了fd的事件但是没有处理或者没有完全处理完，下次调用select还会报告该fd(即使该fd_set中没有事件发生）
2. **poll**: `int poll ( struct pollfd * fds, unsigned int nfds, int timeout);`
   ```C++
   struct pollfd {
   int fd;         /* 文件描述符 */
   short events;         /* 等待的事件 */
   short revents;       /* 实际发生了的事件 */
   } ; 
   ```
   每一个pollfd结构体指定了**一个**被监视的文件描述符，可以**传递多个结构体**，指示poll()监视多个文件描述符。每个结构体的events域是监视该文件描述符的事件掩码，由用户来设置这个域。revents域是文件描述符的操作结果事件掩码，内核在调用返回时设置这个域。
   在select中，不能直接传入原本的fd_set, 而需要创建一个**副本传入**，因为select会**直接修改fd_set会造成原本的fd_set丢失**的情况。
   而在poll中，有事件发生的fd会体现在在revents中，不需要创建临时副本。
   poll的缺点：
   * 与select一样，也需要遍历发生事件的fd，也同样需要用户态和内核态之间切换时间开销过大
   * 也是和select一样的水平触发(LT)，存储是数组(结构体数组)的形式

3. **epoll**:相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。
   ```C++
   int epoll_create(int size);
   int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
   ```
   * `int epoll_create(int size);` : 创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
    * `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);` epoll的事件注册函数,它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监    听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
   EPOLL_CTL_ADD：注册新的fd到epfd中；
   EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
   EPOLL_CTL_DEL：从epfd中删除一个fd；
   第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
   ```C++
   struct epoll_event {
     __uint32_t events;  /* Epoll events */
     epoll_data_t data;  /* User data variable */
   };
   ```
   与poll相似  
   events可以是以下几个宏的集合：  
   EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭；  
   EPOLLOUT：表示对应的文件描述符可以写；  
   EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来；  
   EPOLLERR：表示对应的文件描述符发生错误；  
   EPOLLHUP：表示对应的文件描述符被挂断；  
   **EPOLLET**： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。  
   EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里  
   ET模式：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只    会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你  
   ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务    饿死。  
4. **总结比较**： 
   1. **功能**:select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。  
      * select 会修改描述符，而 poll 不会；  
      * select 的描述符类型使用数组实现，FD_SETSIZE 大小默认为 1024，因此默认只能监听少于 1024 个描述符。如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；而 poll 没有描述符       数量的限制；  
      * poll 提供了更多的事件类型，并且对描述符的重复利用上比 select 高。  
      * 如果**一个线程对某个描述符调用了select或者poll，另一个线程关闭了该描述符，会导致调用结果不确定**。  
   2. **速度**:select和poll 速度都比较慢，每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区(轮询的方式查看)。  
   3. **可移植性**:几乎所有的系统都支持 select，但是只有比较新的系统支持 poll 
   4. **epoll**:
      * epoll_ctl() 用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵**红黑树**上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调       用 epoll_wait() 便可以得到事件完成的描述符。
      * 从上面的描述可以看出，epoll 只需要将描述符从进程缓冲区向内核缓冲区拷贝一次，并且进程不需要通过轮询来获得事件完成的描述符。
      * epoll 仅适用于 Linux OS。
      * epoll 比 select 和 poll 更加灵活而且没有描述符数量限制。
      * epoll 对多线程编程更有友好，一个线程调用了 epoll_wait() 另一个线程关闭了同一个描述符也不会产生像 select 和 poll 的不确定情况。
   5. **应用场景**：
      * **select**: select的timeout参数精度为微秒，而poll和epoll为毫秒，因此select更加适用于实时性要求比较高的场景，比如核反应堆的控制。select可移植性更好，几乎被所有主流平台所支持.
      * **poll**: poll没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用poll而不是select。
      * **epoll**: 只需要运行在Linux平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。需要同时监控小于1000个描述符，就没有必要使用epoll，因为这个应用场景下并不能体现epoll的优势。需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用epoll。因为epoll中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过epoll_ctl()进行系统调用，频繁系统调用降低效率。并且epoll的描述符存储在内核，不容易调试。

   6. **主线城和工作线程分开的原因**
      * 工作线程一般处理网络I\O,这样的时间比较长，如果也交给主线城来处理，则后续的网络连接不能及时响应
      * 便于负载均衡，主线程可以轮询分配工作任务，避免某个线程过忙
      * 可以让空闲的线程干些其他的事情

## 操作系统
   1. **PCB含有三大类信息**：
      1. 进程标识，哪个程序在执行，执行了几次（本进程的标识），产生者标识（父进程标识），用户标识
      2. 处理机状态信息保存区，主要就是寄存器，保存进程的运行现场信息：
         * 用户可见寄存器，程序使用的数据，地址
         * 控制和状态寄存器，程序计数器pc，程序状态字PSW
         * 栈指针，过程调用/系统调用/中断处理和返回时需要用到
      3.  进程控制信息
         * 调度和状态信息，用于操作系统调度进程并占用处理机使用。运行状态？等待？进程当前的执行现状
         * 进程间通信信息，各种标识、信号、信件等
         * 进程本身的存储管理信息，即指向本进程映像存储空间的数据结构，内存信息，占了多少？要不要回收？
         * 进程所用资源，打开使用的系统资源，如文件
         * 有关数据结构连接信息，父进程，子进程，构成一个链，进程可以连接到一个进程队列，或链接到其他进程的PCB
