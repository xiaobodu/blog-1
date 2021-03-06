记录最近出的几个bug

## connection reset by peer

最近服务器经常性的出现connection reset by peer的错误，开始我们只是以为小概率的网络断开导致的，可是随着压力的增大，每隔2分钟开始出现一次，这就不得不引起我们的重视了。

我们的业务很简单，lvs负责负载均衡（采用的是DR模式），keepalive timeout设置的为2分钟，后面支撑两台推送服务（后面叫做push），客户端首先通过lvs路由到某台push之后，频繁的向其发送推送消息。

客户端使用的是python request（底层基于urllib3），首先我很差异出了这样的错误竟然没有重试，因为写代码的童鞋告诉我会有重试机制的。于是翻了一下request的代码，竟然发现默认的重试是0，一下子碉堡了。

不过，即使改了重试，仍然没有解决reset by peer的问题。通常出现这种情况，很大的原因在于客户端使用的是keep alive长连接保活tcp，但是服务器端关闭了该连接。可是我们的服务器实现了定时ping的保活机制，应该也不会出现的。

然后我将目光投向了lvs，因为它的timeout设置的为2分钟，而reset by peer这个错误也是两分钟一个，所以很有可能就是我们的定时ping机制不起作用，导致lvs直接close掉了连接。

于是查看push自己的代码，陡然发现我们自己设置的定时ping的时间是3分钟，顿时无语了，于是立刻改成1分钟，重启push，世界清静了。

## ifconfig overruns

push换上新的机器之后，（性能妥妥的强悍），我们竟然发现推送的丢包率竟然上升了，一下子碉堡了，觉得这事情真不应该发生的。通常这种情况发生在cpu处理网络中断响应不过来。但是我们可是妥妥的24核cpu，并且开启了irqbalance。

好不，用cat /proc/interrupts之后，发现所有的网卡中断都被cpu0处理了，irqbalance完全没有起作用。google之后发现，有些网卡在PCI-MSI模式下面irqbalance无效，而我们的网卡恰好是PCI-MSI模式的。

没办法，关停irqbalance，手动设置网卡中断的SMP_AFFINITY，一下子世界清静了。

## 总结

可以发现，最近出的几次蛋疼的事情都是在运维层面上面出现的，实际测试也测不出来，碰到这样的问题，只能通过log这些的慢慢摸索排查了。当然也给了我一个教训，任何error级别的log都应该重视，不应该想当然的忽略。