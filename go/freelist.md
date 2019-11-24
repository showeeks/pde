# Freelist

并发编程的工具甚至可以让非并发的想法更容易表达。这里有一个从RPC包中抽象出来的例子。客户端goroutine循环接收来自某个来源的数据，可能是网络。为了避免分配和释放缓冲区，它保留一个自由列表freelist，并使用缓冲通道来表示它。如果通道为空，则会分配一个新的缓冲区。一旦消息缓冲区准备好，它就被发送到serverChan上的服务器。
