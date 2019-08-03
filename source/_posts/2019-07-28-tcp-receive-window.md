title: TCP 接收窗口
date: 2019-07-28 20:28:00
tags:
- TCP
- Linux Kernel
categories:
- Develop
---

最近有空阅读了一些 TCP 窗口相关的内核代码，知识点相对比较琐碎，做一些简单的笔记，本篇主要关于 TCP 接收窗口。

本篇主要基于 Linux v5.2.2 内核，如有不准确之处，会日后修订。

# 前置知识

在 TCP 传输中提到的窗口一般有三个，接收窗口、发送窗口和拥塞窗口：

- **接收窗口**：一般是接收端向发送端通告自己还可以接收和缓冲多少数据， 可在用户态通过设置 `SO_RCVBUF` 一定程度上调整初始的接受窗口大小。
- **拥塞窗口**：可以理解为拥塞算法定义的逻辑窗口，是有发送端的流控算法，一般是 MSS（最大分段大小）的数倍。在一般意义的拥塞算法中，窗口在慢开始阶段增长，当窗口值到达 ssthresh（慢启动阈值）或数据丢失时，会进行相应调整。总之，这是一个由拥塞算法所控制的窗口。
- **发送窗口**：算是一个综合性的窗口，主要用于调整可发送数据的大小，和发送缓冲区剩余大小，拥塞窗口及发送中的数据包量（packet in flight）相关。

# 接收窗口的初始化

初始化的入口在 `tcp_select_initial_window`，会由 `tcp_connect_init` 在连接初始时（未发送 syn，syn-ack 包时）被调用。

简单贴一下 `tcp_select_initial_window` 的声明：

```c++
void tcp_select_initial_window(const struct sock *sk, int __space, __u32 mss,
                   __u32 *rcv_wnd, __u32 *window_clamp,
                   int wscale_ok, __u8 *rcv_wscale,
                   __u32 init_rcv_wnd);
```

这里主要解释一下各参数的含义及在上层调用中是如何得出的：

1. **sk** [传入值]：sock 结构体，代表 socket。
2. **space** [传入值]：接收缓冲区的大小，在上层通过 `tcp_full_space` 再调用 `tcp_win_from_space` 来计算出。不直接使用缓冲区大小的原因是，内核将 socket 的缓冲区会划分为网络缓冲区和应用缓冲区，网络缓冲区及通常所提到的数据，而应用缓冲区为 skb 头部一类的数据。具体划分比率主要依赖 sysctl 中设置的 `tcp_select_initial_window`，`tcp_adv_win_scale` 大于 0 时，网络缓冲区的计算逻辑为 `space - (space >> tcp_adv_win_scale)`；否则计算逻辑为 `space>>(-tcp_adv_win_scale)`。更具体的可以参照内核实现或 [Ipsysctl tutorial - TCP Variables](https://www.frozentux.net/ipsysctl-tutorial/chunkyhtml/tcpvariables.html)。
3. **mss** [传入值]：即 MSS（最大分段大小），在上次通过 `tcp_sock` 中的 `advmss` 计算而来。这里的 `advmss` 即从路由表设置中拉取的通告的 MSS（advertised MSS），如果当前连接开启了时间戳选项，则 mss 传入值再减去时间戳选项这部分的大小。具体开启时间戳与否的判断是通过 `tp->rx_opt.ts_recent_stamp` 是否有值来判断的，这个变量代表开启 timestamp 选项后收到的最近一个数据包的时间戳，当开启了 timestamp 选项后才会有值。
4. **rcv_wnd** [传出值]：接收窗口的值，即上层 `tcp_sock` 中的 `rcv_wnd`。
5. **window_clamp** [传入／传出值]：窗口的最大值，即上层 `tcp_sock` 中的 `window_clamp`。
6. **rcv_window_scaling** [传入值]：是否支持接收窗口缩放选项，窗口缩放选项的逻辑具体由 RFC 1323 定义。简单的说通过系数的缩放（具体公式为 `window size << window scale`），窗口的大小最大可以达到 65535 Bytes，更详细的逻辑可以查看 [相关 WIKI](https://en.wikipedia.org/wiki/TCP_window_scale_option)。
7. **rcv_wscale** [传出值]：接收窗口缩放选项。
8. **init_rcv_wnd** [传入值]：初始的接收窗口，优先会选择 bpf 上通告的初始窗口，具体可以查看这个 [patch](https://www.spinics.net/lists/netdev/msg443179.html)，如果 bpf 不作配置，从路由表中获取初始接收窗口，即 `dst_metric(dst, RTAX_INITRWND)`。

整个函数体如下所示：

```c
unsigned int space = (__space < 0 ? 0 : __space);

if (*window_clamp == 0)
	(*window_clamp) = (U16_MAX << TCP_MAX_WSCALE);
space = min(*window_clamp, space);

// 如果空间足够，将 space 调整为 mss 的整数倍
if (space > mss)
	space = rounddown(space, mss);

// 针对初始窗口最大值的修正逻辑
if (sock_net(sk)->ipv4.sysctl_tcp_workaround_signed_windows)
	(*rcv_wnd) = min(space, MAX_TCP_WINDOW);
else
	(*rcv_wnd) = min_t(u32, space, U16_MAX);

// 如果有传入外部的 init_rcv_wnd，与当前 rcv_wnd 取最小值
if (init_rcv_wnd)
	*rcv_wnd = min(*rcv_wnd, init_rcv_wnd * mss);

*rcv_wscale = 0;
if (wscale_ok) {
    // 根据当前最大可能的窗口值来选择窗口缩放系数
	space = max_t(u32, space, sock_net(sk)->ipv4.sysctl_tcp_rmem[2]);
	space = max_t(u32, space, sysctl_rmem_max);
	space = min_t(u32, space, *window_clamp);
    // 减去 15 是因为窗口默认用 16 位表示
	*rcv_wscale = clamp_t(int, ilog2(space) - 15,
				  0, TCP_MAX_WSCALE);
}
(*window_clamp) = min_t(__u32, U16_MAX << (*rcv_wscale), *window_clamp);
```

`sysctl_tcp_workaround_signed_windows` 变量所在的行数是对 TCP 初始窗口最大值的一个修正逻辑。有些客户端将窗口值视为 signed 类型，如果使用 unsinged 的最大值作为最大值，则对端可能会溢出产生 bug。那么这个选项的意思是，如果开启的话，在对端通告支持窗口缩放选项之前，都将窗口裁剪为 16 位 signed 数值的最大值，保证合法，即为了不出 bug 做一个 workaround；如果不开启的话，最大值就为 `U16_MAX`。

针对初始窗口，通过上层提供的默认接收窗口值，在`init_rcv_wnd` 及最大值等的约束下使其尽可能的最大化。

针对窗口缩放选项，通过 `space`、`ipv4.sysctl_tcp_rmem[2]` 和 `sysctl_rmem_max` 中的最大值，在 `window_clamp` 及 `TCP_MASX_WSCALE` 的约束下获得最大的窗口缩放系数。在得出窗口缩放选项后，通过更新后的值来调整原 `window_clamp`。

# 接收窗口的调整

接收窗口的设定主要和两个函数相关 `tcp_select_window` 和 `tcp_grow_window`。

## tcp_select_window

这个函数的主要调用者为 `__tcp_transmit_skb`，即整个数据包发送之前调用一下来选择合适的接收窗口。

函数只接收一个 `sock` 类型，从参数上不作赘述。

整个函数体如下所示：

```c
struct tcp_sock *tp = tcp_sk(sk);
u32 old_win = tp->rcv_wnd;
u32 cur_win = tcp_receive_window(tp);
u32 new_win = __tcp_select_window(sk);

// 实际不会撤回已分配出去的窗口
if (new_win < cur_win) {
	if (new_win == 0)
		NET_INC_STATS(sock_net(sk),
			      LINUX_MIB_TCPWANTZEROWINDOWADV);
	new_win = ALIGN(cur_win, 1 << tp->rx_opt.rcv_wscale);
}
// 更新 rcv_wnd（接收窗口）和 rcv_wup（上次发生窗口更新时，下一个要接收的包序号）
tp->rcv_wnd = new_win;
tp->rcv_wup = tp->rcv_nxt;

// 针对之前说的窗口溢出的情况做处理
if (!tp->rx_opt.rcv_wscale &&
	sock_net(sk)->ipv4.sysctl_tcp_workaround_signed_windows)
	new_win = min(new_win, MAX_TCP_WINDOW);
else
	new_win = min(new_win, (65535U << tp->rx_opt.rcv_wscale));

// 根据窗口缩放系数进行调整
new_win >>= tp->rx_opt.rcv_wscale;

// 如果通告零窗口，则不启用 fast path 处理
if (new_win == 0) {
	tp->pred_flags = 0;
	if (old_win)
		NET_INC_STATS(sock_net(sk),
			      LINUX_MIB_TCPTOZEROWINDOWADV);
} else if (old_win == 0) {
	NET_INC_STATS(sock_net(sk), LINUX_MIB_TCPFROMZEROWINDOWADV);
}

return new_win;
```

### __tcp_select_window

新窗口的产生主要调用了 `__tcp_select_window`，以下主要来讨论这个函数。

注释中提到了窗口的增长（或者说移动）主要基于以下的约束：

1. 窗口一旦分配出去就不会被撤销回来（RFC 793）。
2. 每个包的所占用的内存空间都是有限制的。

整个的函数逻辑判断较多，贴一下经过注释的代码：

```c
struct inet_connection_sock *icsk = inet_csk(sk);
struct tcp_sock *tp = tcp_sk(sk);
// 最早使用了 mss 最大值，现在使用估计后的对端的 mss
int mss = icsk->icsk_ack.rcv_mss;
int free_space = tcp_space(sk);
int allowed_space = tcp_full_space(sk);
int full_space = min_t(int, tp->window_clamp, allowed_space);
int window;

if (unlikely(mss > full_space)) {
    mss = full_space;
    if (mss <= 0)
        return 0;
}

if (free_space < (full_space >> 1)) {
    // 关闭 quick ack 功能
    icsk->icsk_ack.quick = 0;

    if (tcp_under_memory_pressure(sk))
        tp->rcv_ssthresh = min(tp->rcv_ssthresh,
                       4U * tp->advmss);

    // 根据接收窗口系数调整空闲缓冲区大小
    free_space = round_down(free_space, 1 << tp->rx_opt.rcv_wscale);

    // 窗口过于小，尝试调整为零窗口
    if (free_space < (allowed_space >> 4) || free_space < mss)
        return 0;
}

if (free_space > tp->rcv_ssthresh)
    free_space = tp->rcv_ssthresh;

if (tp->rx_opt.rcv_wscale) {
    window = free_space;

    // 根据窗口缩放系数对齐 window 的大小，这个是向上对齐，
    // 可以避免在窗口大小介于 0 和 1 << tp->rx_opt.rcv_wscale 下零窗的产生
    window = ALIGN(window, (1 << tp->rx_opt.rcv_wscale));
} else {
    // 大部分情况下，window 落在 (空闲缓冲区大小 - MSS，空闲缓冲区] 内，不参与
    // 之后的计算，可以减少运算量
    window = tp->rcv_wnd;

    // 如果不在 (空闲缓冲区大小 - MSS，空闲缓冲区] 的范围内，向下对齐
    if (window <= free_space - mss || window > free_space)
        window = rounddown(free_space, mss);
    // 如果 mss 和整个接收缓冲区空间大小相等或窗口值小于空闲接收缓冲区大小 - 整个接收缓冲区大小的一半
    else if (mss == full_space &&
         free_space > window + (full_space >> 1))
        window = free_space;
}

return window;
```

### 针对 SWS 的处理

接收段的 SWS（Silly window syndrome）主要接收端和发送端速率不匹配情况下，产生大小的小包导致网络传输效率地下的问题。例如考虑一种极端情况，接受端缓冲区满了，用户从中消费 1 Byte 的数据，接受端向发送段通告一个 1 Byte 的窗口，随后发送段再发送 1 Byte 的数据，接着等接收端消费后再发送 1 Byte 的数据，如此往复。这样持续下去，窗口变动频繁，发送数据量小，效率很低。

在 `__tcp_select_window` 的注释中就讨论了 RFC 1122 中提到的避免 SWS 的算法，即保持 `RCV.NEXT + RCV.WIN` 不变（即只允许发这段数据）直到 `RCV.BUFF - RCV.USER - RCV.WIN >= min(1/2 RCV.BUFF, MSS)`。其中 `RCV.NXT` 表示下一个接收的序列号，`RCV.BUFF` 表示整个接收缓冲区的大小，`RCV.USER` 来表示被接收和 ACK，但未被用户取走的数据，`RCV.WINDOW` 就是我们当前说的接收窗口值了。简单说就是知直道可以发送大于 MSS 的数据以前，窗口的右界不做任何变化。

内核中并没有直接采用推荐算法的直接实现，因为窗口的调整打破了 Header Predition 的前提前提条件。Header Predition 是 Linux 内核中用于加速处理 TCP 包的一种机制，通过 `pred_flags` 的值将 TCP 的包处理区分为 fast path 和 slow path，其中 `pred_flags` 的计算中间接使用了接收窗口值，且在使用时假设其不变。大概实现可以参考 [Header Predition](https://www.pdl.cmu.edu/mailinglists/ips/mail/msg00133.html)，这里不赘述。但严格的说，如果保持窗口始终不变的话，仅靠接受端算法无法解决 SWS 的问题的，但如果在发送端做一些相应的调整是可行的。

BSD 做了如下的折中算法：如果空闲空间小于 1/4 的最大可用空间，同时剩余空间小于 1/2 的 MSS，直接通告零窗口。这种折中的做法将原来算法中线性的调整修改为了一个分段的调整，可以减少窗口的频繁变动。Linux 实现中使用了相似的做法，即 `__tcp_select_window` 在窗口过小的情况下，会提前一些向上通告零窗口。`tcp_select_window` 收到零窗口后不会撤回已通告的窗口，会在窗口消耗完毕后，向对端通告零窗口，同时关闭 Header Predition。

## tcp_grow_window

`rcv_ssthresh` 的调整依赖于函数 `tcp_grow_window`，`rcv_ssthresh` 可以理解为在慢开始更严格意义上的 `window_clamp`。

从设计上 `rcv_ssthresh` 主要用于两件事：

1. 给 header predition 机制在缓冲区中留足空间。
2. 减少因为接收窗口的误预测导致接受队列空间不足，最后使包收不到接收队列里的情况。

整个函数体如下所示：

```c
struct tcp_sock *tp = tcp_sk(sk);
int room;

// room 即 rcv_ssthresh 可增长的空间，其中限制了预留给应用的部分
room = min_t(int, tp->window_clamp, tcp_space(sk)) - tp->rcv_ssthresh;

// 确定是否可以增长或者处于 memory pressure 的状态下，即给其他机制留足缓冲区
if (room > 0 && !tcp_under_memory_pressure(sk)) {
	int incr;

    // 是否接收接收缓冲区不足，不足则立即增大 rcv_ssthresh
    // skb 可以给网络缓冲区的总长度小于 skb 数据部分长度，增长设为两倍 MSS，否则通过 __tcp_grow_window 计算
	if (tcp_win_from_space(sk, skb->truesize) <= skb->len)
		incr = 2 * tp->advmss;
	else
		incr = __tcp_grow_window(sk, skb);

	if (incr) {
		incr = max_t(int, incr, 2 * skb->len);
		tp->rcv_ssthresh += min(room, incr);
        // 如果 rcv_ssthresh 增长则启用 quick ack
		inet_csk(sk)->icsk_ack.quick |= 1;
	}
}
```

`__tcp_grow_window` 是针对接收缓冲区不足情况的更详细校验，其函数体部分如下：

```c
struct tcp_sock *tp = tcp_sk(sk);
int truesize = tcp_win_from_space(sk, skb->truesize) >> 1;
int window = tcp_win_from_space(sk, sock_net(sk)->ipv4.sysctl_tcp_rmem[2]) >> 1;

// 这部分可以用不完全准确的用如下公式来表示：
// 如果 ceil(log2(truesize / skb->len)) < ceil(log2(window / rcv_ssthresh)) 则 rcv_ssthresh += 2 * rcv_mss
// 更简单说，skb 内数据、实际分配网络缓冲区空间比率与缓冲区、rcv_ssthresh 比率不匹配则增大 rcv_ssthresh
while (tp->rcv_ssthresh <= window) {
	if (truesize <= skb->len)
		return 2 * inet_csk(sk)->icsk_ack.rcv_mss;

	truesize >>= 1;
	window >>= 1;
}
return 0;
```

--------------

参考资料：
1. https://www.frozentux.net/ipsysctl-tutorial/chunkyhtml/tcpvariables.html
2. https://www.spinics.net/lists/netdev/msg443179.html
3. https://tools.ietf.org/html/rfc1122
4. https://en.wikipedia.org/wiki/Silly_window_syndrome
5. https://www.pdl.cmu.edu/mailinglists/ips/mail/msg00133.html
6. https://www.freebsd.org/cgi/man.cgi?query=mbuf&sektion=9&manpath=FreeBSD+6.0-stable
7. https://patchwork.ozlabs.org/patch/794600/
8. https://www.freesoft.org/CIE/RFC/1323/17.htm