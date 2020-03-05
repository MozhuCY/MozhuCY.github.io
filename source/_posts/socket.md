---
title: CVE-2019-6778
date: 2020-03-03 08:51:01
tags: PWN
---



# 一切皆socket

socket起源于UNIX,可以看出来socket完美的贴合了UNIX一切皆文件的编程思想.我们可以对socket描述符进行ORW的操作,类似普通文件.

在网络编程中,都是由socket实现的,在linux中,socket由上层的libc中的socket和内核中的socket两部分组成,当然为了和硬件交互,最后还需要控制网卡驱动,才是一个完整的体系.

这里我们可以进行一个完整流程的追踪,直接断在tcp_emu上.然后查看调用栈,回溯一下

```
#0  tcp_emu (so=0x7fffe0193ed0, m=0x7fffe0180e20) at /home/mozhucy/qemu-3.0.0/slirp/tcp_subr.c:617
#1  0x0000555555bfc697 in tcp_input (m=0x7fffe0180e20, iphlen=20, inso=0x0, af=2) at /home/mozhucy/qemu-3.0.0/slirp/tcp_input.c:572
#2  0x0000555555bf35c7 in ip_input (m=0x7fffe0180e20) at /home/mozhucy/qemu-3.0.0/slirp/ip_input.c:206
#3  0x0000555555bf6a4a in slirp_input (slirp=0x55555680c500, pkt=0x7fffe02f2700 "RU\n", pkt_len=1334) at /home/mozhucy/qemu-3.0.0/slirp/slirp.c:874
#4  0x0000555555bdf408 in net_slirp_receive (nc=0x55555680c350, buf=0x7fffe02f2700 "RU\n", size=1334) at /home/mozhucy/qemu-3.0.0/net/slirp.c:121
#5  0x0000555555bd5117 in nc_sendv_compat (nc=0x55555680c350, iov=0x7fffeedc3310, iovcnt=1, flags=0) at /home/mozhucy/qemu-3.0.0/net/net.c:701
#6  0x0000555555bd51d9 in qemu_deliver_packet_iov (sender=0x5555578b7d90, flags=0, iov=0x7fffeedc3310, iovcnt=1, opaque=0x55555680c350) at /home/mozhucy/qemu-3.0.0/net/net.c:728
#7  0x0000555555bd7b08 in qemu_net_queue_deliver (queue=0x55555680c290, sender=0x5555578b7d90, flags=0, data=0x7fffe02f2700 "RU\n", size=1334) at /home/mozhucy/qemu-3.0.0/net/queue.c:164
#8  0x0000555555bd7c24 in qemu_net_queue_send (queue=0x55555680c290, sender=0x5555578b7d90, flags=0, data=0x7fffe02f2700 "RU\n", size=1334, sent_cb=0x0) at /home/mozhucy/qemu-3.0.0/net/queue.c:199
#9  0x0000555555bd4f7e in qemu_send_packet_async_with_flags (sender=0x5555578b7d90, flags=0, buf=0x7fffe02f2700 "RU\n", size=1334, sent_cb=0x0) at /home/mozhucy/qemu-3.0.0/net/net.c:655
#10 0x0000555555bd4fb6 in qemu_send_packet_async (sender=0x5555578b7d90, buf=0x7fffe02f2700 "RU\n", size=1334, sent_cb=0x0) at /home/mozhucy/qemu-3.0.0/net/net.c:662
#11 0x0000555555bd4fe3 in qemu_send_packet (nc=0x5555578b7d90, buf=0x7fffe02f2700 "RU\n", size=1334) at /home/mozhucy/qemu-3.0.0/net/net.c:668
#12 0x0000555555aeb758 in rtl8139_transfer_frame (s=0x5555578b2d30, buf=0x7fffe02f2700 "RU\n", size=1334, do_interrupt=1, dot1q_buf=0x0) at /home/mozhucy/qemu-3.0.0/hw/net/rtl8139.c:1804
#13 0x0000555555aecb73 in rtl8139_cplus_transmit_one (s=0x5555578b2d30) at /home/mozhucy/qemu-3.0.0/hw/net/rtl8139.c:2332
#14 0x0000555555aecc31 in rtl8139_cplus_transmit (s=0x5555578b2d30) at /home/mozhucy/qemu-3.0.0/hw/net/rtl8139.c:2359
#15 0x0000555555aed8a2 in rtl8139_io_writeb (opaque=0x5555578b2d30, addr=217 '\331', val=64) at /home/mozhucy/qemu-3.0.0/hw/net/rtl8139.c:2742
#16 0x0000555555aee87f in rtl8139_ioport_write (opaque=0x5555578b2d30, addr=217, val=64, size=1) at /home/mozhucy/qemu-3.0.0/hw/net/rtl8139.c:3279
#17 0x000055555585cbb6 in memory_region_write_accessor (mr=0x5555578b57c0, addr=217, value=0x7fffeedc37e8, size=1, shift=0, mask=255, attrs=...) at /home/mozhucy/qemu-3.0.0/memory.c:527
#18 0x000055555585cdce in access_with_adjusted_size (addr=217, value=0x7fffeedc37e8, size=1, access_size_min=1, access_size_max=4, access_fn=0x55555585cacc <memory_region_write_accessor>, mr=0x5555578b57c0, attrs=...) at /home/mozhucy/qemu-3.0.0/memory.c:594
#19 0x000055555585f9f6 in memory_region_dispatch_write (mr=0x5555578b57c0, addr=217, data=64, size=1, attrs=...) at /home/mozhucy/qemu-3.0.0/memory.c:1473
#20 0x00005555557fba28 in flatview_write_continue (fv=0x7fffe017c000, addr=4273803481, attrs=..., buf=0x7ffff7ff0028 "@\020", len=1, addr1=217, l=1, mr=0x5555578b57c0) at /home/mozhucy/qemu-3.0.0/exec.c:3255
#21 0x00005555557fbb72 in flatview_write (fv=0x7fffe017c000, addr=4273803481, attrs=..., buf=0x7ffff7ff0028 "@\020", len=1) at /home/mozhucy/qemu-3.0.0/exec.c:3294
#22 0x00005555557fbe78 in address_space_write (as=0x5555566e6640 <address_space_memory>, addr=4273803481, attrs=..., buf=0x7ffff7ff0028 "@\020", len=1) at /home/mozhucy/qemu-3.0.0/exec.c:3384
#23 0x00005555557fbec9 in address_space_rw (as=0x5555566e6640 <address_space_memory>, addr=4273803481, attrs=..., buf=0x7ffff7ff0028 "@\020", len=1, is_write=true) at /home/mozhucy/qemu-3.0.0/exec.c:3395
#24 0x000055555587ac06 in kvm_cpu_exec (cpu=0x555556833070) at /home/mozhucy/qemu-3.0.0/accel/kvm/kvm-all.c:1979
#25 0x0000555555841f35 in qemu_kvm_cpu_thread_fn (arg=0x555556833070) at /home/mozhucy/qemu-3.0.0/cpus.c:1215
#26 0x0000555555d668d7 in qemu_thread_start (args=0x555556854b30) at /home/mozhucy/qemu-3.0.0/util/qemu-thread-posix.c:504
#27 0x00007ffff62526ba in start_thread (arg=0x7fffeedc4700) at pthread_create.c:333
#28 0x00007ffff5f8841d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

可以看到还是很多的调用流程,不过大概可以梳理出一个流程,28-24属于kvm/多CPU初始化的过程,23-17属于内存的处理过程,16-12属于网卡虚拟化内部的数据流处理流程,11-0属于qemu对于数据包的底层调用,也就是qemu网络的后端实现

There are two parts to networking within QEMU:

- the virtual network device that is provided to the guest (e.g. a PCI network card).
- the network backend that interacts with the emulated NIC (e.g. puts packets onto the host's network).

slirp_input函数作为入口

```C
static ssize_t net_slirp_receive(NetClientState *nc, const uint8_t *buf, size_t size)
{
    SlirpState *s = DO_UPCAST(SlirpState, nc, nc);

    slirp_input(s->slirp, buf, size);

    return size;
}

```



这里是qemu调用slirp_input的地方,可以看到,传入参数为buf,size,还有Slirp结构体,其中buf为网络包的内容size为网络包的大小

```
void slirp_input(Slirp *slirp, const uint8_t *pkt, int pkt_len)
{
    struct mbuf *m;
    int proto;

    if (pkt_len < ETH_HLEN)
        return;

    proto = ntohs(*(uint16_t *)(pkt + 12));
    switch(proto) {
    case ETH_P_ARP:
        arp_input(slirp, pkt, pkt_len);
        break;
    case ETH_P_IP:
    case ETH_P_IPV6:
        m = m_get(slirp);
        if (!m)
            return;
        /* Note: we add 2 to align the IP header on 4 bytes,
         * and add the margin for the tcpiphdr overhead  */
        if (M_FREEROOM(m) < pkt_len + TCPIPHDR_DELTA + 2) {
            m_inc(m, pkt_len + TCPIPHDR_DELTA + 2);
        }
        m->m_len = pkt_len + TCPIPHDR_DELTA + 2;
        memcpy(m->m_data + TCPIPHDR_DELTA + 2, pkt, pkt_len);

        m->m_data += TCPIPHDR_DELTA + 2 + ETH_HLEN;
        m->m_len -= TCPIPHDR_DELTA + 2 + ETH_HLEN;

        if (proto == ETH_P_IP) {
            ip_input(m);
        } else if (proto == ETH_P_IPV6) {
            ip6_input(m);
        }
        break;

    case ETH_P_NCSI:
        ncsi_input(slirp, pkt, pkt_len);
        break;

    default:
        break;
    }
}
```



到达slirp_input函数后,libslirp开始解析网络包,首先验证packet_len和ETH_HLEN长度的大小,ETH_HLEN为以太帧头部的大小,两个6字节表示mac地址,两字节表示协议

判断后,取出以太帧传输的协议部分.开始进行数据的分发,这里分为四个case,三个处理块,分别为ARP协议IP协议和NCSI协议

在ip处理块中,计算了偏移,去除了ETH_HLEN,并且将待处理的值,放到m中

```C
struct mbuf {
	/* XXX should union some of these! */
	/* header at beginning of each mbuf: */
	struct	mbuf *m_next;		/* Linked list of mbufs */
	struct	mbuf *m_prev;
	struct	mbuf *m_nextpkt;	/* Next packet in queue/record */
	struct	mbuf *m_prevpkt;	/* Flags aren't used in the output queue */
	int	m_flags;		/* Misc flags */

	int	m_size;			/* Size of mbuf, from m_dat or m_ext */
	struct	socket *m_so;

	caddr_t	m_data;			/* Current location of data */
	int	m_len;			/* Amount of data in this mbuf, from m_data */

	Slirp *slirp;
	bool	resolution_requested;
	uint64_t expiration_date;
	char   *m_ext;
	/* start of dynamic buffer area, must be last element */
	char    m_dat[];
};
```

可以看到mbuf是由一个双向链表所维护的,还包括了socket结构体,data指针和长度,最后将这个传入ip_input中

```C
void
ip_input(struct mbuf *m)
{
	Slirp *slirp = m->slirp;//取出slirp结构体
	register struct ip *ip;//初始化ip结构体
	int hlen;

	if (!slirp->in_enabled) {
		goto bad;
	}

	DEBUG_CALL("ip_input");
	DEBUG_ARG("m = %p", m);
	DEBUG_ARG("m_len = %d", m->m_len);

	if (m->m_len < sizeof (struct ip)) { //判断数据长度和ip结构体长度的关系
		goto bad;
	}

	ip = mtod(m, struct ip *);//给数据转换类型为ip

	if (ip->ip_v != IPVERSION) {//检查是不是IPV4
		goto bad;
	}

	hlen = ip->ip_hl << 2; //计算头部ip长度
	if (hlen<sizeof(struct ip ) || hlen>m->m_len) {/* min header length */
	  goto bad;                                  /* or packet too short */
	}

        /* keep ip header intact for ICMP reply
	 * ip->ip_sum = cksum(m, hlen);
	 * if (ip->ip_sum) {
	 */
	if(cksum(m,hlen)) {//计算校验和
	  goto bad;
	}

	/*
	 * Convert fields to host representation.
	 */
	NTOHS(ip->ip_len);
	if (ip->ip_len < hlen) {
		goto bad;
	}
	NTOHS(ip->ip_id);
	NTOHS(ip->ip_off);

	/*
	 * Check that the amount of data in the buffers
	 * is as at least much as the IP header would have us expect.
	 * Trim mbufs if longer than we expect.
	 * Drop packet if shorter than we expect.
	 */
	if (m->m_len < ip->ip_len) {
		goto bad;
	}

	/* Should drop packet if mbuf too long? hmmm... */
	if (m->m_len > ip->ip_len)
	   m_adj(m, ip->ip_len - m->m_len);

	/* check ip_ttl for a correct ICMP reply */
	if (ip->ip_ttl == 0) {
	    icmp_send_error(m, ICMP_TIMXCEED, ICMP_TIMXCEED_INTRANS, 0, "ttl");
	    goto bad;
	}

	/*
	 * If offset or IP_MF are set, must reassemble.
	 * Otherwise, nothing need be done.
	 * (We could look in the reassembly queue to see
	 * if the packet was previously fragmented,
	 * but it's not worth the time; just let them time out.)
	 *
	 * XXX This should fail, don't fragment yet
	 */
	if (ip->ip_off &~ IP_DF) {//IP分片处理
	  register struct ipq *fp;
      struct qlink *l;
		/*
		 * Look for queue of fragments
		 * of this datagram.
		 */
		for (l = slirp->ipq.ip_link.next; l != &slirp->ipq.ip_link;
		     l = l->next) {
            fp = container_of(l, struct ipq, ip_link);
            if (ip->ip_id == fp->ipq_id &&
                    ip->ip_src.s_addr == fp->ipq_src.s_addr &&
                    ip->ip_dst.s_addr == fp->ipq_dst.s_addr &&
                    ip->ip_p == fp->ipq_p)
		    goto found;
        }
        fp = NULL;
	found:

		/*
		 * Adjust ip_len to not reflect header,
		 * set ip_mff if more fragments are expected,
		 * convert offset of this to bytes.
		 */
		ip->ip_len -= hlen;
		if (ip->ip_off & IP_MF)
		  ip->ip_tos |= 1;
		else
		  ip->ip_tos &= ~1;

		ip->ip_off <<= 3;

		/*
		 * If datagram marked as having more fragments
		 * or if this is not the first fragment,
		 * attempt reassembly; if it succeeds, proceed.
		 */
		if (ip->ip_tos & 1 || ip->ip_off) {
			ip = ip_reass(slirp, ip, fp);
                        if (ip == NULL)
				return;
			m = dtom(slirp, ip);
		} else
			if (fp)
		   	   ip_freef(slirp, fp);

	} else
		ip->ip_len -= hlen;

	/*
	 * Switch out to protocol's input routine.
	 */
	switch (ip->ip_p) { //选择协议模拟
	 case IPPROTO_TCP:
		tcp_input(m, hlen, (struct socket *)NULL, AF_INET);
		break;
	 case IPPROTO_UDP:
		udp_input(m, hlen);
		break;
	 case IPPROTO_ICMP:
		icmp_input(m, hlen);
		break;
	 default:
		m_free(m);
	}
	return;
bad:
	m_free(m);
}
```

可以看到这里是对数据包的检测以及分段的模拟等,随后进入到协议模拟

