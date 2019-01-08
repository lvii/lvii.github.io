---
layout: post
title:  "ACPI 电源管理导致的 CPU loadavg 负载虚高"
category: system
tags: [loadavg]
---

## 起因

厂里有位兄台反映：有台 DELL R620 机器空置，应用还没上线 load 就彪的很高。怀疑机器坏了，让换机器

## 科普

先科普下 load 高的原因 `man uptime` 给的解释 ：

    load averages is the average number of processes that are either in a runnable or uninterruptable state.
    A process in a runnable state is either using the CPU or waiting to use the CPU.
    A process in uninterruptable state is waiting for some I/O access, eg waiting for disk.

- CPU 真的忙。top 中按 `%CPU` 排序会看到比较 **彪悍** 的进程
- CPU 假装忙。top 中查看每个 CPU 的 `idle` 都很闲。是有进程的 I/O 请求没被 **满足** 太饥渴了。。。

等待 I/O 而 **睡死过去** 的进程，系统中对应的 **进程状态** 一般是 D

### 进程状态

`man ps` 手册对 `D` **进程状态** 的解释：`D uninterruptible sleep (usually IO)` 再引用一下 **维基百科** 的解释：

<http://en.wikipedia.org/wiki/Sleep_(system_call)#Uninterruptible_sleep>

> Uninterruptible sleep
>
> An `uninterruptible sleep` state is a sleep state that won't handle a signal right away.
> It will wake only as a result of a waited-upon resource becoming available or after a
> time-out occurs during that wait (if specified when put to sleep).
> It is mostly used by device drivers **waiting for disk or network IO** (input/output).
> When the process is sleeping uninterruptibly, signals accumulated during the sleep will be noticed
> when the process returns from the system call or trap.

`kill -9` 见了 `D` 状态的进程都耸，硬的不行，只能来软的，得满足它对 I/O 的请求才行。

## 经过

top 显示 CPU `%idle` 很高，24 个 core 好多都闲的蛋疼。但是 loadavg `5.24, 4.56, 4.38` 却不低

    top - 10:58:45 up 38 days, 11:02,  2 users,  load average: 5.24, 4.56, 4.38
    Tasks: 482 total,   1 running, 480 sleeping,   0 stopped,   1 zombie
    Cpu0  : 91.4%us,  8.6%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu1  :  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu2  :  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu3  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu4  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu5  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu6  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu7  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu8  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu9  :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu10 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu11 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu12 :  0.0%us,  0.3%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu13 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu14 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu15 :  0.0%us,  0.3%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu16 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu17 :  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu18 :  0.0%us,  0.3%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu19 :  2.0%us,  3.7%sy,  0.0%ni, 94.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu20 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu21 :  1.0%us,  0.3%sy,  0.0%ni, 98.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu22 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Cpu23 :  0.0%us,  0.0%sy,  0.0%ni,100.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st

top 找到 **power_saving** 进程状态是 `D` ：

    # top -b -n 1 | awk '{if (NR <=7) print; else if ($8 == "D") {print; count++} }'

    top - 11:00:41 up 38 days, 11:04,  2 users,  load average: 5.10, 4.70, 4.45
    Tasks: 481 total,   4 running, 476 sleeping,   0 stopped,   1 zombie
    Cpu(s):  0.4%us,  0.2%sy,  0.0%ni, 99.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
    Mem:  132112476k total, 105675476k used, 26437000k free,  1128692k buffers
    Swap: 12582904k total,        0k used, 12582904k free, 98710784k cached

      PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
      179 root      20   0     0    0    0 D  0.0  0.0   0:00.00 kacpi_notify
    28626 root      -2   0     0    0    0 D  0.0  0.0   0:06.40 power_saving/0
    28627 root      -2   0     0    0    0 D  0.0  0.0   0:03.57 power_saving/1
    28628 root      -2   0     0    0    0 D  0.0  0.0   0:06.48 power_saving/2

`/var/log/message` 和 `power_saving` 有关系的日志：

    Fri Feb 14 22:35:30 2014 - INFO: task kacpi_notify:179 blocked for more than 120 seconds.
    Fri Feb 14 22:35:30 2014 - "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
    Fri Feb 14 22:35:30 2014 - kacpi_notify  D 0000000000000000     0   179      2 0x00000000
    Fri Feb 14 22:35:30 2014 -  ffff881018d53b30 0000000000000046 0000000000000000 ffff881018d53af4
    Fri Feb 14 22:35:30 2014 -  0000000000000000 ffff88103fc28400 ffff880028276680 0000000000000200
    Fri Feb 14 22:35:30 2014 -  ffff881018d3a638 ffff881018d53fd8 000000000000fb88 ffff881018d3a638
    Fri Feb 14 22:35:30 2014 - Call Trace:
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814f0685>] schedule_timeout+0x215/0x2e0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff81062c81>] ? __enqueue_rt_entity+0x2c1/0x300
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8106310b>] ? enqueue_rt_entity+0x6b/0x80
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814f0303>] wait_for_common+0x123/0x180
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8105fa40>] ? default_wake_function+0x0/0x20
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814f041d>] wait_for_completion+0x1d/0x20
    Fri Feb 14 22:35:30 2014 -  [<ffffffff81090a0b>] kthread_stop+0x4b/0xd0
    Fri Feb 14 22:35:30 2014 -  [<ffffffffa018342a>] acpi_pad_idle_cpus+0xbc/0xd6 [acpi_pad]
    Fri Feb 14 22:35:30 2014 -  [<ffffffffa018370c>] acpi_pad_handle_notify+0x96/0x196 [acpi_pad]
    Fri Feb 14 22:35:30 2014 -  [<ffffffff810096f0>] ? __switch_to+0xd0/0x320
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8105ab19>] ? find_busiest_queue+0x69/0x150
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814ef810>] ? thread_return+0x4e/0x76e
    Fri Feb 14 22:35:30 2014 -  [<ffffffff812c7d20>] ? acpi_os_execute_deferred+0x0/0x36
    Fri Feb 14 22:35:30 2014 -  [<ffffffffa018382a>] acpi_pad_notify+0x1e/0x5b [acpi_pad]
    Fri Feb 14 22:35:30 2014 -  [<ffffffff812d7ccf>] acpi_ev_notify_dispatch+0x64/0x71
    Fri Feb 14 22:35:30 2014 -  [<ffffffff812c7d49>] acpi_os_execute_deferred+0x29/0x36
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8108b4b0>] worker_thread+0x170/0x2a0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff81090d20>] ? autoremove_wake_function+0x0/0x40
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8108b340>] ? worker_thread+0x0/0x2a0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff810909b6>] kthread+0x96/0xa0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8100c0ca>] child_rip+0xa/0x20
    Fri Feb 14 22:35:30 2014 -  [<ffffffff81090920>] ? kthread+0x0/0xa0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8100c0c0>] ? child_rip+0x0/0x20
    Fri Feb 14 22:35:30 2014 - INFO: task power_saving/0:28626 blocked for more than 120 seconds.
    Fri Feb 14 22:35:30 2014 - "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
    Fri Feb 14 22:35:30 2014 - power_saving/ D 0000000000000000     0 28626      2 0x00000000
    Fri Feb 14 22:35:30 2014 -  ffff881ae5923dc0 0000000000000046 0000000000000000 ffff881ae5923fd8
    Fri Feb 14 22:35:30 2014 -  000000000000fb88 ffff882018411b00 ffff881019874080 ffff882018411540
    Fri Feb 14 22:35:30 2014 -  ffff882018411af8 ffff881ae5923fd8 000000000000fb88 ffff882018411af8
    Fri Feb 14 22:35:30 2014 - Call Trace:
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814f0e6e>] __mutex_lock_slowpath+0x13e/0x180
    Fri Feb 14 22:35:30 2014 -  [<ffffffff814f0d0b>] mutex_lock+0x2b/0x50
    Fri Feb 14 22:35:30 2014 -  [<ffffffffa01830ef>] power_saving_thread+0xd4/0x353 [acpi_pad]
    Fri Feb 14 22:35:30 2014 -  [<ffffffffa018301b>] ? power_saving_thread+0x0/0x353 [acpi_pad]
    Fri Feb 14 22:35:30 2014 -  [<ffffffff810909b6>] kthread+0x96/0xa0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8100c0ca>] child_rip+0xa/0x20
    Fri Feb 14 22:35:30 2014 -  [<ffffffff81090920>] ? kthread+0x0/0xa0
    Fri Feb 14 22:35:30 2014 -  [<ffffffff8100c0c0>] ? child_rip+0x0/0x20

哇。下面一行提到了 `acpi_pad` 内核模块，是 **电源管理** 这货搞的鬼：

    Fri Feb 14 22:35:30 2014 -  [<ffffffffa01830ef>] power_saving_thread+0xd4/0x353 [acpi_pad]

下面是前人对 dell 机器遭遇 `acpi_pad` 电源管理 [「bug」][0] 的吐操贴：

[「"power_saving" in centos 6.2 deployed on the PowerEdge R620」][1]

[1]: http://en.community.dell.com/support-forums/servers/f/1466/t/19456558.aspx
[0]: https://bugzilla.kernel.org/show_bug.cgi?id=42981

## 结果

- `modprobe -r acpi_pad` 无法卸载模块
- 也不知道怎么满足 `power_saving` 进程的 IO 请求，唤醒进程

那就禁用 **acpi 电源管理** 吧：

在 grub 的 kernel 配置后面，添加 `acpi_pad.disable=1` 重启机器之后，开机就不会自动加载 **acpi_pad** 模块

后面用户再没发现 load 虚高问题复现了，大概就这样收尾了。


## 参考

<http://en.community.dell.com/support-forums/servers/f/1466/t/19456558.aspx>

<http://www.linuxquestions.org/questions/linux-server-73/high-load-average-low-cpu-usage-on-centos-5-4-64-bit-794709/>

<http://www.linuxquestions.org/questions/linux-newbie-8/high-cpu-load-but-low-cpu-usage-high-idle-cpu-902632/>


<br/>

本文标题 | [{{ page.title }}]({{ page.url }})
-------- |:--------
原始链接 | <{{ site.url }}{{ page.url }}>
