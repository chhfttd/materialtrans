# UDP
之前的章节说到：目前所有的网络通信都是建立在包基础上的。**包的长度不会超过几千个比特**。这些包能独立地在网络中传输，如果遇到服务器阻塞还能绕道到达目的地——包的确可能在传输的过程中发生故障。**如果网络条件很差，包可能根本无法达到目的地**。

建立了网络应用之后，设计者很快面临了一个问题：**网络通信真的能建立在这些独立、无序、不可靠的的网络数据包上吗？**如果真的有那种有序稳定的比特流，还能让客户端和服务器之间的交流就好像在本地进行一样，那么在此基础上的网络应用好设计吗？用户用的顺手吗？

在IP层上建立通信有三种可能的手段，**按流行程度降序排列**：

- 众多的网络应用都建立在**TCP（Transmission Control Protocol）**基础上。TCP提供了IP应用之间**可靠有序的数据传输**。我们在第三章探讨。
- 本章要讨论像**UDP（User Datagram Protocol）**这样的协议。**这些协议有简短的、独立的请求-应答**；以及请求丢失了再重新发送一次也并无大碍的客户端。
- **特意避开上述两种设定的协议**。这些协议在IP上开展了不同于TCP和UDP的完全不同的网络协议。

上述第三种协议十分稀少。标准操作系统的用户没了TCP或者UDP在网络上几乎无法进行任何通信。虽然书写纯粹的网络包这样的功能对于ping和nmap这样的网络嗅探程序都非常有用，但是**本书不讨论用Python建立和制作纯粹包的内容**。如果你需要用到这部分内容，可以去找一些相关的C代码的例子，然后在Python里，模仿C代码的方式调用socket。

本书首先讲述UDP。虽说UDP的使用远不及TCP普遍，但是**UDP足够简单，跟更让我们明白网络数据包是如何工作的**。这点在后面学习TCP的工作原理十分有帮助。

另一个先讨论UDP的原因是，实际上，**UDP使用起来要比TCP复杂的多**。因为用户能进行的操作很少，同时用户 还必须记得要查看丢包和包重排的情况。这样用户也能在使用TCP更加复杂的接口之前，先用UDP练练手。


### Should You Read This Chapter?
如果你想在IP的基础上进行网络编程，当然要读这章，以及下一章。上面谈到的都是十分基础的问题。无论你今后是想要从服务器获取一个页面还是想要对行业数据库做一次复杂的查询，了解一下底层网络是什么样的都对你有好处。

另外一个问题就来了？这章学到的东西你真的用得到？**可能用不上！**除非你真的需要和一个强制使用UDP进行通信的服务器进行通信，一般都会有其他通信手段供选择。坐在电脑前用UDP往出发信息的日子基本上一去不复返了。

**UDP的调度甚至都会波及到IP网络的寿命**。对于丢包问题，网络不佳的时候TCP通信会自动收回；相形之下却很少有UDP程序员去着手开发阻塞回避算法——所以能够正确实现出来的就更少了。结果造成了，**即便是十分简单的UDP网络应用也会造成网络阻塞**——不断地重复请求直到请求成功，**这种机制蚕食了大量的带宽**。

看到这，如果你还琢磨着用UDP协议的话，你更有可能用一个信息队列系统来代替UDP。看一看第八章，你就有答案了。看了第八章，你很快就会发现ØMQ能替你实现UDP的所有功能。ØMQ系统由那些曾经花好几个月时间研究操作系统，已经摸清了操作系统网络栈的效率和特性的开发人员实现，用不着你动手。

如果你真的想用IP的底层网络进行通信，那就用UDP吧。但是，无论如何都要把这章读完。否则像DNS, 实时影音传输和DHCP这些大受欢迎的协议的底层细节，你是根本搞不定的。


### Addresses and Port Numbers
我们在第一章学习了IP协议和IP地址。这些地址一般取18.9.22.69这样的形式，伴随着每一台和互联网相连的计算机。实际上，还有一些东西值得注意：**如果一个连接到网络的机器有多个网卡，一般而言这台机器会分别给每个网卡分配一个不同的IP地址**。所以其他机器可以选择性地连入你的机器。这种多接口设计也被用来降低网络冗余和提升带宽。

但是对于那些只有一块网卡的机器，通过前面的学习，我们也知道了他应该还有一个地址——127.0.0.1——也就是计算机自己。这个地址无论有没有网线、有没有WIFI一直存在。

这些IP地址使得数以万计的使用着不同网络设备计算机能够通过IP网络和电缆利用包相互通信。

到了UDP和TCP的层面上，就先不用考虑路由问题了。考虑一下一台计算机上跑具体应用的要求。我们首先注意到的就是：一台电脑在一段时间内可以跑很多很多的程序。**很多程序都要在同一个时间使用网络**：开着Chrome一边下载文件，一边用Thunderbird客户端查邮件、通过网络用pip装第三方库同时还通过SSH查看一个远程服务器的状态。这些需要**同时连接网络建立通信的程序都必须同时运行，之间还不能有冲突**。

但这个需求无论在计算机网络界还是在电磁信号处理界，这都是老大难。实现这种效果需要采用多路技术：**多个通信共享一个通道，通信之间还不能有任何干扰**。收音机频道是可以采用不同频率的电磁波来分开的。为了区分不同UDP包的目的地（要是人去做只需要看目的地的首字母就行了），IP设计师在这里用了一个十分简单的办法：**给每个UDP包上再附加一个16位的无符号数字（0-65535），用这个数来代表端口号**——应用可以连接和监听特定号码的端口。 

**假设你打算在你的计算机上配制一个DNS服务器，IP地址位192.168.1.9。为了能让其他的电脑找到这个服务器，此服务器会向操作系统申请UDP标准DNS端口——53端口——的控制权。如果这个端口没有被占用，那么此DNS服务器马上就能得到这个端口。**

**之后，再假设位于192.168.1.30的客户端程序（和上面的DNS服务器位于同一子网内）知道我们之前DNS服务器的IP地址，想要访问我们之前的服务器。那么此客户端程序就会在内存中建立一个DNS查询，之后向操作系统发送一段数据来充当UDP包。因为当包返回的时候这个包必须有办法来找到刚才发送它的客户端，也因为客户端最初并没有明确申请某个端口，所以操作系统会为此客户端随机指定一个目标端口。**

之后，这个包会朝着服务器的53端口发送，同时包上还写了：
192.168.1.30:44137——这是接收的客户端的IP地址和端口号，44137是操作系统随机指派给客户端的端口号。
目的地会这样书写：
192.168.1.9:53 ——这是发出的服务器的IP地址和端口号

这个目的地看上去十分简单——也就写了目标计算机的IP和端口号。但是对于把包发送到指定的地点已经足够了。**DNS服务器会从操作系统处接收到来自客户端的请求，同时也接收到了客户端IP和端口号。一旦服务器设定好了回应，DNS服务器会让操作系统向指定的客户端IP地址和指定的客户端端口号发送一个UDP数据包作为回应。** 

回应包内含了请求包最初所请求的资源和回应包目的地。之后，带着请求包已经到达过服务器这一信息，回应包会被发送回服务器。

### Port Number Ranges
所以UDP方案其实是非常简单的，有了IP地址和端口号就足以直接将包发送到目的地。

正如前面所述，如果两个程序要用UDP进行通信，其中之一必然要发送第一个包。**所以发起通信的那个程序（客户端）就必须知道目的地的IP地址和端口号。**而另一个只能等信息到达的程序（服务器）就不需要事先知道客户端的IP地址和端口号。

客户端-服务器模式通常指：**服务器在一个被客户端已知的IP和端口号处长期运行等待请求，并且可以对成千上万的客户端请求作出回应**。当服务器-客户端的模式不再成立的时候——也就是服务器不提供服务同时客户端也不接收服务的时候——客户端和服务器连同他们各自的socket**就都退化成为节点**了。

那客户端是怎么知道服务器的IP地址和端口号的呢？一般有三个方法：
-  习惯: 很多端口号已经被官方指定了——官方是IANA（the Internet Assigned Numbers Authority）。这就是为什么我们在之前的那个例子里**设定DNS在53端口**下运行。

-  自动配置: 像DNS这种关键服务的IP地址一般会被电脑在第一次和网络相连的时候获取到。之后再把已知的端口号和这个IP地址结合，程序就能进行关键服务。 

-  手动配置: 对于上述方法都不能解决的情况，要用其他的方法来传输IP地址或者主机名。也有许多用来**手动获取IP地址和端口号**的方法: 要求用户输入主机名称；从配置文件读取；从其他的服务来获取。

当决定到底如何分配端口号的时候（例如53就分配给了DNS服务），IANA会将全部端口划分成三个范围（对UDP和TCP端口都适用）：

-  知名端口（0-1023）：这些端口提供给那些最重要、最常用的网络协议。**在许多类Unix系统上，普通用户程序无法使用这些端口**。这个设计是为了防止用户在多用户计算机上把普通应用伪装成重要系统服务。

-  注册端口（1024-49151）：这些端口不是操作系统专用端口——比如何用户都可以编写程序然后将它绑定到5432端口，这样就把自己编写的程序伪装成了PostgreSQL数据库——但是**这些端口也都能被IANA注册成特定的网络协议端口**。同时IANA也建议你在这些端口上不要使用除了指定的网络协议之外的任何程序。

-  剩下的端口号（49152-65535）**随便使用**。我们在后面的章节会学到，这些端口其实就是操作系统为客户端随机指定端口号时所使用的**端口池**。

如果你写了一个从用户输入或者配置文件获取端口号的程序，那么用户输入端口名称而非端口数字的设计才是更加人性化的。端口名称都是标准的，Python标准库中的socket模块有一个getservbyname函数可供用户查询这些名字所对应的端口。比如我们查询一下DNS服务所在的端口:
```python
>>> import socket
>>> socket.getservbyname('domain')
53
```

到了第四章我们就能看到了：端口名可以被更加复杂的getaddrinfo（也是socket模块）函数编码。

知名服务名称的数据库一般会存储在Unix机器的file /etc/services目录下。这个文件已经被一些几十年不用的网络协议搞得乱七八糟，而且这些端口还存在。该文件另一个比较流行的版本被IANA放到了www.iana.org/assignments/port-numbers. 

接下来的讨论我们会在第三章进行，和TCP通信一起讨论。实际上，**IANA认为端口号是TCP和UDP唯一共享的资源**。IANA从来不会把一个服务的TCP通信安排成一个端口号却给它的UDP通信安排了另一个端口号；他们一般把**同一个服务的这两种通信都绑定到一个端口下面**——但今天的应用基本上只用TCP。


### Sockets
解释的够多了，下面咱们看源代码。
Python没有自己去造轮子，而是做了一个重要的决定：**Python把那些冗长的底层操作系统命令——一般是在兼容POSIX系统上用的——封装成一个对象化的接口，然后把接口提供给用户**。

很懒，但也很巧妙，还能有两个不同的原因：**第一，编程语言设计师一般不会在已经存在的网络API上再改进一次，不管网络程序员把这个接口写的多烂；第二，除了那些需要直接操作复杂的底层命令的时候，面向对象接口就足够了。**但对于需要操作底层命令的任务，封装好的接口也做不来。

实际上，这也正是为什么九十年代每天疲于处理底层命令的程序员对python的到来会感到如沐春风。最终，这种上层语言也并没有强制我们用“简洁的笨方法”来处理底层命令，工作如果需要我们还可以接着用。

Python只是把UDP和TCP连接在标准POSIX系统上的调用展示给了我们， 而没有重新发明这些东西。**标准POSIX网络操作全是围绕socket展开的**。

如果你之前用过标准POSIX系统的话，你可能遇到过这样的情况：POSIX不会让你一遍又一遍地去输入文件名，POSIX让你**用文件名造一个文件描述符**。**文件描述符代表了一个到目标文件的连接**。通过这个连接就可以访问文件了。如果你不用了，把连接断开就行了。

**socket在这里用的是完全同一个想法**：如果你想访问一条通信线路——比如一个UDP端口，**你应该创建一个抽象的socket对象然后把它绑定到你需要用的那个端口上去**。绑定成功以后，socket就帮你“抓住了”这个端口。一直到你关闭socket的时候你才失去控制权。

实际上，socket和文件描述符不仅概念上相似；甚至，**socket本身就是文件描述符**，只是碰巧**socket连接的对象是位于网络上的数据而非本地文件中的数据**。所以，socket和文件描述符和其他一般的文件描述符就大不相同了。但是POSIX还是**允许你用一些基本的文件操作手段read()和write()来操作socket**——一个 只想读写数据的程序甚至可以无差别地向socket里面读写数据。

socket在操作过程中是什么样的呢？看一下程序2-1，此程序实现了一个简单的服务器和客户端。

Listing 2–1. UDP Server and Client on the Loopback Interface

```python
#!/usr/bin/env python
# Foundations of Python Network Programming - Chapter 2 - udp_local.py
# UDP client and server on localhost
import socket, sys
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
MAX = 65535
PORT = 1060
if sys.argv[1:] == ['server']:
	s.bind(('127.0.0.1', PORT))
	print 'Listening at', s.getsockname()
	while True:
    	data, address = s.recvfrom(MAX)
    	print 'The client at', address, 'says', repr(data)
    	s.sendto('Your data was %d bytes' % len(data), address)
elif sys.argv[1:] == ['client']:
  	## print 'Address before sending:', s.getsockname()
  	s.sendto('This is my message', ('127.0.0.1', PORT))
  	print 'Address after sending', s.getsockname()
  	data, address = s.recvfrom(MAX) 
  	# overly promiscuous - see text!
  	print 'The server', address, 'says', repr(data)
else:
  	print sys.stderr, 'usage: udp_local.py server|client'
```


即便计算机没有联网，电脑也能运行这个脚本，应为服务器和客户端都用了“本地主机”的IP地址。运行，首先尝试运行一下服务器程序：（命令行）

```python
$ python udp_local.py server

Listening at ('127.0.0.1', 1060)
```
这行输出出现之后，服务器挂起并开始等待接受信息。源代码中，实际上服务器挂起总共花了三步：第一步,用socket()创建一个空的socket，现在这个socket还没有名字也没和任何程序连接，如果你现在就要和这个socket通信，会直接报错。
这个socket还是有一些参数的：比如他所属的是**AF_INET类——某个互联网协议类**；也具有**SOCK_DGRAM数据表类型**——代表了UDP通信（数据表（datagram）就是应用层传输数据段的术语）。
第二步，这个简单的服务器现在要用bind方法来请求UDP网络地址，你看到的就是一个包含了IP地址和UDP接口号的元组。
如果这个UDP端口现在已经被另外一个程序占有，那我们的服务器就不能获取这个端口了，所以会报错。试试再通过命令行启动一个服务器，你会看到下面的信息：

```python
python udp_local.py server
Traceback (most recent call last):
...
socket.error: [Errno 98] Address already in use
```

当然了，你第一次运行程序的时候也不太可能获得这条信息。笔者在写书的时候，第一次选的端口号不太好，遇到了一些麻烦。端口必须是大于1023的，否则，除非你是管理员，你就无法运行上面的程序。不过读者要注意，如果你的计算机上有SAP运行，SAP会占据1060端口从而导致上述脚本无法运行。如果真的无法运行，请尝试修改脚本中的PORT常数，对于这种情况笔者在此表示歉意。

**Python程序总可以用socket里的getsockname()方法来获取socket正在绑定的IP地址和port端口——注意2.7版本要绑定之后才行，空socket会报错。**一旦socket被绑定成功，服务器就开始准备接受请求了。接下来程序进入了while循环，重复执行recvfrom()，告诉监听的线路：**随时准备接受最大长度为MAX的信息——65535比特，这恰好也是UDP包可能达到的最大长度。**所以，我们总能看到包的全部内容。直到向服务器发送了一条请求，否则recvfrom就一直处于等待状态。

所以下面就开始创建客户端然后看看程序的运行结果（笔者在这里将服务器和客户端代码写在同一个脚本下是为了便于读者比较二者代码上的逻辑和对应关系）。在服务器还在运行的手，在操作系统里再打开一个命令行窗口，然后输入以下命令：

```python
python udp_local.py client
Address before sending: ('0.0.0.0', 0)
Address after sending ('0.0.0.0', 33578)
The server ('127.0.0.1', 1060) says 'Your data was 18 bytes'
##发送了第二次
python udp_local.py client
Address before sending: ('0.0.0.0', 0)
Address after sending ('0.0.0.0', 56305)
The server ('127.0.0.1', 1060) says 'Your data was 18 bytes'
```

再切换回到服务器的命令行窗口，你应该能看到服务器连接的每个客户端：

```python
The client at ('127.0.0.1', 41201) says, 'This is my message'
The client at ('127.0.0.1', 59490) says, 'This is my message'
```

尽管客户端代码比服务器代码简单一点。但是却**引入了几个新概念**：客户端在把任何一个IP地址传给socket之前，都会用getsockname试探一下，这就是为什么我们能看到最开始的时候全为0的那个IP地址。然后，客户端程序调用sendto()，传入目标IP地址和所要发送的信息，这个简单的调用才是发送给服务器信息最重要的一步。但是，要想实现通信，我们还需要客户端自己的IP和端口号。所以操作系统会随机指定一个端口号，从第二次调用getsockname()就能看到随机指派的结果了。


### Unreliability, Backoff, Blocking, Timeouts
因为前面的客户端和服务器都是在同一台机器上进行交流，所以上述程序是不可能出现丢包的情况——根本不是真实的网络。那该如何改进，使得它更像一个现实世界的网络呢？
我们在2-2里面做了两个改进。首先，服务器对于客户端的请求开始实施随机性接受——一半一半的几率，而不是一旦没有请求就一直等下去的模式——我们用这种方法来模拟丢包的情况。

```python
#!/usr/bin/env python
# Foundations of Python Network Programming - Chapter 2 - udp_remote.py
# UDP client and server for talking over the network
import random, socket, sys
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
MAX = 65535
PORT = 1060

if 2 <= len(sys.argv) <= 3 and sys.argv[1] == 'server':
	interface = sys.argv[2] if len(sys.argv) > 2 else ''
	s.bind((interface, PORT))
	print 'Listening at', s.getsockname()
	while True:
		data, address = s.recvfrom(MAX)
		if random.randint(0, 1):
			print 'The client at', address, 'says:', repr(data)
			s.sendto('Your data was %d bytes' % len(data), address)
		else:
			print 'Pretending to drop packet from', address

elif len(sys.argv) == 3 and sys.argv[1] == 'client':
	hostname = sys.argv[2]
	s.connect((hostname, PORT))
	print 'Client socket name is', s.getsockname()
	delay = 0.1
	while True:
		s.send('This is another message')
		print 'Waiting up to', delay, 'seconds for a reply'
		s.settimeout(delay)
		try:
			data = s.recv(MAX)
		except socket.timeout:
			delay *= 2 # wait even longer for the next request
			if delay > 2.0:
				raise RuntimeError('I think the server is down')
		except:
			raise # a real error, so we let the user see it
		else:
			break # we are done, and can stop looping
	print 'The server says', repr(data)

else:
	print >>sys.stderr, 'usage: udp_remote.py server [ <interface> ]'
	print >>sys.stderr, ' or: udp_remote.py client <host>'
	sys.exit(2)
```

先前的那个服务器告诉我们只在127.0.0.1的端口进行监听，我们现在这个服务器会监听全部本地端口，接受来自任何本地UDP端口的包。所以我们把IP地址设置成了空白：

```python
python udp_remote.py server
Listening at ('0.0.0.0', 1060)
```

每次请求收到后，服务器都生成一次随机数，并根据结果来决定是否应答请求。无论随机结果如何，服务器的运行结果都会显示出来。

那么如何写一个真实的UDP客户端呢？首先，不可靠性意味着客户端得到服务器回复的时间是任意的。一般有三种原因：太慢、丢包、服务器挂了。所以服务器得决定当等待了多长时间之后再次发送请求。这种做法可能占用服务器资源，第二次请求有可能不需要处理。所以我们这样设计客户端：先给socket引入settimeout()，使其可以延迟数秒。**这就是在告诉操作系统，客户端不想等太长时间，超出延迟时间后请抛出一个timeout异常。**

一个调用因为等待一个网络操作完成而等了很长时间叫做“block” the caller。"blocking"用来描述，recv()使得客户端在接收到下一个包之前要一直等待的情形。第六章讲服务器结构，non-blocking和blocking会有很大区别。这个客户端首先设置了0.1的延迟。对于我家的网络，ping也就十几个毫秒，基本不会出现，因为没有收到回复，客户端发送两次请求的情况。

这个客户端一个非常重要的特征在于**时间限制到达之后的处理手段，它不是没完没了地发送请求。**这个客户端用了指数补偿的技术，让试探越来越低频，发送请求的时间间隔一次比一次长。

记住，如果客户端到服务器再得到应答的时间本来就在200毫秒以上，则这个补偿算法永远会至少发送两次请求，延迟永远比0.1大呀！如果你想写一个强壮的UDP客户端，记录两次发起请求之间的时间间隔，用这个数作为基础值而不是永远0.1。当然了，也没人希望客户端死慢，一旦请求失败了，延迟就会被拉得很长。更好的方法是设置计时器，计算一次成功的连接需要多长时间，然后用这个时间逐步调节延迟。有点像移动平均的感觉。运行客户端的时候，记得给他服务器的主机

```python
$ python udp_remote.py client guinness
Client socket name is ('127.0.0.1', 45420)
```

等待至多0.1秒，服务器有回应。多尝试请求几次就能看到丢包的情况，这时候服务器会显示出：

```python
$ python udp_remote.py client guinness
Client socket name is ('127.0.0.1', 58414)
Waiting up to 0.1 seconds for a reply
Waiting up to 0.2 seconds for a reply
Waiting up to 0.4 seconds for a reply
Waiting up to 0.8 seconds for a reply
The server says 'Your data was 23 bytes'
```

但是，我们的服务器还不能识别服务器完全崩溃的情况。不过也没有必要抱怨，因为不存在和无法探明的事物本身就无法区分。这时候最好的方法就是放弃请求。

### Connecting UDP Sockets
我们之前一直用bind()来连接socket和端口：有显示的连接方式，使用bind()方法；也有隐式的方法，客户端第一次利用socket的时候也会被操作系统随机指定一个端口——这是隐式的binding。此外，UDP还有connect()方法。使用了connect()，就不必想sendto()方法那样每次都传入地址了。connect()方法会把远程端口号提前告知操作系统，**所以在使用了connect()方法之后我们利用一个send()将数据传过去就行了，而用不着调用很多个sendto()**。send还有更加重要的作用。先运行2-1：

```python
$ python udp_local.py client
Address before sending: ('0.0.0.0', 0)
Address after sending ('0.0.0.0', 47873)
```

现在客户端正在等待目标服务器的回应。如果我们现在从另一台服务器对这个客户端进行答复呢？现在，运行一个额外的python并输入以下代码，用刚才客户端发信息后返回的那个端口号重新建立一个服务器。

```python
>>> import socket
>>> s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
>>> s.sendto('Fake reply', ('127.0.0.1', 47873))
10
```

虽然这个服务器和端口1060 一点关系都没有，但客户端仍然能看到应答；但是我们却在客户端看到了这个：

```
The server ('127.0.0.1', 49371) says 'Fake reply'
```
甚至这个：
```python
The server ('192.168.5.10', 59970) says 'Fake reply from elsewhere'
```
如果一个真实的UDP客户端是这么写的，那么也太好入侵了。但如果你再次上试一下2-2的程序，你会发现这种连接方式就失效了——**因为connect指定此次连接的唯一的目标服务器**。**一旦你使用了connect命令，操作系统会拒绝任何连接你客户端的，但是端口和IP不与目标服务器匹配的的数据包**。
下面有两种写UDP客户端的方法，二者都能实现检查返回包地址的一致性：
- 直接用sendto()：每次都指定目的地和发送内容；然后用recvfrom()来接受包；之后检查接收到的地址。

- 直接使用connect()：使用了connect之后就可以直接来查看你的客户端所连接到的地址。

关于connect还有两点需要讨论：

- 第一，对一个UDP-socket（connect()方法）进行连接，并不会通过网络发送什么信息。他只是把地址和端口号写到操作系统内存中备用——使用函数recv, send的时候会用到**

- 第二，进行connect()或者直接利用返回的地址来过滤掉你不想要的包是非常不安全的。黑客肯定有办法伪造服务器回应来绕过你的地址过滤。对着另一台计算机的返回地址直接发送包的行为叫做欺骗，是协议设计者们首要考虑的事情之一

### Request IDs: A Good Idea
2-1和2-2两个例子。但你要是自己设计UDP的请求回应的话，你可能非常想给**每个请求上都加一个序列号，同时确认返回来的回应要依然包含这个序列号**。在服务器端，把每个请求中的序列号复制粘贴到回复中就行了。

这至少有两个优点：
第一，防止你被指数补偿机制发送的多个包混淆：
- 你发送请求A，等了半天没有回复；所以你重复请求A，最后你拿到了答案回复A——我们假设：你认为你的第一次请求丢失了，所以你发送了第二个请求。
- 但是，如果向服务器发送请求很慢，从客户端回复也很慢，然后你只能收到两个回复中的一个，那该怎么办呢？**如何区分这个回复到底对应哪个请求呢？**
- 这种情况下，如果你发出了请求B（**和请求A不同，二者所请求的数据不同**），然后开始等回复。那么很有可能你得到的那个回复是对于请求A产生的回复A，但是你却以为它是回复B。
- 这下就麻烦了。这个时候就能用到请求ID了。如果你给每个请求A的复制一个ID——#42496；再给每个请求B的复制一个ID——#16916。**那么等待回应B的程序会自动屏蔽掉那些请求ID不和请求B相符的回复**。这种机制能保证你 不会受到前文所述的两个回复的干扰。
- 同时，网络环境有的时候可能在服务器和客户端之间的某个地方错误地把某个包复制一份。

另一个请求ID的功能就是防止欺骗，当然如果对方能看见你的包那还是没用——那个时候IP，端口，请求ID他们都能看到，也能发送假回应。但如果他们不能观察到你的流量，那就只能盲目递向你的服务器发送包，合适的请求ID能大大降低你的客户端接收这些应答的可能性。


### Binding to Interfaces

现在我们已经看到了用bind()实现的两种绑定方法：1. 直接绑定到127.0.0.1——本机地址，这表示只接受来自同一台机器上其他程序的数据包；2. 绑定到一个空字符串表示可以接受来自任意地址的数据。显然第三个选择也是可以的，你可以绑定到外部IP接口，比如以太网连接和无线网卡。这样服务器就会监听发送到这些IP上的数据。2-2中，我们总要给bind()方法提供一个服务器字符串，所以我们做如下尝试：
首先，如果我们只把服务器绑定到外部接口会怎么样？我们在服务器字符串的位置输入任意一个操作系统告诉我们的外部IP地址，比如192.168.5.130：
```python
$ python udp_remote.py server 192.168.5.130
Listening at ('192.168.5.130', 1060)
```
从另一台机器连接到这个接口应该也没有任何问题：
```python
$ python udp_remote.py client guinness
Client socket name is ('192.168.5.130', 35084)
Waiting up to 0.1 seconds for a reply
The server says 'Your data was 23 bytes'
```
但如果你使用用“本机IP”创建客户端和这个外部IP服务器相连，则永远无法实现连接，也无法发送数据包：
```python
$ python udp_remote.py client 127.0.0.1
Client socket name is ('127.0.0.1', 60251)
Waiting up to 0.1 seconds for a reply
Traceback (most recent call last):
...
socket.error: [Errno 111] Connection refused
```
至少在笔者使用的操作系统上，也不是完全没法发送数据包：因为操作系统会检查是否存在打开但是没有向网络传输数据的接口，一但找到了，系统会毫不犹豫地说对那个接口的连接不可能。

那是不是说在localhost IP上运行的程序就永远连不到使用外部IP的服务器了呢？试试用外部地址IP作为客户端的IP：

```python
$ python udp_remote.py client 192.168.5.130
Client socket name is ('192.168.5.130', 34919)
Waiting up to 0.1 seconds for a reply
The server says 'Your data was 23 bytes'
```
你看，服务器还是会回应来自任何IP地址的客户端请求，即便是和服务器同一个IP地址都没有问题。所以，**绑定到哪个IP接口可能会限制客户端能连接的外部主机，但是决不会影响到同一台主机上的服务器和它通信**。

第二，如果我们同时运行两台服务器呢？停止所有脚本，在同一个沙箱里运行两个服务器。第一个服务器连接到本地地址，127.0.0.1：
```python
$ python udp_remote.py server 127.0.0.1
Listening at ('127.0.0.1', 1060)
```
第二台服务器连接到通配IP地址，让服务器监听任意一个本机接口：
```python
$ python udp_remote.py server
Traceback (most recent call last):
...
socket.error: [Errno 98] Address already in use
```
不行了，**操作系统不允许两个服务器监听同一个接口**，否则接到的数据包操作系统就没法分配了。
如果不用全部的IP，而只把第二个服务器运行在一个外部IP上呢？
```python
$ python udp_remote.py server 192.168.5.130
Listening at ('192.168.5.130', 1060)
```
现在同一台机器上有两个服务器运行，一个连接到了向内的1060端口，监听着回路接口的信息；另一个连接到了向外的1060端口，等待着从其他地方发送到无线网卡的信息。如果你的沙箱恰好有好几个远程接口，你甚至可以开更多服务器，每一个都占据一个远程接口。然后用一个客户端发送请求，每次都只有一个服务器接到了请求，这个服务器的IP地址将是你的客户端直接通信的那个。


这里的问题是，**操作系统不认为udp端口只有两个状态——使用中和未使用。** 相反，系统从socket入手，系统说：**socket的名字代表了一个IP地址和一个UDP端口的连接**。这些socket名字最重要的功能是：确保服务器之间互相不冲突。

最后一个忠告，既然把127.0.0.1绑定到本地服务器可以保护服务器不会被外部信息入侵，那绑定到本机外部IP能不能有相同的功效呢？不是的，这个要看是什么操作系统。

本章的三种疑难情形：
1. 服务器在本机绑定外网IP，客户端绑定在另一台机器上，向着我们机器上的外网IP服务器请求——正常通信；
2. 服务器在本机绑定外网IP，客户端绑定在本机上，朝着本机内部IP通信，上述二者端口不同——无法通信；
3. 服务器在本机绑定外网IP，客户端绑定在本机上，向着我们机器上的外网IP服务器请求，但上述二者端口不同——正常通信；
4. 服务器A在本机绑定内部IP，服务器B绑定在通配IP，二者端口不同，从一个客户端向二者发送信息——无法通信。系统不知道对A的请求是该发到服务器A还是服务器B，因为B监听任意接口。
5. 服务器A在本机绑定外网IP，服务器B在本机绑定内部IP，上述二者端口相同，服务器可以共存。

### UDP Fragmentation

UDP包不过是你的文本信息+发送端和接收端的IP和端口号。 UDP最多可以传输64kb，但是以太网只允许1500比特呀——存在分包了。所以，打包可能会被丢弃——分成的小包都不能传输到目的地的话，打包自然没法恢复。一般用户不用关心分包，但是有三种情况不一样：
- 如果考虑到了效率，你说不定想要限制协议本省，不想让包分的太碎。
- ICMP 包会被防火墙阻挡——防火墙一般会自动检测你的主机和目标主机之间的MTU值，超过这个值的包就丢失了
- 如果你想根据你的需求来调整协议中MTU的大小
  Linux是允许用户操作分包的：

Listing 2–3. Sending a Very Large UDP Packet
```python
#!/usr/bin/env python
# Foundations of Python Network Programming - Chapter 2 - big_sender.py
# Send a big UDP packet to our server.

import IN, socket, sys
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
MAX = 65535
PORT = 1060

if len(sys.argv) != 2:
	print >>sys.stderr, 'usage: big_sender.py host'
	sys.exit(2)

hostname = sys.argv[1]
s.connect((hostname, PORT))
s.setsockopt(socket.IPPROTO_IP, IN.IP_MTU_DISCOVER, IN.IP_PMTUDISC_DO)
try:
	s.send('#' * 65000)
except socket.error:
	print 'The message did not make it'
	option = getattr(IN, 'IP_MTU', 14) # constant taken from <linux/in.h>
	print 'MTU:', s.getsockopt(socket.IPPROTO_IP, option)
else:
	print 'The big message was sent! Your network supports really big packets!'
```
如果用这个客户端和家庭网络之外的任何网络连接，不难从返回信息中发现大多数无线网络支持的是1500比特传输。
```python
$ python big_sender.py guinness
The message did not make it
MTU: 1500
```
我的笔记本原先能够支持和内存一般大的包，但是也设置了限制。
```python
$ python big_sender.py localhost
The message did not make it
MTU: 16436
```
但是无论如何，查看MTU的方法一般都很难。

### Socket Options
POSIX下的socket也支持所有控制特定功能的socket选项。从getsockopt()和setsockopt()就能查看和控制这些选项。设置的时候首先要弄清楚选项所在的“option group”。
. Just like the Python calls getattr() and setattr(), the set call simply
takes one more argument:
```python
value = s.getsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, value)
```

### Broadcast
UDP的广播能力简直就是超能力，它可以把消息发给计算机所在的子网还不用去多次复制这条讯息。现在，广播已经成了明日黄花；多播技术已经深入到了很多系统和网络设备当中去了。多播可以和不再本地子网内的主机协作，使得广播技术直接失效。当时如果你想要在本地子网实现类似于积分版这样的东西，而且用户也能忍受时不时的掉线，UDP广播还能用。下面的例子就是一个接受广播包的服务器和一个能够发广播包的客户端。我们先设置了UDP选项。
Listing 2–4. UDP Broadcast
```python
#!/usr/bin/env python
# Foundations of Python Network Programming - Chapter 2 - udp_broadcast.py
# UDP client and server for broadcast messages on a local LAN

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
MAX = 65535
PORT = 1060

if 2 <= len(sys.argv) <= 3 and sys.argv[1] == 'server':
	s.bind(('', PORT))
	print 'Listening for broadcasts at', s.getsockname()
	while True:
		data, address = s.recvfrom(MAX)
		print 'The client at %r says: %r' % (address, data)
		
elif len(sys.argv) == 3 and sys.argv[1] == 'client':
	network = sys.argv[2]
	s.sendto('Broadcast message!', (network, PORT))
	
else:
	print >>sys.stderr, 'usage: udp_broadcast.py server'
	print >>sys.stderr, ' or: udp_broadcast.py client <host>'
	sys.exit(2)
```
如果你用客户端向着一个指定的服务器发送消息，这个程序和前面的就没有区别了。最重要的地方在于查看你的当地网络的设置然后使用它的广播IP作为客户端的传输目标服务器。首先建立几个服务器：
```python
$ python udp_broadcast.py server
Listening for broadcasts at ('0.0.0.0', 1060)
```
先是给每个服务器发消息，只有一个服务器能得到消息。
```python
$ python udp_broadcast.py client 192.168.5.10
```
之后使用本地网络广播地址，突然你会看到，所有的服务器都能同时受到消息。我机器上的广播地址是：
```python
$ python udp_broadcast.py client 192.168.5.255
```
如果你真的想一个一个地发送，用这个命令行：
```python
$ python udp_broadcast.py client "<broadcast>"
```
### When to Use UDP
两种情况：
- 你所使用的协议是基于UDP的
- 你的程序中用到了UDP广播，而且形成了固定的模式了。

### Summary
UDP，一种通过IP网络发送单独包的协议。 典型的UDP客户端会向UDP服务器发送包，然后根据包的来源回复信息。POSIX网络栈允许你用socket的概念来操作UDP——socket是基于端口和IP地址的网络节点，基础的操作手段全都在socket库里面。server在接受信息前要bind()端口和IP地址。
UDP客户端只能发消息，且系统会自动为他们指定端口号。因为UDP建立在网络包的具体行为上。故不可靠：会丢包。所以设计师一般要设计再次传输或者其他机制——注意不要使得网络过于阻塞。RequestID对于重复回应的问题十分重要，随机的RequestID还能防止包诈骗。使用socket的时候要分清“binding”和“connecting”。socket最强大的地方在于广播能力，这也是选择使用UDP为数不多的记过原因。
