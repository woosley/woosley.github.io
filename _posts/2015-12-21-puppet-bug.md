---
layout: post
section-type: post
title: puppet bug 一则
tags: ['puppet']
---

不知道有没有人还在用 2.7 版本的 puppet，由于我们有些业务还在使用 Centos 5 系列 ，于是还在使用这个相当老的版本，各种 performance 不够，痛定思痛，决定将其升级到 puppet 3。将 master 升级之后发现有个坑爹的 bug。

首先是发现 Rabbitmq 模块添加用户的功能失效了。对应 puppet 代码为

<pre><code data-trim class="puppet">
rabbitmq_user { "guest":
    ensure => absent,
    provider => "rabbitmqctl",
    require => Rabbitmq_user["root"],
}
</code></pre>

这里使用到了自定义的 Provider 和 Type，可以在官方模块中[这里](https://github.com/puppetlabs/puppetlabs-rabbitmq/blob/master/lib/puppet/type/rabbitmq_user.rb) 和 [这里](https://github.com/puppetlabs/puppetlabs-rabbitmq/blob/master/lib/puppet/provider/rabbitmq_user/rabbitmqctl.rb) 看到。

当时第一反应是是不是模块内部出问题了，用 irb 调试了一番也没发现什么。

之后运行`puppet agent -t`的时候发现输出好像有点不对，一般来说正常的输出是:

<pre><code data-trim class="bash">
[root@host~]# puppet agent -t
info: Retrieving plugin
info: Loading facts in /etc/puppet/modules/imlb/lib/facter/imlb_version.rb
info: Loading facts in /etc/puppet/modules/mysql/lib/facter/mysql_version.rb
info: Loading facts in /etc/puppet/modules/puppet/lib/facter/ethernet_mtus.rb
info: Loading facts in /etc/puppet/modules/stdlib/lib/facter/root_home.rb
info: Loading facts in /etc/puppet/modules/stdlib/lib/facter/pe_version.rb
info: Loading facts in /etc/puppet/modules/stdlib/lib/facter/facter_dot_d.rb
info: Loading facts in /etc/puppet/modules/stdlib/lib/facter/puppet_vardir.rb
info: Loading facts in /etc/puppet/modules/lsg/lib/facter/lsg_version.rb
info: Loading facts in /var/lib/puppet/lib/facter/datacenter.rb
info: Loading facts in /var/lib/puppet/lib/facter/mysql_version.rb
info: Loading facts in /var/lib/puppet/lib/facter/root_home.rb
info: Loading facts in /var/lib/puppet/lib/facter/pe_version.rb
</code></pre>

而现在的输出是
<pre><code data-trim class="bash">
[root@ash6020 ~]# puppet agent -t
info: Retrieving plugin
err: /File[/var/lib/puppet/lib]: Failed to generate additional resources using 'eval_generate: Error 400 on SERVER: this master is not a CA
</code></pre>

以前看到这个错误基本就忽视了，因为我们的 puppet master 不是一个 CA，ssl 验证过程是通过 Apache 代理的。现在仔细一看，接下来的 loading facters 全部没有被执行，一看 agent 的 `/var/lib/puppet/lib/puppet/`，里面全是空的

puppet 的运行原理是 master 负责将 catalog 编译发送给 client， 而client需要在本机生成 facter，以及进行各种操作。这里添加 Rabbitmq 的用户，那么需要将自定义的 provider，type 对应的类库下载到本机才能执行。这个目录为空，基本断定 puppet server 没有将需要的文件发送给 client。

最后发现是[这个](https://projects.puppetlabs.com/issues/17864) bug，在 puppet不是 CA 的情况下，agent 会请求 cert 的回收列表，puppet 3 返回 400 的错误，导致agent 端无法跑接下来的逻辑。在 puppet 2.7 中，master 返回的是 404，这个倒是不会中断后续工作。

鉴于在 centos5 上不能安装 puppet 3+，我们最后自己给 agent 打了一个 patch 了事。
