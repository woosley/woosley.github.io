---
layout: post
section-type: post
title: Http Keepalive 超时
---

线上有一个服务，会启动大量的python worker，然后通过python的requests库，连接到后端的
arangodb上。前一阵子发现一个问题，python worker启动超过一定的时间之后，所有的
worker都会卡住，使用strace会发现是卡在本机到arangodb的一个socket连接上，worker从这个socket读取数据，但永远没有返回。

`requests`是开启了keep-alive的，这样的话client和arangodb之间是长连接。刚开始的时
候对这里的超时感觉到不理解。

由于client和arangodb之间有防火墙。通过防火墙的session追踪功能看了一下。

<pre><code data-trim class="bash">user@fw-01> show security flow session source-port 33175 source-prefix 10.16.1.13 protocol tcp
Flow Sessions on FPC1 PIC0:

Session ID: 20392406, Policy name: my_policy/92, State: Active, Timeout: 1714, Valid
  In: 10.16.1.13/33175 --> 10.16.6.104/8529;tcp, If: reth1.910, Pkts: 4, Bytes: 440
  Out: 10.16.6.104/8529 --> 10.16.1.13/33175;tcp, If: reth1.915, Pkts: 3, Bytes: 399
Total sessions: 1
</code></pre>

这里可以清楚的看到防火墙在这里会维护session的一个超时机制，在junos里面，这个超
时时间默认为30分钟。防火墙由于长时间没有收到数据，于是将这个session删除掉了。

但是这与我印象中的keep-alive不太相符。keep-alive应该在连接idle的时候发送相应的
心跳来阻止连接超时。而且linux服务器上的TCP keep-alive似乎是默认打开的。

<pre><code data-trim class="bash">sysctl -a|grep keepalive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 75
</code></pre>

这些参数分别表示

- `tcp_keealive_time`:TCP连接发送最后一个数据包后多长时间发送keepalive探测包。
- `tcp_keepalive_intvl`：keepalive探测包的发送间隔
- `tcp_keepalive_probes`：发送probe的数量，当超过这个数量之后，认为TCP连接已经断开。

显然如果keepalive打开的话，在client和arangodb之间应该有持续的probe探测，但是通过TCPDUMP发现并没有。

之后仔细研究了一下keepalive的相关[文档](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html)。发现和我原来想的有点不一样.

<pre>
Remember that keepalive support, even if configured in the kernel, is not the default behavior in Linux. Programs must request keepalive control for their sockets using the setsockopt interface. There are relatively few programs implementing keepalive, but you can easily add keepalive support for most of them following the instructions explained later in this document. 
</pre>

也就是说即使linux配置了相关的keepalive选项。一个应用程序需要显示的调用`setsockopt`才能真正的开启keepalive支持。而这个选项在`requests`里面并没有默认开启。http的keepalive支持不过是在header里面加上一个参数`Connection': 'keep-alive'`用来告诉服务器不要主动关闭连接.

requests本身也没有提供更改相关TCP level的参数的方法。如果需要启用keepalive，可以使用`request_toolbelt`提供的[TCPKeepAliveAdapter](http://toolbelt.readthedocs.org/en/latest/adapters.html#tcpkeepaliveadapter)。




