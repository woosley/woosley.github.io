---
layout: post
section-type: post
title: mount namespace in golang
---

namespace 是 docker 实现资源隔离的基础，而 mount namespace 是 linux 实现的第一
个 namespace。

以下代码简单的实现了一些常见的 namespace 功能

<pre><code data-trim class="go">package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

var registered = make(map[string]func())
var name = "namespace_init"
var self = "/proc/self/exe"
var shell string = "/usr/bin/bash"

func init() {
	//register a function in memory
	fmt.Printf("register %s \n", name)
	if _, exists := registered[name]; exists {
		panic(fmt.Sprintf("name already registered: %p", name))
	}
	registered[name] = namespace_init

	initializer, exists := registered[os.Args[0]]
	if exists {
		initializer()
		os.Exit(0)
	}
}

func namespace_init() {

	fmt.Printf("setup hostname as container1\n")
	if err := syscall.Sethostname([]byte("container1")); err != nil {
		fmt.Println(err)
	}

	defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV

	if err := syscall.Mount("", "/", "", uintptr(defaultMountFlags|syscall.MS_PRIVATE|syscall.MS_REC), ""); err != nil {
		fmt.Println(err)
	}

	fmt.Printf("mouting proc\n")
	if err := syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), ""); err != nil {
		fmt.Println(err)
	}

	container_command()
}

var command string = "/usr/bin/bash"

func container_command() {

	fmt.Printf("starting container command %s\n", command)
	cmd, _ := exec.LookPath(shell)
	err := syscall.Exec(cmd, []string{}, os.Environ())
	if err != nil {
		fmt.Println("error", err)
	}
}

func setup_self_command(args ...string) *exec.Cmd {
	return &exec.Cmd{
		Path: self,
		Args: args,
		SysProcAttr: &syscall.SysProcAttr{
			Pdeathsig: syscall.SIGTERM,
		},
	}
}

func main() {
	cmd := setup_self_command(name)
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWNS |
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWIPC |
			syscall.CLONE_NEWNET}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	fmt.Printf("starting current process %d\n", os.Getpid())

	if err := cmd.Run(); err != nil {
		fmt.Println("error", err)
		os.Exit(1)
	}
	fmt.Printf("command ended\n")
}
</code></pre>

以下是命令的运行结果

 
<pre><code data-trim class="go">[root@localhost go]# go run namespace-mount.go
register namespace_init
starting current process 3860
register namespace_init
setup hostname as container1
mouting proc
starting container command /usr/bin/bash
[root@container1 go]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 07:59 pts/2    00:00:00 [bash]
root        86     1  0 07:59 pts/2    00:00:00 ps -ef
[root@container1 go]# exit
exit
command ended
[root@localhost go]#
</code></pre>

可以看到这个程序实现了基本的`proc`隔离。

假如我们注释掉

<pre><code data-trim class="go">if err := syscall.Mount("", "/", "", uintptr(defaultMountFlags|syscall.MS_PRIVATE|syscall.MS_REC), ""); err != nil {
    ...
}
</code></pre>

在 centos7 上，命令退出后会出现问题

<pre><code data-trim class="go">[root@localhost go]# go run namespace-mount.go
register namespace_init
starting current process 4094
register namespace_init
setup hostname as container1
mouting proc
starting container command /usr/bin/bash
[root@container1 go]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 08:03 pts/2    00:00:00 [bash]
root        86     1  0 08:03 pts/2    00:00:00 ps -ef
[root@container1 go]# exit
exit
command ended
[root@localhost go]# ps -ef
Error, do this: mount -t proc proc /proc
[root@localhost go]# mount
mount: failed to read mtab: No such file or directory
</code></pre>

`ls /proc`会发现所有的`pid`目录都没有了

<pre><code data-trim class="shell">
[root@localhost go]# ls /proc/
ls: cannot read symbolic link /proc/self: No such file or directory
acpi       cgroups   crypto     driver       fs          irq       key-users
loadavg  misc     net           scsi      stat           sysvipc      uptime
zoneinfo asound     cmdline   devices    execdomains  interrupts  kallsyms  kmsg
locks    modules  pagetypeinfo  self      swaps          timer_list   version
buddyinfo  consoles  diskstats  fb           iomem       kcore     kpagecount
mdstat   mounts   partitions    slabinfo  sys            timer_stats vmallocinfo
</code></pre>

这里涉及到 mount namespace 的事件传播机制，linux 中一共定义了四种

- `MS_SHARED`: 同一个 peer group的成员共享 mount 事件
- `MS_PRIVATE`: 私有，不发送，也不接收任何 mount 事件
- `MS_SLAVE`: 介于私有和共享之间。mount group 有一个 master，master 的事件传递到 slave 而 slave 不能传递到 master
- `MS_UNBINDABLE`: 除了不发送和接受 mount 事件之外，这个类型的还不能被 `mount --bind` 

而`systemd`将默认的 mount namespace 的事件传播机制定义成了 `MS_SHARED`。这样`container`中的`mount`事件被系统接收，从而覆盖了系统本身自己的`proc`目录，而在进程退出之后，仅有的一个 pid 1 目录也被删除，在 proc 下面就没有了任何 pid 目录。

这里可以做一个简单的动手，主要参考了 `https://lwn.net/Articles/689856/`


