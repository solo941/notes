# Linux常见命令总结

## 查看日志

小的日志文件：

- `cat service.log`
- `tail -f service.log`
- `vim serivice.log`

如果日志文件比较大：

需要grep搜索关键词。如果想搜索关键词的上下文，使用下面的命令，可以查看关键词所在行行数：

cat -n service.log | grep 13888888888

假设所在行为29506，如果想要查看前面的20行，可以使用命令：

- `sed -n "29496,29516p" service.log`：从29496行开始检索，到29516行结束
- `cat -n service.log | tail -n +29496 | head -n 20`:从29496行开始检索，往前推20条

在日志文件中，需要统计出现关键词的个数，使用下面的命令：

cat -n service.log | grep 13888888888 | wc -l

也可以将搜索结果以文件格式输出：

cat service.log | grep 13888888888 &gt; /home/log.txt

## 查看进程和端口

查看Netconf进程：

ps -ef |grep Netconf

查看10020端口，使用以下命令：

netstat -anp |grep 10020

lsof -i:10020

## 查看系统状态

### 查看进程状态

Linux进程可以分为三个状态：

- 阻塞进程
- 可运行的进程
- 正在运行的进程

使用top命令可以查看load average，分别代表：1分钟、5分钟、15分钟内运行进程队列中的平均进程数量。即非阻塞进程在一定时间内的均值。

### 查看内存使用状况

free打印出的内存信息主要分为两种，mem是物理内存，swap是用磁盘虚拟内存。

![pic](https://github.com/solo941/notes/blob/master/Linux/pics/697611-20181210234331537-1515989707.png)

物理内存是计算机的实际内存大小，由RAM芯片组成。虚拟内存则是虚拟出来的、使用磁盘代替内存。虚拟内存的出现，让机器内存不够的情况得到部分解决。当程序运行起来由操作系统做具体虚拟内存到物理内存的替换和加载（相应的页与段的虚拟内存管理）。

当用户提交程序，然后产生进程在机器上运行。机器会判断当前物理内存是否还有空闲允许进程调入内存运行，如果有则直接调入内存进行；如果没有，则会根据优先级选择一个进程挂起，把该进程交换到swap中等待，然后把新的进程调入到内存中运行。根据这种换入和换出，实现了内存的循环利用，让用户感觉不到内存的限制。从这也可以看出swap扮演了一个非常重要的角色，就是暂存被换出的进程。内存与swap之间是按照内存页为单位来交换数据的，一般Linux中页的大小设置为4Kb。而内存与磁盘则是按照块来交换数据的。

linux的内存管理机制的思想包括内存利用率最大化，内核会把剩余的内存申请为cached，而cached不属于free范畴。如果free的内存不够，内核会把部分cached的内存回收，回收的内存再分配给应用程序。所以对于linux系统，可用于分配的内存不只是free的内存，还包括cached的内存（其实还包括buffers）。

## 链接

```
# ln [-sf] source_filename dist_filename
-s ：默认是实体链接，加 -s 为符号链接
-f ：如果目标文件存在时，先删除目标文件
```

### 硬链接

![pic](https://github.com/solo941/notes/blob/master/Linux/pics/1e46fd03-0cda-4d60-9b1c-0c256edaf6b2.png)

在Linux为文件系统中，保存在磁盘分区中的文件不管是什么类型都给它分配一个编号，称为索引节点号（inode）；硬链接在目录下创建一个条目，记录着文件名与 inode 编号，这个 inode 就是源文件的 inode。删除任意一个条目，文件还是存在，只要引用数量不为 0。

### 符号连接

符号链接文件保存着源文件所在的绝对路径，在读取时会定位到源文件上，可以理解为 Windows 的快捷方式。当源文件被删除了，链接文件就打不开了。

## awk

awk是一个强大的文本分析工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。

awk 每次处理一行，处理的最小单位是字段，每个字段的命名方式为：$n，n 为字段号，从 1 开始，$0 表示一整行。

应用：取出最近五个登录用户的用户名和 IP

last -n 5 | awk '{print $1 "\t" $3}'

可以根据字段的某些条件进行匹配，例如匹配字段小于某个值的那一行数据。

awk '条件类型 1 {动作 1} 条件类型 2 {动作 2} ...' filename

示例：/etc/passwd 文件第三个字段为 UID，对 UID 小于 10 的数据进行处理。

cat /etc/passwd | awk 'BEGIN {FS=":"} $3 < 10 {print $1 "\t " $3}'

awk 变量：

| 变量名称 | 代表意义                     |
| -------- | ---------------------------- |
| NF       | 每一行拥有的字段总数         |
| NR       | 目前所处理的是第几行数据     |
| FS       | 目前的分隔字符，默认是空格键 |

示例：显示正在处理的行号以及每一行有多少字段

last -n 5 | awk '{print $1 "\t lines: " NR "\t columns: " NF}'

AWK 将每条线视为由多个字段组成，每个字段由“间隔符”分隔。 默认情况下，这是一个或多个空格字符，字段分隔符由 awk 内部变量 FS 设置，因此如果您设置 FS= ": "则它将根据 ‘:’ 的位置划分一行。

举个例子

```unix
ls -l
总用量 140
-rw-r--r-- 1 root root 55121 1月   3 17:03 combined_log_format.log
-rw-r--r-- 1 root root 80644 1月   3 17:03 combined_log_format_w_resp_time.log
-rw-r--r-- 1 root root    71 1月   3 17:55 sampler.log
```

AWK 支持（‘for’ 和 ‘while’）循环和分支（使用 'if '）。 所以，如果你想修剪一个文件并且只在每个第 3 行操作，你可以这样做：

```
ls -l | awk '{for (i=1;i<3;i++) {getline}; print NR,$0}'
3 -rw-r--r-- 1 root root 80644 1月   3 17:03 combined_log_format_w_resp_time.log
4 -rw-r--r-- 1 root root    71 1月   3 17:55 sampler.log
```

awk本质上就是一个for循环，它每次对输入文件的一行进行处理，然后转而执行下一行，直到整个文件的每一行都被执行完毕。整个过程是自动的，你无需做什么。但是，getline命令却可以让你去控制循环。getline得到的并不是当前行，而是当前行的下一行。以上面的例子来分析，awk首先读取到了第一行，就是1，然后执行两次getline，就得到了第三行，因为getline之后，awk会改变对应的NF，NR，FNR和$0等内部变量，所以此时的$0的值就不再是1，而是3了，然后将它打印出来。同理打印第四行。

同样，我们可以利用getline只打印出奇数行。

```
seq 10 | awk '{getline tmp; print tmp; print $0}'
```

另外getline也可以从另外一个文件中读取内容。下面例子实现将两个文件的每一行都打印在一行上。

```
awk '{printf "%s ", $0; getline < "b.txt"; print $0}' a.txt
```

下面再给出几个例子

以/为分隔符，匹配倒数第二行的s或者没有s后面是bin的整行

```
awk -F "/" '$(NF-1)~/(s|)bin/' /etc/passwd    
```

以一个或多个/为行的分割符，打印第二行的第二列，列的分隔符为默认的空格,并打印行号

```
awk 'BEGIN{RS="[/]+"} NR==2{print NR,$2}' test  
```

使用awk命令按域名统计 返回码大于等于400的百分比，假如优酷总共有4行，大于等于400的返回码有两行，那占比就为50%

```
 http://youku.com 200
 http://youku.com 302
 http://youku.com 403
 http://youku.com 502
 http://baidu.com 302
 http://baidu.com 404
```

```
awk '{
        count[$1]++; 
        if($2>400)above400[$1]++
    }
    END{
        for(i in count){
            print i, count[i], above400[i]/count[i]
        }
    }' < xxx.txt
```

END只在最后一行遍历完执行。

tomcat并发数

```
netstat -an|grep 10050|awk '{count[$6]++} END{for (i in count) print(i,count[i])}'
```

## sed

Sed是Strem Editor(流编辑器)缩写，是操作、过滤和转换文本内容的强大工具。常用功能有增删改查，过滤，取行。

概括流程：Sed软件从文件或管道中读取一行，处理一行，输出一行；再读取一行，再处理一行，再输出一行

a 追加文本到指定行后

 i 插入文本到指定行前

```
sed '2a 106,dandan,CSO\n107,bingbing,CCO' person.txt
```

我们对sampler.log我文件进行操作

```
sed -e 's/input/output/' sampler.log 
boot
book
booze
machine
boots
bungie
bark
aardvark
broken$tuff
robots
```

如果你希望在搜索命令中使用的某个字符是特殊符号，例如 ‘/’，该怎么办？（例如在文件名中）或 ‘*’ 等？ 然后你必须像 grep（和awk）那样转义符号。

```
sed -e 's/\/bin/\/usr\/local\/bin/' my_script > new_script
```

每行以你的文件中的数字开头，并用括号括起该数字：

```
sed -e 's/[0-9]*/(&)/' 
```

**其它 SED 命令**

```
sed -e '/pattern/ command' sampler.log
```

s’= search＆replace，或 ‘p’= print，或 ‘d’=delete，或 ‘i’=insert，或 ‘a’=append

删除文件的前10行以外的所有行

```
sed -e '11,$ d' sampler.log
```

想打印从 ‘boot’ 到 ‘machine’ 的所有行

```
sed -n -e '/boot$/,/mach/p' sampler.log
boot
book
booze
machine
```



## 进程状态

![pic](https://github.com/solo941/notes/blob/master/Linux/pics/2bab4127-3e7d-48cc-914e-436be859fb05.png)

| 状态 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| R    | running or runnable (on run queue) 正在执行或者可执行，此时进程位于执行队列中。 |
| D    | uninterruptible sleep (usually I/O) 不可中断阻塞，通常为 IO 阻塞。 |
| S    | interruptible sleep (waiting for an event to complete)   可中断阻塞，此时进程正在等待某个事件完成。 |
| Z    | zombie (terminated but not reaped by its parent) 僵死，进程已经终止但是尚未被其父进程获取信息。 |
| T    | stopped (either by a job control signal or because it is being traced)   结束，进程既可以被作业控制信号结束，也可能是正在被追踪。 |

### SIGCHLD 信号与wait()，waitpid()方法

当一个子进程改变了它的状态时（停止运行，继续运行或者退出），有两件事会发生在父进程中：

- 得到 SIGCHLD 信号；
- waitpid() 或者 wait() 调用会返回。

在子进程退出时，它的进程描述符不会立即释放，这是为了让父进程得到子进程信息，父进程通过 wait() 和 waitpid() 来获得一个已经退出的子进程的信息。

#### pid_t wait(int *status)

父进程调用 wait() 会一直阻塞，直到收到一个子进程退出的 SIGCHLD 信号，之后 wait() 函数会销毁子进程并返回。参数 status 用来保存被收集的子进程退出时的一些状态，如果对这个子进程是如何死掉的毫不在意，只想把这个子进程消灭掉，可以设置这个参数为 NULL。

#### pid_t waitpid(pid_t pid, int *status, int options)

waitpid()方法作用和 wait() 完全相同，但是多了两个可由用户控制的参数 pid 和 options。pid 参数指示一个子进程的 ID，表示只关心这个子进程退出的 SIGCHLD 信号。如果 pid=-1 时，那么和 wait() 作用相同，都是关心所有子进程退出的 SIGCHLD 信号。

options 参数主要有 WNOHANG 和 WUNTRACED 两个选项，WNOHANG 可以使 waitpid() 调用变成非阻塞的，也就是说它会立即返回，父进程可以继续执行其它任务。

## 孤儿进程与僵尸进程

### 孤儿进程

一个父进程退出，而它的一个或多个子进程还在运行，那么这些子进程将成为孤儿进程。

 孤儿进程将被 init 进程（进程号为 1）所收养，并由 init 进程对它们完成状态收集工作。

 由于孤儿进程会被 init 进程收养，所以孤儿进程不会对系统造成危害。

### 僵尸进程

一个子进程的进程描述符在子进程退出时不会释放，只有当父进程通过 wait() 或 waitpid() 获取了子进程信息后才会释放。如果子进程退出，而父进程并没有调用 wait() 或 waitpid()，那么子进程的进程描述符仍然保存在系统中，这种进程称之为僵尸进程。

系统所能使用的进程号是有限的，如果产生大量僵尸进程，将因为没有可用的进程号而导致系统不能产生新的进程。要消灭系统中大量的僵尸进程，只需要将其父进程杀死，此时僵尸进程就会变成孤儿进程。

## 参考资料

[**【Linux】Swap与Memory**](https://blog.csdn.net/qq_36838191/article/details/82768303)

[Linux.](https://github.com/CyC2018/CS-Notes/blob/master/notes/Linux.md)

[工作中常用到的Linux命令](https://mp.weixin.qq.com/s/2jZ2jwjv0-yeqAw27FAcFQ)

[**性能工具之linux三剑客awk、grep、sed详解**](https://blog.csdn.net/zuozewei/article/details/85709621)

[**awk getline命令解析**](https://blog.csdn.net/xibeichengf/article/details/51367311)

[awk  sed   grep 详解](https://www.cnblogs.com/dazhidacheng/p/8030481.html)