---
layout: post
section-type: post
title: 使用 Carbon Query 查询 Graphite Carbon Cache 内存数据
tags: ['graphite', 'carbon']
---

最近有同事反映搭建的 [Graphite](https://github.com/graphite-project) Cluster 上
数据上报有明显的延时，当前时间的数据大约要两个小时之后才能出现。

我的第一反应是是不是 Relay 和 Carbon Cache 之间的 Consistent Hash 出错了，导致保存
在内存里面的数据不能够及时的从 Cache 里面读出来，而必须等到写入磁盘之后，从而导
致了延时。于是就一直在找一个方法，希望能从 Carbon Cache 的内存里面查询到保存的
metrics 数据。

这件事首先要确定的是对于一个 Metrics，它通过 Consistent Hash 映射到哪个 Carbon
Cache instance 上面。[Carbonate](https://github.com/jssjr/carbonate) 项目提供了
对应的工具，安装后脚本为 `carbon-lookup`，用法如下。

{% highlight bash %}
# carbon-lookup servers.ash.ash0012.Network.eth0.UnicastPktsOut
# ash0013:2204:3
{% endhighlight %}

Carbonate 并没有相关的工具用来查询内存里保存的数据，于是我翻看了一下 Graphite
的源码，照着写了一个脚本，姑且称之为 `carbon-query`

{% highlight python %}

import sys
import socket
import pickle
import struct
def recv_exactly(conn, num_bytes):
    buf = ''
    while len(buf) < num_bytes:
        data = conn.recv( num_bytes - len(buf) )
        if not data:
            raise Exception("Connection lost")
        buf += data
    return buf


metric = sys.argv[1]
hosts = ['somehosts'] # CHANGE TO YOUR CARBON HOSTS
ports = range(7201, 7211) # CHANGE TO YOUR CARBON PORTS
for h in hosts:
    for p in ports:
        conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        conn.connect((h, p))
        request = dict(type='cache-query', metric=metric)
        metric = request['metric']
        serialized_request = pickle.dumps(request, protocol=-1)
        len_prefix = struct.pack("!L", len(serialized_request))
        request_packet = len_prefix + serialized_request
        conn.sendall(request_packet)
        len_prefix = recv_exactly(conn, 4)
        body_size = struct.unpack("!L", len_prefix)[0]
        body = recv_exactly(conn, body_size)
        print h, p, pickle.loads(body)
        
{% endhighlight %}

脚本运行示例如下

{% highlight bash %}
[root@mypc~]# python2.7 carbon-query.py servers.ash.ash0012.Network.eth0.UnicastPktsOut
ash0003 7201 {'datapoints': []}
ash0003 7202 {'datapoints': []}
ash0003 7203 {'datapoints': []}
ash0003 7204 {'datapoints': []}
ash0003 7205 {'datapoints': []}
ash0003 7206 {'datapoints': []}
ash0003 7207 {'datapoints': []}
ash0003 7208 {'datapoints': []}
ash0003 7209 {'datapoints': []}
ash0003 7210 {'datapoints': []}
ash0013 7201 {'datapoints': []}
ash0013 7202 {'datapoints': []}
ash0013 7203 {'datapoints': [(1446986222.0, 754735607.0), (1446986281.0, 754746449.0), (1446986342.0, 754761006.0), (1446986402.0, 754773906.0), (1446986462.0, 754786530.0), (1446986521.0, 754797609.0), (1446986582.0, 754809617.0), (1446986642.0, 754825521.0), (1446986702.0, 754840010.0), (1446986762.0, 754852165.0), (1446986821.0, 754861513.0), (1446986882.0, 754873532.0), (1446986942.0, 754884357.0), (1446987002.0, 754894571.0), (1446987062.0, 754908550.0), (1446987121.0, 754918568.0), (1446987182.0, 754932662.0), (1446987242.0, 754946677.0), (1446987302.0, 754958301.0), (1446987362.0, 754972934.0), (1446987421.0, 754982688.0), (1446987482.0, 754995836.0), (1446987542.0, 755011046.0), (1446987602.0, 755023627.0), (1446987662.0, 755038518.0), (1446987721.0, 755049424.0), (1446987782.0, 755062970.0), (1446987842.0, 755077243.0), (1446987902.0, 755091580.0), (1446987962.0, 755105138.0), (1446988022.0, 755115642.0), (1446988081.0, 755127361.0), (1446988142.0, 755140769.0), (1446988202.0, 755152935.0), (1446988262.0, 755164088.0), (1446988322.0, 755175527.0), (1446988381.0, 755188091.0), (1446988442.0, 755200334.0), (1446988502.0, 755213654.0), (1446988562.0, 755225641.0), (1446988621.0, 755235989.0), (1446988681.0, 755248447.0), (1446988742.0, 755259571.0), (1446988802.0, 755272117.0), (1446988862.0, 755283038.0), (1446988921.0, 755295489.0), (1446988982.0, 755309818.0), (1446989042.0, 755322986.0), (1446989102.0, 755335887.0), (1446989162.0, 755347366.0), (1446989221.0, 755360052.0), (1446989282.0, 755371923.0), (1446989342.0, 755386517.0)]}
ash0013 7204 {'datapoints': []}
ash0013 7205 {'datapoints': []}
ash0013 7206 {'datapoints': []}
ash0013 7207 {'datapoints': []}
ash0013 7208 {'datapoints': []}
ash0013 7209 {'datapoints': []}
ash0013 7210 {'datapoints': []}
{% endhighlight %}

结果表明对应 metrics 正确的发送到了相应的 Carbon Cache，cluster 的配置没有问题。

之后我再对同一个 metrics 进行了多次查询，发现问题出现在 carbon relay 上面，由于
某些原因，relay 上堆积了大量的数据，没有能够及时的发送到 Cache。最后通过添加
Relay的数目，解决了整个问题。
