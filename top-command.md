# Top 的解释

这将启动一个交互式命令行应用程序，类似于下面截图中的应用程序。输出的上半部分包含进程和资源使用的统计数据，而下半部分包含当前正在运行的进程的列表。您可以使用箭头键和向上/向下翻页键浏览列表。如果你想退出，只需按“q”。


top有许多变体，但是在本文的其余部分，我们将讨论最常见的变体——procp-ng包附带的变体。您可以通过运行以下命令来验证这一点:

在屏幕的最左上角(如上图所示)，顶部显示当前时间。接下来是系统正常运行时间，它告诉我们系统运行的时间。例如，在我们的示例中，当前时间是“15:39:37”，系统已经运行了90天15小时26分钟。

接下来是活动用户会话的数量。在本例中，有两个活动的用户会话。这些会话可以在TTY(物理上在系统上，通过命令行或桌面环境)或PTY(例如终端仿真器窗口或SSH)上进行。事实上，如果您通过桌面环境登录到一个Linux系统，然后启动一个终端仿真器，您会发现将有两个活动会话。

如果您想获得有关活动用户会话的更多详细信息，请使用who命令。

“内存”部分显示有关系统内存使用的信息。标有“内存”和“交换”的行分别显示内存和交换空间的信息。简单地说，交换空间是硬盘的一部分，像内存一样使用。当内存使用率接近满时，内存中不常使用的区域将被写入交换空间，以备以后需要时检索。但是，由于访问磁盘速度很慢，过多依赖交换会损害系统性能。

正如你自然会想到的，“总的”、“免费的”和“用过的”价值有它们通常的含义。“avail mem”值是指在不引起更多交换的情况下，可以分配给进程的内存量。

Linux内核还试图以各种方式减少磁盘访问时间。它在内存中维护一个“磁盘高速缓存”，其中存储了磁盘的常用区域。此外，磁盘写入被存储到“磁盘缓冲区”，内核最终将它们写入磁盘。它们消耗的总内存是“缓冲/缓存”值。这听起来可能是件坏事，但事实并非如此——如果需要，缓存使用的内存将分配给进程。

“任务”部分显示系统上运行的进程的统计信息。“总”值仅仅是过程的总数。例如，在上面的截图中，有27个进程正在运行。为了理解其余的值，我们需要一些关于Linux内核如何处理进程的背景知识。

进程执行输入/输出绑定工作(如读取磁盘)和CPU绑定工作(如执行算术运算)的混合。当一个进程执行输入/输出时，中央处理器是空闲的，因此操作系统在此期间切换到执行其他进程。此外，操作系统允许给定的进程执行很短的时间，然后切换到另一个进程。这就是操作系统看起来好像是“多任务处理”的样子。做所有这些需要我们跟踪一个过程的“状态”。在Linux中，进程可能处于以下状态:

- Runnable(R): 处于这种状态的进程或者在中央处理器上执行，或者存在于运行队列中，准备执行。
- Interruptible Sleep(S): 处于这种状态的进程正在等待事件完成。
- Uninterruptable Sleep(D):在这种情况下，一个进程正在等待输入/输出操作完成。
- Stopped(T):这些进程已被作业控制信号(如按下`Ctrl+Z`)或因为它们正在被跟踪而停止。
- Zombie(Z):内核在内存中维护各种数据结构来跟踪进程。一个进程可能会创建多个子进程，当父进程还在时，它们可能会退出。但是，这些数据结构必须保留，直到父进程获得子进程的状态。这种数据结构仍然存在的终止进程被称为僵尸。

处于“休眠”状态的进程显示为“休眠”，处于“停止”状态的进程显示为“停止”。僵尸数量显示为“僵尸”值。

“CPU使用率”部分显示了花费在各种任务上的CPU时间的百分比。`us`值是CPU在用户空间中执行进程所花费的时间。同样，`sy`值是运行内核进程所花费的时间。

Linux使用一个“nice”值来确定一个进程的优先级。具有高“nice”值的流程比其他流程“nice”，并且优先级较低。类似地，具有较低“nice”的进程获得较高的优先级。正如我们将在后面看到的，默认的“nice”值可以改变。用手动设置的“尼斯”执行进程所花费的时间显示为ni值。

接下来是标识，这是CPU保持空闲的时间。大多数操作系统在空闲时将中央处理器置于省电模式。接下来是`wa`值，即CPU等待输入/输出完成的时间。

中断是给处理器的关于需要立即注意的事件的信号。外设通常使用硬件中断来告诉系统事件，例如键盘上的按键。另一方面，软件中断是由于处理器上执行的特定指令而产生的。无论哪种情况，操作系统都会处理它们，处理硬件和软件中断所花费的时间分别由`hi`和`si`给出。

在虚拟化环境中，一部分CPU资源分配给每个虚拟机。操作系统会检测何时有工作要做，但无法执行，因为其他虚拟机上的CPU正忙着。以这种方式损失的时间量是“窃取”时间，显示为st。

负载平均部分表示一分钟、五分钟和十五分钟内的平均“负载”。“负载”是对系统执行的计算工作量的度量。在Linux上，负载是在任何给定时刻处于研发状态的进程数。“负载平均值”为您提供了一个相对的指标，可以衡量您必须等待多长时间才能完成任务。
让我们考虑几个例子来理解这个概念。在单核系统上，平均负载为0.4意味着系统只能完成40%的工作。平均负载为1意味着系统完全处于容量状态——即使增加一点点额外工作，系统也会过载。平均负载为2.12的系统意味着其过载的工作量比它能处理的多112%。

在多核系统上，您应该首先将负载平均值除以CPU内核数量，以获得类似的度量。

此外，“平均负载”实际上并不是我们大多数人都知道的典型“平均负载”。这是一个“指数移动平均线”，这意味着一小部分以前的负荷平均线被计入当前值。如果你感兴趣，这篇文章涵盖了所有的技术细节。
