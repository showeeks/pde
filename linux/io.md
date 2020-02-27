# IO

也许操作系统设计最混乱的方面是输入/输出。因为有如此多的设备和这些设备的应用，所以很难开发通用的、一致的解决方案。

我们首先简要讨论输入/输出设备和输入/输出功能的组织。这些主题通常属于计算机体系结构的范围，从操作系统的角度为检查输入/输出奠定了基础。

下一节检查操作系统设计问题，包括设计目标和输入/输出功能的结构方式。然后检查输入/输出缓冲；操作系统提供的基本输入/输出服务之一是缓冲功能，它提高了整体性能。

本章的下一节专门讨论磁盘输入/输出。在当代系统中，这种形式的输入/输出是最重要的，也是用户感知性能的关键。我们首先开发一个磁盘输入/输出性能模型，然后研究几种可以用来提高性能的技术。

附录J总结了二级存储设备的特性，包括磁盘和光学存储器。

如第1章所述，与计算机系统进行输入/输出的外部设备大致可分为三类:

1. 人类可读的:适合与计算机用户通信。例子包括打印机和终端，后者包括视频显示器、键盘，也许还有其他设备，如鼠标。

2. 机器可读的:适合与电子设备通信。例如磁盘驱动器、u盘、传感器、控制器和执行器。

3. 通信:适合与远程设备通信。例如数字线路驱动器和调制解调器。

不同班级之间有很大的差异，甚至有很大的差异
在每个班级里。主要区别如下:

数据传输速率:数据传输速率之间可能有几个数量级的差异。图11.1给出了一些例子。

- 应用:设备的使用对操作系统和支持实用程序中的软件和策略有影响。例如，用于文件的磁盘需要文件管理软件的支持。虚拟内存方案中用作页面后备存储的磁盘取决于虚拟内存硬件和软件的使用。此外，这些应用程序对磁盘调度算法有影响(本章后面将讨论)。作为另一个例子，终端可以由普通用户或系统管理员使用。这些使用意味着操作系统中不同的权限级别和不同的优先级。

- 控制的复杂性:打印机需要相对简单的控制接口。磁盘要复杂得多。这些差异对操作系统的影响在一定程度上由控制设备的输入/输出模块的复杂性来过滤，这将在下一节中讨论。

- 传输单位:数据可以以字节或字符流(如终端输入/输出)或更大的块(如磁盘输入/输出)的形式传输。
  数据表示:不同的设备使用不同的数据编码方案，包括字符编码和奇偶校验约定的差异。

- 错误条件:错误的性质、报告方式、后果以及可用的响应范围因设备而异。

附录三总结了执行输入输出的三种技术:

1. 程序介质/输出:进程接口/命令，在进程接口上，到一个输入/输出模块；然后，该进程会忙于等待操作完成，然后继续。
2. 中断驱动的输入输出:处理器代表进程发出输入输出命令。然后有两种可能性。如果来自进程的输入/输出指令是非阻塞的，那么处理器继续执行来自发出输入/输出命令的进程的指令。如果输入/输出指令被阻塞，那么处理器执行的下一条指令来自操作系统，这将使当前进程处于阻塞状态并调度另一个进程。
3. 直接存储器存取:直接存储器存取模块控制主存储器和输入输出模块之间的数据交换。处理器向数据存储器模块发送数据块传输请求，并且仅在整个数据块传输完毕后中断。
   表11.1显示了这三种技术之间的关系。在大多数计算机系统中，DMA是操作系统必须支持的主要传输形式。

随着计算机系统的发展，单个组件的复杂性和复杂性不断增加。这一点在输入输出功能中最为明显。进化步骤可以总结如下:

随着计算机系统的发展，单个组件的复杂性和复杂性不断增加。这一点在输入输出功能中最为明显。进化步骤可以总结如下:

1. 处理器直接控制外围设备。这可以在简单的微处理器控制的设备中看到。
2. 添加了控制器或输入/输出模块。处理器使用编程输入输出，没有中断。通过这一步，处理器在某种程度上脱离了外部设备接口的具体细节。
3. 使用与步骤2相同的配置，但是现在使用中断。处理器不需要花费时间等待输入/输出操作被执行，因此提高了效率。
4. 输入输出模块通过直接存储器存取直接控制存储器。现在，它可以在不涉及处理器的情况下将数据块移入或移出内存，除了在传输的开始和结束。
5. 指令处理器，具有专门为输入输出而设计的指令集。中央处理器指示输入输出处理器在主存储器中执行输入输出程序。输入输出处理器在没有处理器干预的情况下获取并执行这些指令。这允许处理器指定一个输入/输出活动序列，并且仅当整个序列被执行时才被中断。
6. 输入输出模块有自己的本地存储器，事实上，它本身就是一台计算机。利用这种体系结构，可以用最少的处理器参与来控制大量的输入/输出设备。这种架构的常见用途是控制与交互式终端的通信。输入/输出处理器负责控制终端的大部分任务。

随着一个人沿着这条进化的道路前进，越来越多的输入/输出功能是在没有处理器参与的情况下执行的。中央处理器越来越多地摆脱了与输入/输出相关的任务，从而提高了性能。在最后两个步骤(5和6)中，随着能够执行程序的输入/输出模块概念的引入，发生了重大变化。

关于术语的注意事项:对于步骤4到6中描述的所有模块，术语直接内存访问是合适的，因为所有这些类型都涉及到由输入/输出模块对主内存的直接控制。此外，步骤5中的输入/输出模块通常被称为输入/输出通道，而步骤6中的输入/输出模块被称为输入/输出处理器；然而，每个术语有时都适用于这两种情况。在本节的后半部分，我们将使用术语“输入/输出通道”来指代这两种类型的输入/输出模块。

图11.2一般表示直接存储器存取逻辑。直接存储器存取单元能够模仿处理器，事实上，能够像处理器一样接管系统总线的控制。它需要这样做才能通过系统总线在内存中来回传输数据。
DMA技术的工作原理如下。当处理器希望读取或写入一个数据块时，它通过向数据存储器模块发送以下信息向数据存储器模块发出命令:

- 使用处理器和直接存储器存取模块之间的读或写控制线，请求读还是写

- 所涉及的输入/输出设备的地址，通过数据线进行通信

- 存储器中读写的起始位置，通过数据线进行通信，并由数据存储器模块存储在其地址寄存器中

- 要读取或写入的字数，再次通过数据线传送并存储在数据计数寄存器中

然后处理器继续其他工作。它已经将此输入/输出操作委托给了DMA模块。直接存储器存取模块将整个数据块，一次一个字，直接传入或传出存储器，而无需经过处理器。传输完成后，直接存储器存取模块向处理器发送中断信号。因此，处理器只涉及传输的开始和结束(见图4c)。

DMA机制可以通过多种方式进行配置。一些可能性如图11.3所示。在第一个例子中，所有模块共享同一个系统总线。作为替代处理器，直接存储器存取模块使用编程的输入/输出通过直接存储器存取模块在存储器和输入/输出模块之间交换数据。这种配置虽然不贵，但显然效率低下:与处理器控制的编程输入/输出一样，一个字的每次传输消耗两个总线周期(传输请求之后是传输)。

通过集成直接存储器存取和输入/输出功能，可以大大减少所需的总线周期数。如图11.3b所示，这意味着在直接存储器存取模块和一个或多个不包括系统总线的输入/输出模块之间有一条路径。直接存储器存取逻辑实际上可以是输入/输出模块的一部分，也可以是控制一个或多个输入/输出模块的独立模块。通过使用输入/输出总线将输入/输出模块连接到直接存储器存取模块，这个概念可以更进一步(见图11.3c)。这将直接存储器模块中的输入/输出接口数量减少到一个，并提供了一种易于扩展的配置。在所有这些情况下(见图11.3b和11.3c)，直接存储器存取模块与处理器和主存储器共享的系统总线仅用于与存储器交换数据和与处理器交换控制信号。直接存储器存取模块和输入/输出模块之间的数据交换发生在系统总线之外。

在设计输入输出设备时，两个目标是最重要的:效率和通用性。效率很重要，因为输入/输出操作通常是计算系统中的瓶颈。再看图11.1，我们看到大多数输入/输出设备与主内存和处理器相比非常慢。解决这个问题的一种方法是多道程序设计，正如我们已经看到的，它允许一些进程等待输入/输出操作，而另一个进程正在执行。然而，即使当今机器的主内存很大，输入输出仍然经常跟不上处理器的活动。交换用于引入额外的就绪进程来保持处理器忙碌，但这本身就是一种输入/输出操作。因此，输入/输出设计中的主要工作是提高输入/输出效率的方案。由于其重要性，最受关注的领域是磁盘输入/输出，本章的大部分内容将用于研究磁盘输入/输出效率。

另一个主要目标是一般性。为了简单和免于出错，最好以统一的方式处理所有设备。这既适用于进程查看输入/输出设备的方式，也适用于操作系统管理输入/输出设备和操作的方式。由于器件特性的多样性，在实践中很难达到真正的通用性。可以做的是使用分层、模块化的方法来设计输入/输出功能。这种方法在低级例程中隐藏了设备输入/输出的大部分细节，因此用户进程和操作系统的高级可以根据读取、写入、打开、关闭、锁定和解锁等一般功能来查看设备。我们现在来讨论这一方法。

在第二章，在系统结构的讨论中，我们强调了现代操作系统的层次性。分层哲学是操作系统的功能应该根据它们的复杂性、它们的特征时间尺度和它们的抽象层次来分离。将这一理念专门应用于输入/输出设施导致了图11.4所示的组织类型。组织的细节将取决于设备的类型和应用程序。三个最重要的逻辑结构如图所示。当然，特定的操作系统可能不完全符合这些结构。然而，一般原则是有效的，大多数操作系统都是以这种方式处理输入输出的。

让我们首先考虑最简单的情况，即以简单方式通信的本地外围设备，例如字节流或记录流(参见图11.4a)。涉及以下几层: