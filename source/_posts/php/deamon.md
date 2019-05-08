---
title: PHP 守护进程
date: 2019-02-18
categories: PHP
tag: deamon
---


## PHP 守护进程 
守护进程是一种运行在后台的特殊进程，因为它不属于任何一个终端，所以不会收到任何终端发来的任何信号。它与前台进程显著的区别是：
* 它没有控制终端，不能直接和用户交互，在后台运行；
* 它不受用户登录和注销的影响，只受开机或关机的影响，可以长期运行；
* 通常我们编写的程序，都需要在 后台不终止的长期运行 ，此时就可以使用守护进程。当然，我们可以在代码中调用系统函数，或者直接在启动命令后追加&操作符，如下：
``` bash
$ nohup php server.php start &
```
通常&与 nohup 结合使用，忽略 SIGHUP 信号来实现一个守护进程。该方式对业务代码侵入最小，方便且成本低，常用于临时执行任务脚本的场景。

### 守护进程要点

1. 进程守护化 使用 pcntl_fork()创建子进程，终止父进程,使得程序在 shell 终端里造成一个已经运行完毕的假象

``` bash
protected static function daemonize()
{

    $pid = pcntl_fork();
    if (-1 === $pid) {
         exit("process fork fail\n");
    } elseif ($pid > 0) { //父进程直接退出
        exit(0);
    }
    // 将当前进程提升为会话leader
    if (-1 === posix_setsid()) {
        exit("process setsid fail\n");
    }
    //改变工作目录
    chdir('/');
 	//重设文件创建的掩码
    umask(0);
    // 再次fork以避免SVR4这种系统终端再一次获取到进程控制
    $pid = pcntl_fork();
    if (-1 === $pid) {
        exit("process fork fail\n");
    } elseif ($pid > 0) {
        exit(0);
    }
}

```

2. 在子进程中创建新的会话
会话是一个或多个进程组的集合，一个会话有对应的控制终端。
setsid 函数用于创建一个新的会话，并担任该会话组的组长。调用 setsid 的三个作用：让进程摆脱原会话的控制、让进程摆脱原进程组的控制和让进程摆脱原控制终端的控制。
在调用 fork 函数时，子进程全盘拷贝父进程的会话期 (session，是一个或多个进程组的集合)、进程组、控制终端等，虽然父进程退出了，但原先的会话期、进程组、控制终端等并没有改变，因此，那还不是真正意义上使两者独立开来。setsid 函数能够使进程完全独立出来，从而脱离所有其他进程的控制。

3. 改变工作目录
使用 fork 创建的子进程也继承了父进程的当前工作目录。由于在进程运行过程中，当前目录所在的文件系统不能卸载，因此，把当前工作目录换成其他的路径，如 “/” 或 “/tmp” 等。改变工作目录的常见函数是 chdir。

4. 重设文件创建掩码
文件创建掩码是指屏蔽掉文件创建时的对应位。由于使用 fork 函数新建的子进程继承了父进程的文件创建掩码，这就给该子进程使用文件带来了诸多的麻烦。因此，把文件创建掩码设置为 0，可以大大增强该守护进程的灵活性。设置文件创建掩码的函数是 umask，通常的使用方法为 umask (0)。

5. 重定向标准输入输出
用 fork 新建的子进程会从父进程那里继承一些已经打开了的文件。这些被打开的文件可能永远不会被守护进程读或写，但它们一样消耗系统资源，可能导致所在的文件系统无法卸载。
``` bash
protected static function resetStdFd()
{
    global $STDIN, $STDERR, $STDOUT;
    //重定向标准输出和错误输出
    @fclose(STDIN);
    @fclose(STDOUT);
    @fclose(STDERR);
    $STDIN  = fopen('/dev/null', 'r');
    $STDOUT = fopen(static::$stdoutFile, 'a');
    $STDERR = fopen(static::$stdoutFile, 'a');
}
```
如果你关闭了标准输出，标准错误输出文件描述符，那么你打开的前三个文件描述符将成为新的标准输入、输出、错误的描述符。
使用$STDIN, $STDOUT纯粹是障眼法而已, 必须指定为全局变量，否则文件描述符将在函数执行完毕之后被释放。

6. 信号处理
在 Linux 系统中，可使用kill -l命令查看这 62 个信号值,使用信号来实现进程间通信并控制进程的行为，注册信号处理器如下：

``` bash
function installSignal()
{
    pcntl_signal(SIGINT,  'signalHandler', false);
    pcntl_signal(SIGTERM, 'signalHandler', false);

    pcntl_signal(SIGUSR1, 'signalHandler', false);
    pcntl_signal(SIGQUIT, 'signalHandler', false);

    // 忽略信号
    pcntl_signal(SIGUSR2, SIG_IGN, false);
    pcntl_signal(SIGHUP,  SIG_IGN, false);
}

function signalHandler($signal)
{
    switch($signal) {
        case SIGINT:
        case SIGTERM:
            static::stop();
            break;
        case SIGQUIT:
        case SIGUSR1:
            static::reload();
            break;
        default: break;
    }
}
```
其中，SIGINT 和 SIGTERM 信号会触发stop操作，即终止所有进程；SIGQUIT 和 SIGUSR1 信号会触发reload操作，即重新加载所有 Worker 进程；此处忽略了 SIGUSR2 和 SIGHUP 信号，但是并未忽略 SIGKILL 信号，即所有进程都可以被强制kill掉。