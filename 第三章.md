TCP

TCP(传输控制协议)是互联网的主力军。它于1974年首次被定义。通过TCP，一个程序可以向另一个程序不断发送数据流，直到数据内容被完全送达。除非由于网络连接不佳造成的网络问题出现外，TCP能够确保信息传输的完整、有序、无重复。
用来传输文档和文件的协议几乎全都是以TCP为基础的，包括HTTP和所有传输E-mail的主要手段。TCP也能作为实现计算机或者人之间长距离交流的协议的基础，比如SSH和许多交流协议。
互联网发展初期，有的时候可以通过将应用建立在UDP上，然后手动调整每个数据包的大小和时间来略微地提高协议的传输效率。但是现代TCP，在经历了三十年的研究、提升和革新之后，相比之下就更加智能。 目前，即使那些对于传输效率要求十分苛刻的应用程序（如信息队列）都会选择TCP为基础建立通讯。

How TCP Works

如同第二章学到的那样，真实的网络并不稳定，在传输中常常会出现丢包的情况，或者多创建了某个包的一个复制的情况，或者不按照既定顺序进行传输。对于像UDP这样的纯粹的数据包型协议，特定程序的应用代码必须考虑到信息是否能够送达，如果没有送达要如何进行恢复。但是对于TCP，包本身都是隐藏的。程序可以直接向目标程序发送数据流，如果不能送达则可以进行重新传输。

TCP的传统定义由1981年的RFC793完成，后续的很多RFC文件也都对TCP定义进行了修正和改进。

为什么TCP能够提供稳定连接？首先，TCP把我们在第二章讨论的两种机制结合了起来。这里，我们只能通过UDP来实现TCP，所以必须亲自书写着两种机制。但若直接使用TCP，这些机制就都成为了由操作系统提供的内建方法。

首先，每个数据包都配备了一个序列号。这样，数据接收端的操作系统就能把接收到的数据包按顺序组装起来，出现数据包丢失的情况也能够从组装情况及时发现并要求传输端重新传输。这里提到的序列号并不是1，2，...这样的整数序列，实际情况是TCP使用了一个计数器来记录所传输数据的比特。所以，对于一个大小为1024比特的数据包，若它还具有一个7200的序列号，那么它之后紧接着的那个数据包的序列号就应该是8224。这种设计使得繁忙的网络栈不需要去记忆将数据流分割成包的具体方式。如果发生了重新传输，那么数据流可能被其他的方法分割，接收端仍能够把数据包重新组合起来。

在优质的TCP实现中，初始序列号是随机选择的。这样不法分子就无法假设每个链接都以0比特作为初始序列号开始传输了，更不能按照当前截获的包来推测全部数据的序列号并且伪造数据流了。


TCP传输会一次性地发送所有包，不去单独等待某个包的回复。传输端在任意时刻愿意传输的数据量叫做TCP的窗口大小。

在接受端的TCP实现可以规定传输断的窗口大小，这样接收端就能控制传输的速度甚至暂停掉连接，这种设计叫做流控制。这种设计禁止了额外数据包的传输。


最后， 如果TCP发现数据包被丢弃了，那么TCP将判定网络处于阻塞状态并不再以原有的速度发送数据。对于那些数据包偶尔会因为噪声而丢失的传输手段——无线网络或无线媒体——这个性质简直是灾难性的。TCP甚至能毁掉一个运行良好的连接，必须通过重启路由器才能恢复。恢复之后，接收端和传输端会断定网络被流量过载，并会时不时地拒绝传输数据。

真实的协议和上述的特点相比稍有差别，但是我们希望上述描述能够给你一个对于TCP和TCP的工作原理的较为直观的感受。同时要记住，在使用TCP连接的时候，你的应用们只能观察到数据流。而组成数据流的数据包和序列号早就被操作系统的网络栈 隐藏起来了。


When to Use TCP

如果你的网络程序和我的一样，那么你从Python实现的大多数的网络通信都是基于TCP的。 实际上，你的职业生涯几乎都在和TCP打交道，而不是编写UDP。

因为TCP基本上成了两个程序之间通信的默认协议，所及我们更应该看几个例子，来知道TCP对于某些特定的数据并不是最佳选择。

首先，对于那些客户端只想给服务器发送简短、单次的请求，然后就不进行后续通讯的任务，TCP显得很笨重。比如，传输和接受两个主机要想建立一个TCP连接需要用到三个数据包：SYN, SYN-ACK, ACK。关掉这个TCP连接也需要三到四个包。所以，一个请求就涉及到了6个数据包。这种情况下，开发者会直接使用UDP进行开发。

而UDP的优点在于，客户端和服务器之间的连接不会长久地存在；相反TCP连接就有可能耗尽所有的端口号来维持对于这些客户端的连接。

第二种情况是：如果对于包丢失的情况，程序有更佳的解决方案时，TCP就不那么合适了。比如一个语音通讯程序，如果只是丢失了一秒钟通讯的数据，那么完全没有必要将相同的信息重新发送，知道完全送达。相反，客户端可以直接覆盖丢失的那一秒信息。这个对于TCP来说几乎不可能实现，所以还是需要使用UDP。


What TCP Sockets Mean

如同第二章说到的UDP情形那样，TCP也会使用端口号来区分运行在同一IP地址下面的不同应用，并且遵从既定的端口号码惯例。你可以重新阅读 “Addresses and Port Numbers” 来复习相关细节。

正如我们在前面章节所看到的，用UDP进行通讯只需要一个SOCKET：一个服务器可以打开一个数据表端口然后从上千个不同的客户端那里接受数据包。 可以使用数据表socket利用connect()方法连接一个特定的机器，然后你就能用send()方法朝这个机器发送信息，并且可以使用recv()接受仅仅从这个方向发送过来的数据包。这种connection的思想十分简便。connect()方法的效果和你的程序决定利用sendto()方法向一个地址发送信息，并且忽略其他地址的返回信息是完全相同的。但是像TCP这样的一个有状态的流协议，connect()方法成了一个可作为网络传输基础的基本方法。实际上，现在你的操作系统已经可以开始吧握手协议直接了，这种协议被描述成一种能够使得通信两端的TCP流待命的协议。
同时，这也意味着一个TCP的connect()方法可能失效。远程主机可能不会应答；可能拒绝连接；也可能发生更加模糊的协议错误，比如间接地收到一个RST包。
因为一个流连接需要有一个介于两台主机之间的持续连接，所以需要要求其他主机随时待命接收你的连接。在服务器端（本章中将不调用connect()，但是却接收SYN包的称之为服务器端）一个引入的连接会产生一个甚至更加重要的事件，创建一个socket。之歌是因为标准POSIX系统的TCP接口实际上是由两个完全不同的socket组成的：被动监听型和主动连接型。


•  A passive socket holds the “socket name”—the address and port number—at
which the server is ready to receive connections. No data can ever be received or
sent by this kind of port; it does not represent any actual network conversation.
Instead, it is how the server alerts the operating system to its willingness to receive
incoming connections in the first place.
•  An active, connected socket is bound to one particular remote conversation
partner, who has their own IP address and port number. It can be used only for
talking back and forth with that partner, and can be read and written to without
worrying about how the resulting data will be split up into packets—in many
cases, a connected socket can be passed to another POSIX program that expects to
read from a normal file, and the program will never even know that it is talking to
the network!
Note that while a passive socket is made unique by the interface address and port number at which
it is listening (so that no one else is allowed to grab that same address and port), there can be many
active sockets that all share the same local socket name. A busy web server to which a thousand clients
have all made HTTP connections, for example, will have a thousand active sockets all bound to its public
IP address at port 80. What makes an active socket unique is, rather, the four-part coordinate:
(local_ip, local_port, remote_ip, remote_port)

It is this four-tuple by which the operating system names each active TCP connection, and
incoming TCP packets are examined to see whether their source and destination address associate them
with any of the currently active sockets on the system.



