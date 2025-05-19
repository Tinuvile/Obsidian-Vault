## Problem：Building a Network

>[!note] 摘录
>At one time, the term _network_ meant the set of serial lines used to attach dumb terminals to mainframe computers. Other important networks include the voice telephone network and the cable TV network used to disseminate video signals. The main things these networks have in common are that they are specialized to handle one particular kind of data (keystrokes, voice, or video) and they typically connect to special-purpose devices (terminals, hand receivers, and television sets).
>What distinguishes a computer network from these other types of networks? Probably the most important characteristic of a computer network is its generality. Computer networks are built primarily from general-purpose programmable hardware, and they are not optimized for a particular application like making phone calls or delivering television signals. Instead, they are able to carry many different types of data, and they support a wide, and ever growing, range of applications. Today’s computer networks have pretty much taken over the functions previously performed by single-use networks.

各种网络（比如电话网络、有限电视网络）的主要共同点就它们专门用于处理特定类型的数据，并且通常连接到专用设备。而计算机网络与其他网络的区别就在于其通用性。计算机网络没有针对特定应用进行优化，相反，它能承载多种不同类型的数据。

---

## Application

### Classes of Applications

最基础的应用自然是浏览器页面。在表层的浏览器页面下，页面上的每个可选对象都绑定着一个标识符，指向下一个待查看的页面或对象，它被称为统一资源定位符*URL*，它为浏览器中所有可查看对象提供了识别方式。例如：
```html
http://www.cs.princeton.edu/~llp/index.html
```
这个 URL 中，`http`表示使用超文本传输协议*HTTP*下载该页面，`www.cs.princeton.edu`是托管该页面的服务器名称，`~llp/index.html`则标识了 Larry 在该网站的主页。
在点击这样一个 URL 的过程中，互联网上就进行了多次消息交换：最多六条消息用于将服务器名称`www.cs.princeton.edu`转换为其 IP（Internet Protocol）地址`128.112.136.35`；三条消息用于在浏览器与该服务器之间建立 TCP（Transmission Control Protocol）协议；四条消息用于浏览器发送 HTTP “GET” 请求以及服务器返回所请求的页面、以及双方确认收到消息；四条消息用于关闭 TCP 连接。

互联网的另一个广泛应用是流式音频和视频的传输。它们与传统文本、图形与图像传输不同，接收方会在数据到达的同时就开始播放，而非先下载整个文件，但播放仍然需要保持连贯性。这会影响网络如何支持这类应用。

还有一个不同的应用类别是实时音频和视频。它们比流媒体应用有更严格的时间限制，比如一些网络电话或视频会议应用。这类通常涉及双向的音频或视频流，而流媒体应用大多是单向传输的。

---

## Requirements

### Stakeholders

此书将包含三个群体视角对网络的需求：
- _application programmer_：应用程序员会列出应用所需的网络服务；
- _network operator_：网络运营商会列出易于管理和维护的系统特性；
- _network designer_：网络设计师会提出 cost-effective 设计的属性；

### Scalable Connectivity

这部分以互联网为模型讨论可拓展性。

在最底层，网络可以由两台或更多计算机通过某种物理介质（如同轴电缆或光纤）直接连接而成，这种物理介质称为**链路**，其连接的计算机或硬件设备称为**节点**。可以仅限于一对节点（点对点），也可多个节点共享一条物理链路（多路访问）。无线链路，如蜂窝网络和Wi-Fi网络提供的链路，就是多路访问链路的重要类别。多路访问链路在覆盖的地理距离与可连接的节点数量方面是有限的，因此它们通常用于实现“最后一公里”，将终端用户与网络的其余部分连接起来。

但两个节点之间的连通性并不必然意味着它们之间存在直接的物理连接，通过一组协作节点也可以实现间接连通。

![[Pasted image 20250513083740.png]]

这个图展示了一组节点，每个节点都连接到一个或多个点对点链路。而那些至少连接两条链路的节点运行软件，将接收到的数据从一条链路转发到另一条链路。这些转发节点构成了**交换网络**。交换网络有多种类型，最常见的两种是**电路交换**和**分组交换**。前者最显著的应用是电话系统，后者则应用于大多数计算机网络。

分组交换网络的重要特点是，此类网络中的节点彼此间发送的是离散的数据块。每个数据块称为分组或报文。它采用一种称为存储转发*store-and-forward*的策略。存储转发网络中的每个节点首先通过某条链路接收完整的数据包，将其存储在内部内存中，然后将完整的数据包转发到下一个节点。

相比之下，电路交换网络首先建立跨越一系列链路的专用电路，然后允许源节点通过该电路向目标节点发送比特流。在计算机网络中使用分组交换而非电路交换的主要原因是效率。

上图中区分了实现网络的内部节点（称为交换机*switches*，主要功能是存储和转发数据包）与使用网络的外部节点（称为主机*hosts*）。**云**是计算机网络中最重要的图标之一，我们可以用云来表示任何类型的网络。

![[Pasted image 20250513084910.png]]

一组计算机间接连接的第二种方式如上图所示。一组独立的网络（云）相互连接形成一个互联网络。按照惯例，用*internet*指代一般的网络互联网络，用*Internet*指代日常使用的TCP/IP互联网。连接到两个或更多网络的节点通常称为**路由器**或**网关**，其作用是将消息从一个网络转发到另一个网络。

为了让每个节点能指明它希望与网络上的哪个其他节点通信，会为每个节点分配一个**地址**。地址是一个用于标识节点的字符串。基于目标节点地址系统地决定如何转发消息的过程称为**路由**。

除了特定节点外，网络的另一个要求是支持多播和广播地址。

>[!info] Key Takeaway
>我们可以递归地将网络定义为由两个或多个通过物理链路连接的节点组成，或者由两个或多个通过节点互连的网络构成。即网络可以通过网络的嵌套来构建。在最底层，网络由某种物理介质实现。

### Cost-Effective Resource Sharing

存在多个主机希望共同使用一条链路的情况，也即多路复用（*multiplexing*），多个用户发送的数据在构成网络的物理链路上进行多路复用。

![[Pasted image 20250513090033.png]]

上图中，左侧的三台主机通过共享一个仅包含单一物理链路的交换网络，向右侧的三条主机发送数据。三组数据量被*Switch 1*复用至同一条物理链路上，随后由*Switch 2*解复用回独立的数据流。

这有好几种不同的方法实现：
- 一种常用的方法是同步时分复用（*STDM*），它的思想是把时间划分成等长的时隙，并以轮转的方式让每个数据流都有机会通过物理链路发送数据。这种方法的问题在于，可能存在链路空闲，轮转中某条流没有数据发送。
- 另一种方法是频分复用（*FDM*），它的思想是以不同的频率在物理链路上传输每个数据流。

这两种实现都仅适用于最大流数量固定且预先可知的场景。为了解决这些缺点，我们使用统计复用（*statistical multiplexing*）。它的物理链路在时间上类似于STDM，也是共享的。但数据是根据每个流的需求传输，而不是在预定的时隙。同时，它规定了每个数据流在给定时间内允许传输的数据库大小的上限，这种大小受限的数据块通常被称为数据包（*packet*），这样可以确保所有数据流都能获得在物理链路上传输的机会。

![[Pasted image 20250513091100.png]]

若交换机接收数据包的速度超过了共享链路的承载能力，它被迫将数据包缓冲在内存中最终耗尽，而被迫丢弃一些数据包，则称为拥塞（*congested*）。

### Support for Common Services

计算机网络的另一个要求是，连接到网络的主机上运行的应用程序必须能够以有意义的方式进行通信。

我们将网络视为提供逻辑通道，应用层进程通过这些通道相互通信，每个通道提供该应用所需的服务集。

### Identify Common Communication Patterns

网络最早支持的应用是文件访问程序，如文件传输协议（*FTP*）或网络文件系统（*NFS*）。请求访问文件的进程称为**客户端**（*client*），支持文件访问的进程称为**服务器**（*server*）。

读取文件时，客户端向服务端发送一条小型请求消息，服务器返回一条包含文件数据的大型请求消息作为响应。写入则相反。

同时还有诸如视频传输的需求。

为实现这个，会提供两种类型的通道：请求/应答通道（*request/reply*）和消息流通通道（*message stream*）。请求/应答通道确保一方发送的消息都能被另一方接收，并保护其上数据的隐私性和完整性，未授权无法读取与修改双方交换的数据。消息流通通道无需保证所有消息送达，即使部分视频帧未接收，视频应用仍能正常运行，它需要保证已送达的消息保持发送时的顺序。

### Reliable Message Deliver

网络设计者主要关注三类常见故障：
- 数据包在物理链路传输时，数据中可能突发比特错误。
- 整个数据包在网络中丢失。
- 物理链路被切断或者连接的计算机崩溃。

### Manageability

>[!warning] 网络需要被管理

---

## Architecture




