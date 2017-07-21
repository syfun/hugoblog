+++
date = "2015-04-05T19:19:35+08:00"
description = ""
title = "Postgresql子进程启动"
thumbnail = ""
tags = ["其他"]
categories = ["其他"]

+++


PG是多进程的运行模式，除了Postmaster主进程外，还有以下子进程：

1. SysLogger process
2. Startup process
3. Bgwriter process
4. Checkpointer process
5. Walwriter process
6. Walreceiver process
7. Pgstat process
8. Pgarchive process
9. Auto Vacuum process

那么子进程到底是在何时启动的？

<!--more-->

通过代码，我们很容易就能发现子进程启动的位置是在PostmasterMain中。

第一个启动的进程是SysLogger：

```c
// postmaster.c:1146
SysLoggerPID = SysLogger_Start();
```

SysLogger_Start中前面的一些处理不做分析了，直接到

```c
// syslogger.c:601
switch ((sysloggerPid = fork_process()))
```

fork_process对linux的fork()函数做了一层简单的封装，返回值是一样的。
如果返回-1表示创建子进程出错；返回0表示创建成功，并且当前是在子进程中；
返回其他值就表明创建成功，并且当前在父进程中，返回值是子进程的PID。


fork_process返回值是0时，进入子进程，接着走到SysLoggerMain。在这个函数中，
做了一些初始化后，进入for(;;)循环。

第二个启动的进程是Startup process:

```c
// postmaster.c:1217
StartupPID = StartupDataBase();

// postmaster.c:503
#define StartupDataBase()  StartChildProcess(StartupProcess)
```

在StartChildProcess中同样是调用fork_process创建子进程，返回0进入子进程，调用
AuxiliaryProcessMain。

```c
// postmaster.c:5103
AuxiliaryProcessMain(ac, av);

// bootstrap.c:414
case StartupProcess:
    /* don't set signals, startup process has its own agenda */
    StartupProcessMain();
    proc_exit(1);

// startup.c:224
StartupXLOG();

/*
* Exit normally. Exit code 0 tells postmaster that we completed recovery
* successfully.
*/
proc_exit(0);
```

starup process实际的入口是StartupProcessMain。但是我们看到，在这个函数中，做完
预写日志恢复后，就正常退出了，在官方给的注释中说明，正常退出告诉postmaster，我们已经
成功恢复。那么这边退出到底是为了什么？

我们先不考虑这个，如果接着跟踪代码，进入到PostmasterMain的ServerLoop中。在for(;;)循环中，
判断其他进程的PID是否为0，如果为0就创建该子进程。

那么其他进程是在这个时候创建的么？

答案是否。如果调试代码到这个地方，会发现，这些进程PID不为0，说明在这之前就已经创建完成了。我们不禁会问，
到底是什么时候创建的，在流程里根本没发现啊。嘿嘿，这里就要说到start process的退出了。

子进程退出会像父进程也就是postmaster进程发送一个SIGCHLD信号。而在PostmasterMain的开始处，就给SIGCHLD注册
了一个信号处理函数reaper。

```c
// postmaster.c:587
pqsignal(SIGCHLD, reaper);	/* handle child termination */

// postmaster.c:2608
if (CheckpointerPID == 0)
	CheckpointerPID = StartCheckpointer();
if (BgWriterPID == 0)
	BgWriterPID = StartBackgroundWriter();
if (WalWriterPID == 0)
	WalWriterPID = StartWalWriter();

/*
* Likewise, start other special children as needed.  In a restart
* situation, some of them may be alive already.
*/
if (!IsBinaryUpgrade && AutoVacuumingActive() && AutoVacPID == 0)
    AutoVacPID = StartAutoVacLauncher();
if (XLogArchivingActive() && PgArchPID == 0)
	PgArchPID = pgarch_start();
if (PgStatPID == 0)
	PgStatPID = pgstat_start();
```

在reaper函数里，我们也发现了这里也有启动进程的方法，一调试代码发现其他子进程就是在这里启动的。

总结一下就是，先启动SysLogger，然后是Startup，Startup退出后触发主进程启动其他进程。
