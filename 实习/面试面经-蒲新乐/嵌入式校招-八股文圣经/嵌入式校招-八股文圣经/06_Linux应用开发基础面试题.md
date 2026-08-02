# Linux 应用开发基础面试题（Q1~Q15）

> 本文档由公众号：**嵌入式校招菌** 原创整理，欢迎关注获取更多嵌入式校招信息
>
> 📢 **嵌入式校招excel** 全年更新中，实习/秋招/春招都包含，纯人工维护，已支持24/25/26届同学毕业，减少招聘信息差，你没听过的公司只要有嵌入式岗位我都会收集，一次付费永久有效，更有嵌入式微信/飞书交流群；用过的同学都说好！
>
> 需要的可以关注公众号：**嵌入式校招菌**，对话框回复：**加群**

> 面向嵌入式 Linux 应用开发岗位，覆盖 Shell 命令、文件 I/O、进程/线程、IPC、网络编程等核心知识。


---

## ★ Linux应用开发核心概念图解（先理解原理，再刷面试题）

### ◆ 进程 vs 线程 vs 协程

- **进程**像一栋独立的房子——有自己的地址空间(墙壁)、自己的资源(家具)。房子之间互不影响(进程隔离)，但搬家(创建/销毁)很贵。
- **线程**像房子里的房间——共享客厅厨房(地址空间/文件描述符)，各自有一张床(栈/寄存器)。同一房子的人交流方便(共享内存)，但容易争抢洗手间(竞争条件)。
- **协程**像一个人在多个任务间"手动切换"——做饭等水开的时候去拖地，水开了回来继续做饭。一个线程内协作式调度，没有抢占。

```c
进程内存布局:
  ┌───────────────────┐ 高地址
  │     内核空间       │ 3G~4G (32位系统)
  ├───────────────────┤
  │     栈(Stack) ↓   │ 局部变量、函数调用
  │        ...        │
  │     堆(Heap) ↑    │ malloc/free
  ├───────────────────┤
  │     BSS段         │ 未初始化的全局/静态变量(自动清零)
  │     数据段(Data)  │ 已初始化的全局/静态变量
  │     代码段(Text)  │ 程序指令(只读)
  └───────────────────┘ 低地址

多线程共享:
  ┌──────────────────────────────┐
  │  进程地址空间(共享):          │
  │  代码段 / 数据段 / 堆 / 文件 │
  │  ┌─────┐ ┌─────┐ ┌─────┐   │
  │  │线程1│ │线程2│ │线程3│   │  ← 每个线程独立的:
  │  │ 栈  │ │ 栈  │ │ 栈  │   │    栈、寄存器、errno、TLS
  │  │ PC  │ │ PC  │ │ PC  │   │
  │  └─────┘ └─────┘ └─────┘   │
  └──────────────────────────────┘
```

| 对比项 | 进程 | 线程 | 协程 |
|--------|------|------|------|
| 地址空间 | 独立 | 共享 | 共享 |
| 创建开销 | 大(fork复制页表) | 中(~8KB栈) | 小(~2KB) |
| 切换开销 | 大(TLB刷新) | 中(保存寄存器) | 极小(用户态切换) |
| 通信方式 | 管道/信号/共享内存 | 直接读写共享变量 | 直接读写 |
| 安全性 | 高(隔离) | 低(需加锁) | 低(单线程无竞争) |
| 调度 | 内核调度 | 内核调度 | 用户态协作式 |

### ◆ 文件描述符与I/O模型


```bash
5种I/O模型(面试必考):

1. 阻塞I/O (Blocking)
   app ──read()──→ kernel ──等数据──→ 复制到用户空间 ──→ 返回
   └──────────── 阻塞等待 ─────────────────────────────┘

2. 非阻塞I/O (Non-blocking)
   app ──read()──→ kernel: 没数据! → 返回EAGAIN
   app ──read()──→ kernel: 没数据! → 返回EAGAIN  (轮询)
   app ──read()──→ kernel: 有了! → 复制 → 返回数据

3. I/O多路复用 (select/poll/epoll)  ★最常考★
   app ──epoll_wait()──→ kernel: 监控N个fd
                         ← fd3有数据了!
   app ──read(fd3)──→ 读取

4. 信号驱动I/O (SIGIO)
   注册信号 → kernel有数据时发SIGIO → 回调中read()

5. 异步I/O (aio_read)
   app ──aio_read()──→ 立即返回
   kernel做完所有事(等待+复制) ──→ 通知app

epoll 为什么比 select 快？
  select: 每次调用都要把fd集合从用户态拷贝到内核态, O(n)遍历
  epoll:  fd注册一次到内核(epoll_ctl), 就绪时回调通知, O(1)
  
  ┌──────────┬──────┬────────┬───────────┐
  │          │select│ poll   │ epoll     │
  ├──────────┼──────┼────────┼───────────┤
  │ fd上限   │ 1024 │ 无限制 │ 无限制    │
  │ 数据结构 │ 位图 │ 链表   │ 红黑树+链表│
  │ 消息传递 │ 拷贝 │ 拷贝   │ mmap共享  │
  │ 复杂度   │ O(n) │ O(n)   │ O(1)      │
  │ 适合场景 │ 少fd │ 少fd   │ 大量fd★  │
  └──────────┴──────┴────────┴───────────┘
```

### ◆ 进程间通信(IPC)全景图

```text
┌──────────────────────────────────────────────────────────┐
│                    IPC 方式选择                            │
├──────────┬──────────┬───────┬─────────┬─────────────────┤
│ 方式     │ 速度     │ 方向  │ 适用    │ 特点             │
├──────────┼──────────┼───────┼─────────┼─────────────────┤
│ 管道pipe │ 中       │ 单向  │ 父子进程│ 简单,4KB缓冲    │
│ FIFO     │ 中       │ 单向  │ 任意进程│ 有名管道,文件系统│
│ 信号signal│ 快      │ 异步  │ 任意进程│ 只传信号号,不传数据│
│ 消息队列 │ 中       │ 双向  │ 任意进程│ 有格式,内核维护  │
│ 共享内存 │ ★最快   │ 双向  │ 任意进程│ 需同步机制(信号量)│
│ 信号量sem│ —       │ 同步  │ 任意进程│ 不传数据,只做同步│
│ Socket   │ 慢       │ 双向  │ 可跨网络│ 最通用,TCP/UDP  │
│ mmap     │ 快       │ 双向  │ 任意进程│ 文件映射到内存   │
└──────────┴──────────┴───────┴─────────┴─────────────────┘

嵌入式中最常考:
  进程间大数据传输 → 共享内存 + 信号量同步
  父子进程通信 → 管道(pipe)
  网络通信 → Socket
  简单通知 → 信号(signal)
```

### ◆ 线程同步四件套

```bash
1. 互斥锁(Mutex): 保证临界区互斥访问(一次只一个线程)
   pthread_mutex_lock(&mtx);
   // 临界区: 操作共享资源
   pthread_mutex_unlock(&mtx);

2. 条件变量(Cond): 等待某个条件满足(生产者-消费者模型)
   // 消费者:
   pthread_mutex_lock(&mtx);
   while (queue_empty()) {  // ★必须用while不用if(防虚假唤醒)
       pthread_cond_wait(&cond, &mtx);  // 原子地释放锁+等待
   }
   // 消费数据
   pthread_mutex_unlock(&mtx);

   // 生产者:
   pthread_mutex_lock(&mtx);
   // 生产数据
   pthread_cond_signal(&cond);  // 唤醒一个等待线程
   pthread_mutex_unlock(&mtx);

3. 读写锁(RWLock): 读多写少场景优化
   多个读者可同时持锁, 写者独占
   pthread_rwlock_rdlock() / pthread_rwlock_wrlock()

4. 自旋锁(Spinlock): 忙等待,不睡眠(适合极短临界区)
   pthread_spin_lock(&spin);
   // 极短操作
   pthread_spin_unlock(&spin);

  ┌──────────┬──────────┬───────────┬──────────┐
  │          │ 互斥锁   │ 读写锁    │ 自旋锁   │
  ├──────────┼──────────┼───────────┼──────────┤
  │ 等待方式 │ 睡眠      │ 睡眠      │ 忙等待   │
  │ 适合场景 │ 通用      │ 读多写少  │ 极短临界区│
  │ 开销     │ 中(上下文)│ 中        │ 低(但占CPU)│
  │ 优先级反转│ 可能     │ 可能      │ 可能      │
  └──────────┴──────────┴───────────┴──────────┘
```

---

---

## 一、Linux 基础命令（Q1~Q15）

### Q1: 常用文件操作命令？

> 🧠 **秒懂：** ls/cp/mv/rm/find/chmod/chown是Linux文件操作的基本功。find + xargs组合是批量处理的利器。嵌入式开发中经常用这些命令操作交叉编译产物和设备文件。


以下是常用命令及其典型用法：

```bash
ls -la          # 列出所有文件(含隐藏), 显示详细信息
cd /path        # 切换目录
pwd             # 显示当前目录
cp -r src dst   # 递归复制
mv old new      # 移动/重命名
rm -rf dir      # 强制递归删除(危险!)
mkdir -p a/b/c  # 递归创建目录
touch file      # 创建空文件 / 更新时间戳
cat file        # 显示文件内容
find / -name "*.c"   # 按文件名查找
which gcc       # 查找可执行文件路径
```

**Linux IPC机制对比速查表：**

| IPC方式 | 数据方向 | 速度 | 适用场景 | 关键API |
|---------|---------|------|---------|--------|
| 管道(pipe) | 单向 | 中 | 父子进程简单通信 | pipe()+read/write |
| 命名管道(FIFO) | 单向 | 中 | 无亲缘关系进程 | mkfifo()+open |
| 消息队列 | 双向 | 中 | 带类型的结构化消息 | msgget/msgsnd/msgrcv |
| 共享内存 | 双向 | 最快 | 大量数据高速共享 | shmget/shmat+信号量同步 |
| 信号量 | 同步控制 | 快 | 进程间互斥/同步 | semget/semop |
| 信号(signal) | 异步通知 | 快 | 异常通知/进程控制 | kill/signal/sigaction |
| Socket | 双向 | 中 | 跨主机/同主机通用 | socket/bind/listen |
| mmap | 双向 | 最快 | 文件映射/零拷贝 | mmap/munmap |

---

### Q2: 文件权限 rwx？

> 🧠 **秒懂：** rwx分别代表读(4)、写(2)、执行(1)，三组分别给属主/属组/其他人。chmod 755表示属主rwx、其他rx。嵌入式中注意设备文件和脚本的执行权限。

```text
-rwxr-xr-- 1 user group 1234 Jan 1 file.txt
│└┬┘└┬┘└┬┘
│ │   │   └── 其他用户: r-- (只读)
│ │   └────── 组用户:   r-x (读+执行)
│ └────────── 属主:     rwx (读+写+执行)
└──────────── 文件类型: - 普通, d 目录, l 链接

权限数字表示: r=4, w=2, x=1
  chmod 755 file  →  rwxr-xr-x (属主全权限, 其他读+执行)
  chmod 644 file  →  rw-r--r-- (属主读写, 其他只读)
```

---


> 💡 **面试追问：** 进程/线程/协程的区别？线程共享什么？创建线程的开销是多少？
> 
> 🔧 **嵌入式建议：** 嵌入式Linux多线程:主线程管事件循环,工作线程处理业务。线程数不宜太多(一般4-8个)。


---
#### 📊 进程间通信(IPC)方式对比

| 方式 | 方向 | 速度 | 适用场景 | 备注 |
|------|------|------|---------|------|
| 管道(pipe) | 单向 | 中 | 父子进程 | 半双工 |
| 命名管道(FIFO) | 单向 | 中 | 无亲缘进程 | 文件系统可见 |
| 消息队列 | 双向 | 中 | 带类型的消息 | 有大小限制 |
| 共享内存 | 双向 | ⭐最快 | 大量数据交换 | 需要同步保护 |
| 信号(signal) | 单向 | 快(异步) | 事件通知 | 信息量少 |
| Socket | 双向 | 中 | 跨主机/通用 | 开销较大 |
| 信号量 | N/A | - | 同步/互斥 | 不传数据 |

---

### Q3: 常用文本处理命令？

> 🧠 **秒懂：** grep搜索文本、sed流编辑、awk列处理、sort排序、uniq去重、wc统计。嵌入式开发中常用grep+awk分析日志、sed批量修改配置文件。


以下是常用命令及其典型用法：

```bash
grep "pattern" file      # 搜索文本
grep -rn "func" src/     # 递归搜索, 显示行号
sed 's/old/new/g' file   # 替换文本
awk '{print $1}' file    # 按列处理
sort file                # 排序
uniq -c                  # 去重并计数
wc -l file               # 统计行数
head -20 file            # 前20行
tail -f log.txt          # 实时跟踪日志
cut -d: -f1 /etc/passwd  # 按分隔符取列
```

---

### Q4: 进程相关命令？

> 🧠 **秒懂：** ps查看进程、top动态监控、kill发信号、nice调优先级、strace跟踪系统调用。嵌入式调试中ps aux查看进程状态，top查看CPU/内存占用。


相关Linux命令速查：

```bash
ps aux           # 查看所有进程
top / htop       # 动态进程监控
kill -9 PID      # 强制杀进程
kill -15 PID     # 温和终止(SIGTERM)
jobs             # 查看后台任务
bg / fg          # 切换前后台
nohup cmd &      # 后台运行不受终端关闭影响
strace -p PID    # 跟踪系统调用(调试神器)
lsof -i :8080    # 查看端口被谁占用
```

---


> 💡 **面试追问：** select/poll/epoll谁最好？epoll为什么快？epoll的ET和LT模式区别？
> 
> 🔧 **嵌入式建议：** 嵌入式网络服务首选epoll+非阻塞IO。ET模式要循环读到EAGAIN;LT模式可读就通知(更安全)。


---
#### 📊 IO多路复用对比表

| 机制 | 最大fd数 | 数据结构 | 触发方式 | 效率 | 适用场景 |
|------|---------|---------|---------|------|---------|
| select | 1024(FD_SETSIZE) | fd_set位图 | 水平触发 | O(n)轮询 | 小规模/跨平台 |
| poll | 无限制 | pollfd数组 | 水平触发 | O(n)轮询 | 中规模 |
| epoll | 无限制 | 红黑树+就绪链表 | 水平/边沿 | O(1)就绪事件 | ⭐大规模/Linux |

> 💡 边沿触发(ET)性能更好但必须一次读完,水平触发(LT)更安全但通知更频繁

---

### Q5: GCC 编译选项？

> 🧠 **秒懂：** 常用选项：-Wall(全警告) -Werror(警告当错误) -O2/-Os(优化) -g(调试信息) -I(头文件路径) -L -l(库路径和库名) -static(静态链接) -march(目标架构)。

```bash
gcc -Wall -Wextra    # 开启所有警告
gcc -O0              # 不优化(调试用)
gcc -O2              # 常用优化级别
gcc -g               # 生成调试信息(配合GDB)
gcc -I./include      # 添加头文件搜索路径
gcc -L./lib          # 添加库文件搜索路径
gcc -lm              # 链接 libm.so (数学库)
gcc -static          # 静态链接(不依赖动态库)
gcc -shared -fPIC    # 生成动态库
gcc -DDEBUG          # 定义宏 DEBUG=1
gcc -std=c11         # 指定C标准

# 完整编译流程:
gcc -E file.c -o file.i   # 预处理
gcc -S file.i -o file.s   # 编译为汇编
gcc -c file.s -o file.o   # 汇编为目标文件
gcc file.o -o app          # 链接为可执行文件
```

---

### Q6: Makefile 基本语法？

> 🧠 **秒懂：** Makefile定义编译规则：目标:依赖→命令。变量(CC/CFLAGS)、自动变量($@目标/$<第一依赖/$^所有依赖)、%.o:%.c模式规则。是嵌入式项目构建的基础。


Makefile的基本语法规则如下：

```makefile
# 变量
CC = gcc
CFLAGS = -Wall -g
SRCS = $(wildcard src/*.c)
OBJS = $(SRCS:.c=.o)

# 目标: 依赖
#     命令(必须Tab开头)
app: $(OBJS)
$(CC) $(CFLAGS) -o $@ $^

# 模式规则
%.o: %.c
$(CC) $(CFLAGS) -c -o $@ $<

# 伪目标
.PHONY: clean
clean:
rm -f $(OBJS) app

# 自动变量:
# $@ = 目标名
# $< = 第一个依赖
# $^ = 所有依赖
```

---

### Q7: 静态库(.a)和动态库(.so)？

> 🧠 **秒懂：** 静态库(.a)在链接时合并进可执行文件(体积大但不依赖外部库)。动态库(.so)在运行时加载(体积小但需要部署库文件)。嵌入式资源受限时常用静态链接。


静态库在编译时链接，动态库在运行时加载：

```bash
# 创建静态库
gcc -c mylib.c -o mylib.o
ar rcs libmylib.a mylib.o      # 打包为 .a
gcc main.c -L. -lmylib -o app  # 链接(代码复制到可执行文件)

# 创建动态库
gcc -shared -fPIC mylib.c -o libmylib.so
gcc main.c -L. -lmylib -o app
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH  # 运行时找到库

#  对比:
# 静态库: 编译时嵌入, 可执行文件大, 不依赖外部
# 动态库: 运行时加载, 可执行文件小, 多程序共享, 升级方便
```

---

### Q8: 环境变量？

> 🧠 **秒懂：** PATH(可执行文件搜索路径)、LD_LIBRARY_PATH(动态库路径)、HOME、CROSS_COMPILE(交叉编译前缀)。export设置环境变量，source .bashrc使其生效。


环境变量影响进程的运行行为：

```bash
echo $PATH               # 可执行文件搜索路径
echo $LD_LIBRARY_PATH    # 动态库搜索路径
echo $HOME               # 用户主目录
export VAR=value         # 设置环境变量(当前shell及子进程)
env                      # 查看所有环境变量

# 永久生效: 写入 ~/.bashrc 或 /etc/profile
# C 程序中获取:
# char *val = getenv("PATH");
```

---

### Q9: 管道和重定向？

> 🧠 **秒懂：** 管道(|)将前一个命令的stdout连接到下一个的stdin。重定向(>/>>/<)将IO连接到文件。2>&1将stderr合并到stdout。嵌入式日志经常用重定向保存。

Linux管道和重定向是Shell编程的基础，也是日常开发必用：

```bash
cmd1 | cmd2         # 管道: cmd1 标准输出 → cmd2 标准输入
cmd > file          # 重定向标准输出到文件(覆盖)
cmd >> file         # 追加
cmd 2> err.log      # 重定向标准错误
cmd > out 2>&1      # 标准输出和错误都到 out
cmd < input.txt     # 文件作为标准输入

# 高级用法:
ls /not_exist 2>/dev/null  # 丢弃错误信息
tee file                   # 同时输出到屏幕和文件
```

---


### Q10: Shell脚本基础语法？

> 🧠 **秒懂：** #!/bin/bash开头、变量无需声明、$1-$9获取参数、if/for/while/case控制流、$(command)命令替换。嵌入式中Shell脚本用于自动化测试和系统初始化。

Shell脚本是嵌入式Linux开发中自动化测试、系统配置的基本工具：

```bash
#!/bin/bash
# 变量
NAME="embedded"
echo "Hello $NAME"    # 双引号内变量展开
echo 'Hello $NAME'    # 单引号内原样输出

# 条件判断
if [ -f /etc/passwd ]; then
    echo "File exists"
elif [ -d /tmp ]; then
    echo "Dir exists"
fi

# 循环
for i in 1 2 3; do
    echo "num: $i"
done

# while循环读文件
while read line; do
    echo "$line"
done < file.txt

# 函数
check_process() {
    if ps aux | grep -q "$1"; then
        return 0
    fi
    return 1
}
check_process "myapp" && echo "Running"
```

### Q11: Linux下如何调试段错误(Segmentation Fault)？

> 🧠 **秒懂：** ①GDB定位(bt查看调用栈)→②dmesg查看内核崩溃信息→③开启core dump(ulimit -c unlimited)→④GDB分析core文件→⑤ASAN编译检测。段错误通常是空指针或越界。

段错误是嵌入式开发中最常见的崩溃问题，调试方法如下：

```bash
# 方法1：GDB直接调试
gcc -g program.c -o program
gdb ./program
(gdb) run
# 崩溃后自动停在出错位置
(gdb) bt         # 打印调用栈
(gdb) info locals # 查看局部变量

# 方法2：Core dump分析
ulimit -c unlimited          # 开启core dump
./program                    # 崩溃后生成core文件
gdb ./program core           # 分析core
(gdb) bt                     # 看崩溃调用栈

# 方法3：AddressSanitizer(推荐)
gcc -fsanitize=address -g program.c -o program
./program
# 会精确报告内存错误位置
```

**常见段错误原因：**
- 空指针解引用
- 数组越界
- 已释放内存再访问(use-after-free)
- 栈溢出(递归太深/局部大数组)

### Q12: strace和ltrace的用法？

> 🧠 **秒懂：** strace跟踪系统调用(如open/read/write/ioctl的参数和返回值)。ltrace跟踪动态库函数调用。嵌入式中strace -e trace=open定位文件访问问题特别有用。

strace跟踪系统调用，ltrace跟踪库函数调用，是排查问题的利器：

```bash
# strace - 跟踪系统调用
strace ./myapp                   # 跟踪所有系统调用
strace -e open,read,write ./myapp  # 只看指定的
strace -p <pid>                  # 附加到运行中的进程
strace -c ./myapp                # 统计系统调用次数和耗时

# ltrace - 跟踪动态库调用
ltrace ./myapp                   # 跟踪所有库函数调用
ltrace -e malloc,free ./myapp    # 只看内存分配

# 实际排错示例：程序打开文件失败
strace -e open,openat ./myapp 2>&1 | grep "No such"
# 输出: openat(AT_FDCWD, "/etc/myapp.conf", O_RDONLY) = -1 ENOENT
```

### Q13: /proc文件系统在调试中的应用？


> 🧠 **秒懂：** /proc/PID/maps(内存映射)、/proc/PID/status(进程状态)、/proc/PID/fd(打开的文件描述符)、/proc/meminfo(系统内存)。不需要额外工具就能获取丰富的调试信息。

/proc是内核暴露到用户态的信息接口，嵌入式调试经常使用：

```bash
# 进程相关
cat /proc/<pid>/maps      # 内存映射(查找内存布局)
cat /proc/<pid>/status    # 进程状态(内存/线程数)
cat /proc/<pid>/fd/       # 打开的文件描述符
cat /proc/<pid>/stack     # 内核栈(需要root)
cat /proc/<pid>/cmdline   # 启动命令行

# 系统信息
cat /proc/meminfo         # 内存总体使用
cat /proc/cpuinfo         # CPU信息
cat /proc/interrupts      # 中断统计
cat /proc/version         # 内核版本
cat /proc/uptime          # 运行时间
```

### Q14: 什么是交叉编译？嵌入式开发为什么需要？

> 🧠 **秒懂：** 交叉编译是在PC(x86)上编译能在目标板(ARM)运行的程序。因为目标板资源有限跑不了编译器。工具链如arm-linux-gnueabihf-gcc。嵌入式开发的必备技能。

交叉编译是在一种平台(Host)上编译出另一种平台(Target)运行的程序：

```bash
# 典型交叉编译工具链命名: arch-vendor-os-abi-tool
arm-linux-gnueabihf-gcc hello.c -o hello   # ARM Linux
arm-none-eabi-gcc bare.c -o bare.elf       # ARM裸机(无OS)
aarch64-linux-gnu-gcc app.c -o app         # ARM64 Linux

# 为什么需要交叉编译:
# 1. 目标设备(MCU/ARM板)资源不足以运行编译器
# 2. 编译速度: PC上编译比目标板快得多
# 3. 开发便利: 在PC上编辑/编译/调试

# 交叉编译Makefile示例
CROSS_COMPILE = arm-linux-gnueabihf-
CC = $(CROSS_COMPILE)gcc
CFLAGS = -Wall -O2
TARGET = myapp

$(TARGET): main.c
	$(CC) $(CFLAGS) $< -o $@
```

### Q15: GDB远程调试的方法？

> 🧠 **秒懂：** 目标板运行gdbserver监听端口→PC的GDB用target remote ip:port连接→即可远程设断点、单步、查看变量。嵌入式Linux调试的标准方法。

嵌入式开发中通过GDB远程连接目标板进行调试：

```bash
# 目标板上运行gdbserver
gdbserver :1234 ./myapp          # 启动程序等待连接
gdbserver --attach :1234 <pid>   # 附加到运行中的进程

# PC上用交叉GDB连接
arm-linux-gnueabihf-gdb ./myapp
(gdb) target remote 192.168.1.100:1234
(gdb) break main
(gdb) continue
(gdb) bt
(gdb) info threads
```

**也可用JTAG/SWD调试(裸机/RTOS)：**
- J-Link + GDB Server
- OpenOCD + GDB

---

## 二、文件IO操作（Q16~Q30）

### Q16: open/read/write/close系统调用？


> 🧠 **秒懂：** open返回文件描述符(fd)→read/write通过fd读写数据→close关闭fd释放资源。底层系统调用，不带缓冲。返回-1表示错误，必须检查errno。

文件IO是Linux应用层最基础的操作：

```c
#include <fcntl.h>
#include <unistd.h>

// 打开文件
int fd = open("/tmp/test.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
if (fd < 0) {
    perror("open");
    return -1;
}

// 写入
const char *msg = "Hello\n";
ssize_t n = write(fd, msg, strlen(msg));

// 定位
lseek(fd, 0, SEEK_SET);  // 回到文件开头

// 读取
char buf[64];
n = read(fd, buf, sizeof(buf) - 1);
buf[n] = '\0';

close(fd);
```

**open的flags：**
- O_RDONLY/O_WRONLY/O_RDWR: 访问模式
- O_CREAT: 不存在则创建
- O_TRUNC: 存在则清空
- O_APPEND: 追加写
- O_NONBLOCK: 非阻塞

### Q17: 标准IO(fopen/fread)和系统IO(open/read)的区别？


> 🧠 **秒懂：** 标准IO(fopen/fread)带用户态缓冲区(减少系统调用次数)，系统IO(open/read)直接调用内核。标准IO效率更高(批量I/O)，系统IO控制更精细(无缓冲延迟)。

标准IO在系统IO之上加了用户态缓冲层：

| 对比项 | 标准IO(FILE*) | 系统IO(fd) |
|--------|--------------|-----------|
| 缓冲 | 有(全缓冲/行缓冲/无缓冲) | 无用户态缓冲 |
| 开销 | 减少系统调用(攒满buffer再write) | 每次调用都进入内核 |
| 可移植 | C标准,跨平台 | POSIX,仅Unix系 |
| 适用 | 普通文件IO | 设备文件/Socket/低层IO |

```c
// 标准IO的缓冲模式
setvbuf(fp, NULL, _IOFBF, 4096);  // 全缓冲(文件)
setvbuf(fp, NULL, _IOLBF, 0);     // 行缓冲(终端stdout)
setvbuf(fp, NULL, _IONBF, 0);     // 无缓冲(stderr)
```

### Q18: 文件描述符的复制(dup/dup2)？


> 🧠 **秒懂：** dup(oldfd)复制fd到最小可用fd号，dup2(oldfd, newfd)复制到指定fd号。经典用法：dup2(pipefd, STDOUT_FILENO)实现重定向——printf的输出就走管道了。

dup/dup2用于复制文件描述符，经典应用是重定向：

```c
#include <unistd.h>
#include <fcntl.h>

// 输出重定向到文件(printf输出到文件)
int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
int old_stdout = dup(STDOUT_FILENO);   // 备份stdout
dup2(fd, STDOUT_FILENO);               // 将fd复制到1号(stdout)
close(fd);

printf("This goes to file\n");        // 输出到output.txt

// 恢复stdout
dup2(old_stdout, STDOUT_FILENO);
close(old_stdout);
printf("Back to terminal\n");
```

### Q19: fcntl和ioctl的区别？


> 🧠 **秒懂：** fcntl操作文件描述符属性(设置非阻塞、文件锁等)。ioctl操作设备(设置串口波特率、控制外设等)。fcntl是通用文件操作，ioctl是设备特定控制——记住这个区别。

两者都用于控制文件描述符，但层级和用途不同：

| 对比项 | fcntl | ioctl |
|--------|-------|-------|
| 级别 | 通用文件描述符操作 | 设备特定操作 |
| 标准化 | POSIX标准 | 设备驱动自定义 |
| 典型用途 | 文件锁、fd属性、非阻塞设置 | 设备控制(如串口波特率) |

```c
// fcntl: 设置非阻塞
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// ioctl: 设置串口波特率
struct termios tio;
tcgetattr(fd, &tio);
cfsetispeed(&tio, B115200);
cfsetospeed(&tio, B115200);
tcsetattr(fd, TCSANOW, &tio);
```

### Q20: mmap内存映射文件的使用？

> 🧠 **秒懂：** mmap将文件映射到进程地址空间→普通的指针读写等于文件IO→munmap解除映射。零拷贝、高效。嵌入式中mmap常用于访问设备寄存器(/dev/mem)和进程间共享内存。

mmap将文件直接映射到进程地址空间，读写内存=读写文件：

```c
#include <sys/mman.h>
#include <fcntl.h>

int fd = open("data.bin", O_RDWR);
struct stat st;
fstat(fd, &st);

// 映射整个文件到内存
char *addr = mmap(NULL, st.st_size, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);
if (addr == MAP_FAILED) {
    perror("mmap");
    return -1;
}

// 直接通过指针操作文件内容
addr[0] = 'A';  // 修改会同步到文件(MAP_SHARED)

// 解除映射
munmap(addr, st.st_size);
close(fd);
```

**mmap vs read/write：**
- mmap优势：大文件随机访问、零拷贝(内核与用户态共享物理页)
- read优势：顺序小文件、代码简单

### Q21: select/poll/epoll的使用场景？


> 🧠 **秒懂：** select适合少量fd(≤1024)的简单场景。poll去掉了fd数量限制但仍需遍历。epoll基于事件驱动(O(1))适合大量连接。嵌入式Linux网络服务用epoll是标准实践。

IO多路复用让单线程同时等待多个fd事件：

```c
// epoll基本使用(嵌入式网络编程首选)
#include <sys/epoll.h>

int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[64];
while (1) {
    int n = epoll_wait(epfd, events, 64, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listen_fd) {
            // 新连接
            int client = accept(listen_fd, NULL, NULL);
            ev.events = EPOLLIN;
            ev.data.fd = client;
            epoll_ctl(epfd, EPOLL_CTL_ADD, client, &ev);
        } else {
            // 数据可读
            handle_client(events[i].data.fd);
        }
    }
}
```

### Q22: 文件锁(flock/fcntl锁)的使用？


> 🧠 **秒懂：** flock整文件加锁(简单)，fcntl锁可以锁文件的某段区域(字节范围锁)。文件锁用于多进程访问同一个文件时的同步。嵌入式配置文件修改、日志写入等场景会用到。

文件锁用于多进程协调访问同一文件：

```c
#include <fcntl.h>

int fd = open("shared.dat", O_RDWR);

// 方法1: flock(整个文件锁)
flock(fd, LOCK_EX);   // 加排他锁(阻塞等)
// ... 操作文件 ...
flock(fd, LOCK_UN);   // 解锁

// 方法2: fcntl(可锁文件区域)
struct flock fl;
fl.l_type = F_WRLCK;     // 写锁
fl.l_whence = SEEK_SET;
fl.l_start = 0;          // 从文件开始
fl.l_len = 100;          // 锁前100字节
fcntl(fd, F_SETLKW, &fl);  // 加锁(W=阻塞等待)
// ... 操作 ...
fl.l_type = F_UNLCK;
fcntl(fd, F_SETLK, &fl);   // 解锁
```

### Q23: inotify文件系统事件监控？


> 🧠 **秒懂：** inotify监控文件/目录的创建、删除、修改、移动等事件。inotify_init→inotify_add_watch→read事件。嵌入式中用于监控配置文件变化或热插拔设备节点。

inotify可以监控文件/目录的变化，嵌入式中用于配置文件热加载：

```c
// 关键系统调用示例(见各函数注释)
#include <sys/inotify.h>

int ifd = inotify_init();
int wd = inotify_add_watch(ifd, "/etc/myapp.conf",
                           IN_MODIFY | IN_DELETE);

char buf[4096];
while (1) {
    int len = read(ifd, buf, sizeof(buf));
    struct inotify_event *event;
    for (char *ptr = buf; ptr < buf + len;
         ptr += sizeof(*event) + event->len) {
        event = (struct inotify_event *)ptr;
        if (event->mask & IN_MODIFY)
            reload_config();
    }
}
```

### Q24: Linux下大文件的处理方法？


> 🧠 **秒懂：** 大文件处理：mmap分段映射、使用O_LARGEFILE标志、fseeko支持64位偏移、sendfile零拷贝传输。编译时定义_FILE_OFFSET_BITS=64启用大文件支持。

嵌入式设备可能需要处理日志等大文件：

```c
// 方法1: mmap分段映射
#define MAP_SIZE (4*1024*1024)  // 每次映射4MB
off_t offset = 0;
while (offset < file_size) {
    size_t len = MIN(MAP_SIZE, file_size - offset);
    char *addr = mmap(NULL, len, PROT_READ, MAP_SHARED, fd, offset);
    process_chunk(addr, len);
    munmap(addr, len);
    offset += len;
}

// 方法2: sendfile零拷贝传输
#include <sys/sendfile.h>
off_t off = 0;
sendfile(out_fd, in_fd, &off, file_size);
```

### Q25: /dev/null和/dev/zero的作用？


> 🧠 **秒懂：** /dev/null是黑洞(写入的数据全丢弃)，用于丢弃不需要的输出。/dev/zero是零源(读出的全是0)，用于初始化内存或创建空文件。嵌入式测试中经常用到。

这两个特殊设备文件在嵌入式开发中经常使用：

```bash
# /dev/null - 数据黑洞(丢弃所有写入)
./noisy_program > /dev/null 2>&1   # 丢弃所有输出
cat /dev/null > file.txt           # 清空文件

# /dev/zero - 零字节源(读取返回全0)
dd if=/dev/zero of=test.bin bs=1M count=10  # 创建10MB全零文件
# 用途: 初始化内存/磁盘区域

# /dev/urandom - 随机数源
dd if=/dev/urandom of=random.bin bs=256 count=1  # 256字节随机数
```

### Q26: 异步IO(aio)的使用？


> 🧠 **秒懂：** aio_read/aio_write发起异步IO→继续干其他事→aio_return/aio_error检查完成状态。或用io_uring(新一代异步IO，更高效)。适合高性能I/O密集场景。

异步IO提交请求后立即返回，完成后通知：

```c
#include <aio.h>

struct aiocb aio;
memset(&aio, 0, sizeof(aio));
aio.aio_fildes = fd;
aio.aio_buf = buffer;
aio.aio_nbytes = 4096;
aio.aio_offset = 0;

// 提交异步读
aio_read(&aio);

// 检查是否完成
while (aio_error(&aio) == EINPROGRESS) {
    // 做其他事情
    do_other_work();
}

// 获取结果
ssize_t ret = aio_return(&aio);
```

### Q27: 目录操作(opendir/readdir)？


> 🧠 **秒懂：** opendir打开目录→readdir逐个读取目录项(返回struct dirent含文件名和类型)→closedir关闭。配合stat获取文件详细信息。遍历目录树用递归。

遍历目录是嵌入式文件管理的常见需求：

```c
#include <dirent.h>

void list_files(const char *path) {
    DIR *dir = opendir(path);
    if (!dir) { perror("opendir"); return; }
    
    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL) {
        if (entry->d_name[0] == '.') continue;  // 跳过.和..
        
        if (entry->d_type == DT_DIR)
            printf("[DIR]  %s\n", entry->d_name);
        else
            printf("[FILE] %s\n", entry->d_name);
    }
    closedir(dir);
}
```

### Q28: stat系列函数获取文件信息？


> 🧠 **秒懂：** stat(path)/fstat(fd)/lstat(path,不跟软链接)获取文件的大小、权限、修改时间、inode号、文件类型等信息。struct stat中st_mode判断是文件/目录/设备。

stat获取文件元数据(大小、权限、修改时间等)：

```c
#include <sys/stat.h>

struct stat st;
if (stat("/tmp/test.txt", &st) == 0) {
    printf("Size: %ld\n", st.st_size);
    printf("Mode: %o\n", st.st_mode & 0777);
    printf("UID: %d\n", st.st_uid);
    printf("Modified: %s", ctime(&st.st_mtime));
    
    // 判断文件类型
    if (S_ISREG(st.st_mode)) printf("Regular file\n");
    if (S_ISDIR(st.st_mode)) printf("Directory\n");
    if (S_ISLNK(st.st_mode)) printf("Symlink\n");
}

// stat vs lstat: lstat不跟随符号链接
// stat vs fstat: fstat用fd而不是路径
```

### Q29: 文件系统空间查询(statfs)？


> 🧠 **秒懂：** statfs/fstatfs获取文件系统信息：总空间、可用空间、块大小、inode数量。嵌入式中用于监控Flash/SD卡剩余空间，防止日志写满文件系统。

嵌入式设备需要监控存储空间避免写满：

```c
// 关键系统调用示例(见各函数注释)
#include <sys/vfs.h>

struct statfs sfs;
if (statfs("/", &sfs) == 0) {
    unsigned long total = sfs.f_blocks * sfs.f_bsize;
    unsigned long free = sfs.f_bfree * sfs.f_bsize;
    unsigned long avail = sfs.f_bavail * sfs.f_bsize;
    printf("Total: %lu MB\n", total / 1024 / 1024);
    printf("Free:  %lu MB\n", free / 1024 / 1024);
    printf("Avail: %lu MB\n", avail / 1024 / 1024);
}
```

### Q30: tmpfile和mkstemp临时文件？

> 🧠 **秒懂：** tmpfile()创建匿名临时文件(自动删除)。mkstemp()创建有名临时文件(需手动unlink)。临时文件用于中间结果存储，避免程序崩溃后留下垃圾文件。

安全创建临时文件(避免竞态条件)：

```c
// 方法1: tmpfile(自动删除)
FILE *fp = tmpfile();  // 关闭时自动删除
fprintf(fp, "temp data");
rewind(fp);

// 方法2: mkstemp(需要手动删除)
char template[] = "/tmp/myapp_XXXXXX";
int fd = mkstemp(template);  // XXXXXX被替换为唯一字符串
write(fd, "data", 4);
close(fd);
unlink(template);  // 删除临时文件
```

---

## 三、进程管理（Q31~Q50）

### Q31: fork创建子进程的典型用法？


> 🧠 **秒懂：** fork后父子进程通过返回值区分→子进程exec加载新程序或执行任务→父进程wait回收子进程。典型场景：Shell执行命令、守护进程、多进程服务器。

fork在嵌入式Linux守护进程、多进程服务器中广泛使用：

```c
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        perror("fork");
        exit(1);
    } else if (pid == 0) {
        // 子进程
        printf("Child PID: %d\n", getpid());
        execl("/bin/ls", "ls", "-la", NULL);
        exit(1);  // exec失败才到这
    } else {
        // 父进程
        int status;
        waitpid(pid, &status, 0);
        if (WIFEXITED(status))
            printf("Child exited with %d\n", WEXITSTATUS(status));
    }
    return 0;
}
```

### Q32: wait/waitpid的作用和区别？


> 🧠 **秒懂：** wait阻塞等待任意子进程结束，waitpid可以指定等某个子进程且支持非阻塞(WNOHANG)。回收子进程防止变成僵尸进程。WIFSIGNALED等宏检查退出原因。

等待子进程结束并回收资源(避免僵尸进程)：

```c
// wait: 等待任意子进程
int status;
pid_t child = wait(&status);

// waitpid: 等待指定子进程, 支持非阻塞
pid_t child = waitpid(pid, &status, WNOHANG);
// WNOHANG: 如果子进程没结束立即返回0

// 解析退出状态
if (WIFEXITED(status))      // 正常退出
    printf("Exit code: %d\n", WEXITSTATUS(status));
if (WIFSIGNALED(status))    // 被信号杀死
    printf("Killed by signal: %d\n", WTERMSIG(status));
if (WIFSTOPPED(status))     // 被暂停
    printf("Stopped by signal: %d\n", WSTOPSIG(status));
```

### Q33: 如何创建守护进程(daemon)？


> 🧠 **秒懂：** 创建守护进程步骤：fork→父进程退出→setsid创建新会话→再fork→关闭所有fd→重定向stdin/stdout/stderr到/dev/null→chdir到/→写PID文件。

守护进程是后台运行、脱离终端的进程，嵌入式服务程序必备：

```c
#include <unistd.h>
#include <sys/stat.h>

void daemonize(void) {
    // 1. fork，父进程退出
    pid_t pid = fork();
    if (pid > 0) exit(0);
    if (pid < 0) exit(1);
    
    // 2. 创建新会话(脱离终端)
    setsid();
    
    // 3. 再次fork(防止重新关联终端)
    pid = fork();
    if (pid > 0) exit(0);
    if (pid < 0) exit(1);
    
    // 4. 修改工作目录
    chdir("/");
    
    // 5. 重设文件权限掩码
    umask(0);
    
    // 6. 关闭所有文件描述符
    for (int i = 0; i < sysconf(_SC_OPEN_MAX); i++)
        close(i);
    
    // 7. 重定向stdin/stdout/stderr到/dev/null
    open("/dev/null", O_RDWR);  // stdin  -> fd 0
    dup(0);                     // stdout -> fd 1
    dup(0);                     // stderr -> fd 2
}
```

### Q34: 进程组和会话的概念？


> 🧠 **秒懂：** 进程组是一组相关进程(如管道命令组)，会话是一组进程组(对应一个终端)。Ctrl+C发信号给前台进程组。守护进程通过setsid脱离终端会话。

理解进程组和会话对于信号管理和终端控制很重要：

```text
终端 tty ─── 会话(Session)
              │
              ├── 前台进程组(接收键盘输入和信号)
              │   ├── shell
              │   └── shell启动的前台命令
              │
              └── 后台进程组(&启动的)
                  ├── bg_proc1
                  └── bg_proc2

API:
  getpgrp()    - 获取进程组ID
  setpgid()    - 设置进程组
  getsid()     - 获取会话ID
  setsid()     - 创建新会话(daemon关键步骤)
  tcsetpgrp()  - 设置前台进程组
```

### Q35: exec族函数(execl/execv/execve)？


> 🧠 **秒懂：** exec族函数用新程序替换当前进程映像(PID不变)。l(列表参数)/v(数组参数)、p(搜索PATH)、e(指定环境变量)的组合。fork+exec是创建新进程的标准模式。

exec用新程序替换当前进程映像：

```c
// execl - 参数列表
execl("/bin/echo", "echo", "hello", "world", NULL);

// execv - 参数数组
char *args[] = {"echo", "hello", "world", NULL};
execv("/bin/echo", args);

// execvp - 自动搜索PATH
execlp("echo", "echo", "hello", NULL);

// execve - 指定环境变量
char *envp[] = {"PATH=/bin", "HOME=/tmp", NULL};
execve("/bin/echo", args, envp);
```

**exec后PID不变，但以下继承：**
- 文件描述符(除非设置FD_CLOEXEC)
- 进程ID、父进程ID
- 信号掩码

### Q36: system()和popen()的使用？


> 🧠 **秒懂：** system()执行Shell命令(创建子进程+Shell解析+等待完成)。popen()执行命令并通过管道读写其输入输出。popen更灵活，system更简单。注意：两者都有Shell注入风险。

执行shell命令的两种便捷方式：

```c
// system() - 执行命令等待结束
int ret = system("ls -la /tmp");
// ret = 子进程的exit status

// popen() - 执行命令并读取输出
FILE *fp = popen("ifconfig eth0", "r");
char buf[1024];
while (fgets(buf, sizeof(buf), fp)) {
    printf("%s", buf);
}
pclose(fp);

// 注意: system/popen有安全风险(命令注入)
// 嵌入式产品中应避免拼接用户输入到命令
```

### Q37: 进程间通信之管道(pipe)？


> 🧠 **秒懂：** pipe()创建匿名管道→fork→父进程关读端写数据→子进程关写端读数据。管道是半双工的——要双向通信需要创建两个管道。最基本的IPC。

管道用于父子进程/兄弟进程间单向数据传输：

```c
int pipefd[2];
pipe(pipefd);  // pipefd[0]读, pipefd[1]写

pid_t pid = fork();
if (pid == 0) {
    // 子进程：写数据
    close(pipefd[0]);
    const char *msg = "Hello from child";
    write(pipefd[1], msg, strlen(msg));
    close(pipefd[1]);
    exit(0);
} else {
    // 父进程：读数据
    close(pipefd[1]);
    char buf[256];
    int n = read(pipefd[0], buf, sizeof(buf) - 1);
    buf[n] = '\0';
    printf("Parent got: %s\n", buf);
    close(pipefd[0]);
    wait(NULL);
}
```

### Q38: 命名管道(FIFO)的使用？


> 🧠 **秒懂：** FIFO是有名字的管道(在文件系统中可见)——无亲缘关系的进程也能通信。mkfifo创建→一个进程open写→另一个open读。常用于简单的进程间数据传递。

FIFO允许无亲缘关系的进程通信：

```c
#include <sys/stat.h>

// 创建FIFO
mkfifo("/tmp/myfifo", 0666);

// 写者进程
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "data", 4);
close(fd);

// 读者进程(另一个程序)
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[256];
int n = read(fd, buf, sizeof(buf));
close(fd);
unlink("/tmp/myfifo");  // 用完删除
```

### Q39: 共享内存(POSIX接口)？


> 🧠 **秒懂：** shm_open创建共享内存对象→ftruncate设置大小→mmap映射到进程→读写→munmap→shm_unlink。是最快的IPC方式——直接操作同一块物理内存。需要配合同步机制。

POSIX共享内存是进程间高速通信的方式：

```c
#include <sys/mman.h>
#include <fcntl.h>

// 创建共享内存对象
int shm_fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
ftruncate(shm_fd, 4096);

// 映射到进程空间
char *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                  MAP_SHARED, shm_fd, 0);

// 写入数据
sprintf(addr, "Hello shared memory");

// 另一个进程读取
int shm_fd2 = shm_open("/my_shm", O_RDONLY, 0);
char *addr2 = mmap(NULL, 4096, PROT_READ, MAP_SHARED, shm_fd2, 0);
printf("Got: %s\n", addr2);

// 清理
munmap(addr, 4096);
shm_unlink("/my_shm");
```

### Q40: 消息队列(POSIX接口)？

> 🧠 **秒懂：** mq_open创建消息队列→mq_send发消息→mq_receive收消息→mq_close→mq_unlink。消息队列自带同步(空时收阻塞、满时发阻塞)，比管道支持消息优先级。

消息队列支持带优先级的消息传递：

```c
#include <mqueue.h>

// 创建消息队列
struct mq_attr attr = { .mq_maxmsg = 10, .mq_msgsize = 256 };
mqd_t mq = mq_open("/my_mq", O_CREAT | O_RDWR, 0666, &attr);

// 发送消息(优先级=1)
mq_send(mq, "Hello", 5, 1);

// 接收消息
char buf[256];
unsigned int prio;
ssize_t n = mq_receive(mq, buf, sizeof(buf), &prio);
printf("Received: %.*s (prio=%u)\n", (int)n, buf, prio);

// 清理
mq_close(mq);
mq_unlink("/my_mq");
```

### Q41: 信号(signal)的处理方法？


> 🧠 **秒懂：** signal()/sigaction()注册信号处理函数。sigaction更强大(可获取信号信息、控制行为)。信号处理函数中只能调用异步信号安全函数(不能用printf/malloc)。

信号处理在嵌入式Linux中用于进程控制和异常处理：

```c
#include <signal.h>

volatile sig_atomic_t running = 1;  // volatile: 防止编译器优化

void sighandler(int sig) {
    if (sig == SIGINT || sig == SIGTERM)
        running = 0;
}

int main() {
    // 推荐使用sigaction(比signal更可靠)
    struct sigaction sa;
    sa.sa_handler = sighandler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);  // 注册信号处理
    sigaction(SIGTERM, &sa, NULL);  // 注册信号处理
    
    // 忽略SIGPIPE(网络编程必须)
    signal(SIGPIPE, SIG_IGN);  // 注册信号处理
    
    while (running) {
        do_work();
        usleep(100000);
    }
    printf("Graceful shutdown\n");
    return 0;
}
```

### Q42: 信号量(POSIX有名/无名信号量)？


> 🧠 **秒懂：** 有名信号量(sem_open，跨进程)和无名信号量(sem_init，线程间或共享内存)。sem_wait减1(为0阻塞)→sem_post加1(唤醒等待者)。用于控制资源访问和同步。

信号量用于进程/线程间的同步和互斥：

```c
#include <semaphore.h>

// 无名信号量(线程间)
sem_t sem;
sem_init(&sem, 0, 1);  // 0=线程间, 初始值1(互斥)
sem_wait(&sem);  // P操作
// 临界区
sem_post(&sem);  // V操作
sem_destroy(&sem);

// 有名信号量(进程间)
sem_t *sem = sem_open("/my_sem", O_CREAT, 0666, 1);
sem_wait(sem);
// 临界区
sem_post(sem);
sem_close(sem);
sem_unlink("/my_sem");
```

### Q43: 多进程并发服务器模型？


> 🧠 **秒懂：** 主进程listen→accept后fork子进程处理连接→父进程继续accept→子进程处理完exit。简单直观但每连接一个进程开销大。改进：预fork进程池或改用多线程/epoll。

嵌入式网络设备的简易多进程服务器：

```c
// 预fork模式(避免每次连接都fork)
#define WORKER_NUM 4

int listen_fd = socket_bindlisten(8080);

for (int i = 0; i < WORKER_NUM; i++) {
    if (fork() == 0) {
        // 子进程循环accept
        while (1) {
            int client_fd = accept(listen_fd, NULL, NULL);
            handle_request(client_fd);
            close(client_fd);
        }
        exit(0);
    }
}
// 父进程等待子进程
while (wait(NULL) > 0);
```

### Q44: 进程资源限制(setrlimit)？


> 🧠 **秒懂：** setrlimit设置进程资源限制：最大文件数(RLIMIT_NOFILE)、最大栈大小(RLIMIT_STACK)、最大CPU时间。嵌入式中用于限制进程资源使用防止资源耗尽。

限制进程资源使用，防止单个进程耗尽系统资源：

```c
#include <sys/resource.h>

// 限制最大文件描述符数
struct rlimit rl;
rl.rlim_cur = 1024;   // 软限制
rl.rlim_max = 4096;   // 硬限制
setrlimit(RLIMIT_NOFILE, &rl);

// 限制进程最大虚拟内存(防OOM)
rl.rlim_cur = 100 * 1024 * 1024;  // 100MB
rl.rlim_max = 200 * 1024 * 1024;
setrlimit(RLIMIT_AS, &rl);

// 限制core dump大小
rl.rlim_cur = RLIM_INFINITY;
setrlimit(RLIMIT_CORE, &rl);  // 允许生成core文件
```

### Q45: 进程优先级设置(nice/setpriority)？


> 🧠 **秒懂：** nice值(-20到19)调整进程优先级：值越低优先级越高。实时进程用sched_setscheduler设置SCHED_FIFO/RR。嵌入式中关键任务设高优先级保证实时性。

调整进程优先级影响CPU调度：

```c
#include <sys/resource.h>
#include <unistd.h>

// nice值范围: -20(最高) ~ 19(最低), 默认0
nice(5);  // 降低当前进程优先级(nice值+5)

// setpriority(更灵活)
setpriority(PRIO_PROCESS, 0, -10);  // 0=当前进程, nice=-10
int prio = getpriority(PRIO_PROCESS, 0);

// 实时优先级(需root)
struct sched_param param;
param.sched_priority = 50;
sched_setscheduler(0, SCHED_FIFO, &param);
```

### Q46: 什么是僵尸进程？如何避免？


> 🧠 **秒懂：** 僵尸进程：子进程退出但父进程没有wait回收，进程表项残留。避免方法：①wait/waitpid②SIGCHLD信号处理中调用waitpid③signal(SIGCHLD, SIG_IGN)。必须回收不然进程表满。

僵尸进程是已退出但未被父进程回收(wait)的子进程：

```c
// 方法1: 父进程调用wait/waitpid
while (waitpid(-1, NULL, WNOHANG) > 0);

// 方法2: 忽略SIGCHLD(最简单)
signal(SIGCHLD, SIG_IGN);

// 方法3: double fork
if (fork() == 0) {
    if (fork() == 0) {
        // 孙子进程(被init收养，不会变僵尸)
        do_work();
        exit(0);
    }
    exit(0);  // 子进程立即退出
}
wait(NULL);  // 回收子进程(瞬间)
```

### Q47: 进程间通过Unix Domain Socket通信？


> 🧠 **秒懂：** Unix Domain Socket在本机进程间通信：和TCP socket类似的API(socket/bind/listen/accept)但走内核不走网络协议栈，效率比TCP/IP高。还能传递文件描述符。

Unix域套接字是本地进程通信的高效方式：

```c
// 服务端
int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/my.sock", sizeof(addr.sun_path)-1);
unlink(addr.sun_path);
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sfd, 5);
int cfd = accept(sfd, NULL, NULL);

// 客户端
int cfd = socket(AF_UNIX, SOCK_STREAM, 0);
connect(cfd, (struct sockaddr *)&addr, sizeof(addr));
write(cfd, "hello", 5);
```

**优势：** 比TCP快(无网络协议栈开销)，支持传递文件描述符(SCM_RIGHTS)。

### Q48: 多进程调试技巧？


> 🧠 **秒懂：** GDB中set follow-fork-mode child跟踪子进程。或用detach-on-fork off同时附加父子。多进程调试用gdbserver分别attach不同进程。strace -f跟踪fork后的子进程。

嵌入式多进程程序调试方法：

```bash
# GDB: follow-fork-mode
(gdb) set follow-fork-mode child    # fork后跟踪子进程
(gdb) set detach-on-fork off        # fork后不分离任何进程
(gdb) info inferiors                # 查看所有被调试进程
(gdb) inferior 2                    # 切换到进程2

# strace跟踪子进程
strace -f ./multi_proc_app          # -f跟踪fork的子进程

# 日志法(嵌入式最常用)
#define LOG(fmt, ...) fprintf(stderr, "[%d][%s:%d] " fmt "\n", \
        getpid(), __func__, __LINE__, ##__VA_ARGS__)
```

### Q49: 环境变量的操作？


> 🧠 **秒懂：** getenv获取、setenv设置、unsetenv删除、putenv设置。子进程继承父进程的环境变量。嵌入式中CROSS_COMPILE、PATH等环境变量影响编译和运行。

环境变量在嵌入式启动脚本和配置中广泛使用：

```c
#include <stdlib.h>

// 获取
char *home = getenv("HOME");

// 设置(覆盖已有)
setenv("MY_VAR", "value", 1);

// 设置(不覆盖)
setenv("MY_VAR", "value", 0);

// 删除
unsetenv("MY_VAR");

// putenv(直接使用传入的字符串，不复制)
putenv("KEY=value");

// 遍历所有环境变量
extern char **environ;
for (char **ep = environ; *ep; ep++)
    printf("%s\n", *ep);
```

### Q50: Linux进程内存查看方法？


> 🧠 **秒懂：** 方法：①/proc/PID/status(VmRSS实际内存) ②/proc/PID/maps(详细映射) ③top/htop(实时监控) ④pmap(进程内存映射) ⑤valgrind(详细分析)。RSS是进程实际占用的物理内存。

分析进程内存使用是嵌入式性能优化的基础：

```bash
# 方法1: /proc/<pid>/status
grep -E "VmRSS|VmSize|Threads" /proc/<pid>/status
# VmSize: 虚拟内存总量
# VmRSS:  物理内存(实际占用)

# 方法2: /proc/<pid>/maps (内存映射详情)
cat /proc/<pid>/maps
# 地址范围  权限 偏移 设备 inode 路径
# 00400000-00401000 r-xp ... /usr/bin/myapp (代码段)

# 方法3: /proc/<pid>/smaps (详细内存统计)
cat /proc/<pid>/smaps | grep -E "^Size|Rss|Pss"

# 方法4: top/htop
top -p <pid>  # 监控特定进程
```

---

## 四、线程编程（Q51~Q70）

### Q51: pthread线程创建和基本使用？


> 🧠 **秒懂：** pthread_create创建线程(传入函数和参数)→线程执行→pthread_join等待结束(或pthread_detach分离)。线程共享进程的地址空间——全局变量共享但要注意同步。

POSIX线程是Linux多线程编程的标准接口：

```c
#include <pthread.h>

void *thread_func(void *arg) {
    int id = *(int *)arg;
    printf("Thread %d running\n", id);
    return (void *)(long)id;
}

int main() {
    pthread_t tid;
    int arg = 42;
    
    // 创建线程
    pthread_create(&tid, NULL, thread_func, &arg);
    
    // 等待线程结束
    void *retval;
    pthread_join(tid, &retval);
    printf("Thread returned: %ld\n", (long)retval);
    
    return 0;
}
// 编译: gcc -pthread program.c
```

### Q52: 线程属性设置(pthread_attr)？


> 🧠 **秒懂：** pthread_attr_setdetachstate(分离/可连接)、pthread_attr_setstacksize(栈大小)、pthread_attr_setschedpolicy(调度策略)。嵌入式中常调小线程栈大小节省内存。

设置线程栈大小、分离状态等属性：

```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// 设置分离状态(分离后不能join)
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

// 设置栈大小(嵌入式需要控制内存)
pthread_attr_setstacksize(&attr, 64 * 1024);  // 64KB

// 设置调度策略和优先级
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
struct sched_param param = { .sched_priority = 50 };
pthread_attr_setschedparam(&attr, &param);

pthread_t tid;
pthread_create(&tid, &attr, thread_func, NULL);
pthread_attr_destroy(&attr);
```

### Q53: 互斥锁(pthread_mutex)的使用？

> 🧠 **秒懂：** pthread_mutex_lock加锁→操作共享资源→pthread_mutex_unlock解锁。加锁失败则阻塞等待。必须确保所有路径都能解锁(包括错误路径)。是最常用的线程同步机制。

互斥锁是线程同步最基本的方式：

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// 或动态初始化(可设置属性)
pthread_mutexattr_t mattr;
pthread_mutexattr_init(&mattr);
pthread_mutexattr_settype(&mattr, PTHREAD_MUTEX_ERRORCHECK); // 错误检查型
pthread_mutex_init(&mutex, &mattr);

// 加锁/解锁
pthread_mutex_lock(&mutex);
// 临界区...
pthread_mutex_unlock(&mutex);

// trylock(非阻塞)
if (pthread_mutex_trylock(&mutex) == 0) {
    // 获取成功
    pthread_mutex_unlock(&mutex);
} else {
    // 锁已被占用
}

pthread_mutex_destroy(&mutex);
```

### Q54: 条件变量(pthread_cond)的使用？

> 🧠 **秒懂：** 条件变量配合互斥锁实现'等待-通知'：mutex保护条件→pthread_cond_wait(释放锁+睡眠+被唤醒后重新加锁)→pthread_cond_signal唤醒。经典的生产者-消费者模型。

条件变量用于线程间的等待/通知机制：

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int data_ready = 0;

// 等待方(消费者)
pthread_mutex_lock(&mutex);
while (!data_ready)  // 必须用while(防止虚假唤醒)
    pthread_cond_wait(&cond, &mutex);  // 原子释放锁+等待
// 处理数据
data_ready = 0;
pthread_mutex_unlock(&mutex);

// 通知方(生产者)
pthread_mutex_lock(&mutex);
data_ready = 1;
pthread_cond_signal(&cond);   // 唤醒一个等待者
// pthread_cond_broadcast(&cond); // 唤醒所有等待者
pthread_mutex_unlock(&mutex);
```


> 全网最详细的嵌入式实习/秋招/春招公司清单，正在更新中，需要可关注微信公众号：嵌入式校招菌。

### Q55: 读写锁(pthread_rwlock)？


> 🧠 **秒懂：** 读写锁允许多个线程同时读(读锁兼容)，但写锁独占。适合读多写少的场景(如配置数据、缓存)。比互斥锁并发度高——多个读者不互相阻塞。

读写锁适合读多写少的场景(如配置数据)：

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// 读者(可并发)
pthread_rwlock_rdlock(&rwlock);
read_shared_data();
pthread_rwlock_unlock(&rwlock);

// 写者(独占)
pthread_rwlock_wrlock(&rwlock);
modify_shared_data();
pthread_rwlock_unlock(&rwlock);
```

### Q56: 线程安全和可重入的区别？

> 🧠 **秒懂：** 线程安全：多线程调用不出错(可以用锁实现)。可重入：同一函数被多次调用(包括中断重入)不出错(不能用锁、不能用全局变量)。可重入是线程安全的子集。

面试常考区分线程安全与可重入的概念：

| 概念 | 含义 | 实现方式 |
|------|------|----------|
| 线程安全 | 多线程同时调用结果正确 | 加锁/TLS/原子操作 |
| 可重入 | 被中断后再次进入仍正确 | 不用全局/static变量、不用锁 |

```c
// 不可重入+非线程安全
char *buf;  // 全局
char *strtok(char *str, const char *delim);  // 经典反面

// 线程安全但不可重入(用了锁)
pthread_mutex_t lock;
int safe_counter(void) {
    pthread_mutex_lock(&lock);
    static int count = 0;
    count++;
    pthread_mutex_unlock(&lock);
    return count;
}

// 可重入(完全不用共享状态)
int add(int a, int b) { return a + b; }
```

### Q57: 线程局部存储(TLS/pthread_key)？

> 🧠 **秒懂：** TLS让每个线程有自己的变量副本——即使变量名相同，各线程访问的是不同的存储。pthread_key_create/pthread_setspecific/getspecific或__thread关键字。用于替代全局变量。

线程私有数据避免加锁和共享冲突：

```c
// 方法1: __thread修饰符(GCC扩展)
__thread int tls_var = 0;  // 每个线程有独立副本

// 方法2: pthread_key
pthread_key_t key;
pthread_key_create(&key, free);  // free=析构函数

// 每个线程设置自己的数据
char *data = malloc(64);
snprintf(data, 64, "Thread %d data", id);
pthread_setspecific(key, data);

// 获取当前线程的数据
char *my_data = pthread_getspecific(key);
```

### Q58: 线程池的设计和实现？

> 🧠 **秒懂：** 线程池预创建一组线程，任务提交到队列→空闲线程取任务执行→执行完放回池中。避免频繁创建/销毁线程的开销。实现：任务队列+条件变量+工作线程。

线程池避免频繁创建销毁线程的开销：

```c
#define POOL_SIZE 4
#define QUEUE_SIZE 64

typedef struct {  // 结构体定义
    void (*func)(void *arg);
    void *arg;
} Task;

typedef struct {  // 结构体定义
    pthread_t threads[POOL_SIZE];
    Task queue[QUEUE_SIZE];
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    int shutdown;
} ThreadPool;

void *worker(void *arg) {
    ThreadPool *pool = arg;
    while (1) {
        pthread_mutex_lock(&pool->lock);  // 加锁(互斥)
        while (pool->count == 0 && !pool->shutdown)
            pthread_cond_wait(&pool->not_empty, &pool->lock);
        if (pool->shutdown) {
            pthread_mutex_unlock(&pool->lock);  // 解锁
            return NULL;
        }
        Task task = pool->queue[pool->head];
        pool->head = (pool->head + 1) % QUEUE_SIZE;
        pool->count--;
        pthread_cond_signal(&pool->not_full);  // 注册信号处理
        pthread_mutex_unlock(&pool->lock);  // 解锁
        
        task.func(task.arg);  // 执行任务
    }
}
```

### Q59: 线程取消(pthread_cancel)？

> 🧠 **秒懂：** pthread_cancel请求取消线程(不是立即杀死)。线程在取消点(如sleep/read/write等系统调用)才会响应取消。可以用pthread_setcancelstate禁用取消。慎用——可能导致资源泄漏。

取消正在执行的线程(需谨慎使用)：

```c
pthread_t tid;
pthread_create(&tid, NULL, worker, NULL);

// 请求取消
pthread_cancel(tid);

// 线程中设置取消属性
pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);   // 允许取消
pthread_setcanceltype(PTHREAD_CANCEL_DEFERRED, NULL);  // 延迟取消(在取消点)

// 清理函数(类似析构)
void cleanup(void *arg) {
    free(arg);
    pthread_mutex_unlock(&some_mutex);
}
pthread_cleanup_push(cleanup, resource);
// ... 工作代码 ...
pthread_cleanup_pop(1);  // 1=执行清理, 0=不执行
```

### Q60: 线程屏障(pthread_barrier)？


> 🧠 **秒懂：** 屏障让一组线程都到达同一点后才继续执行。pthread_barrier_init(设置计数)→各线程调用pthread_barrier_wait→最后一个到达的触发所有线程继续。适合并行计算的阶段同步。

屏障让多个线程在某一点集合同步：

```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 4);  // 4个线程

void *thread_func(void *arg) {
    // 阶段1: 各自初始化
    init_work();
    
    // 屏障等待: 4个线程都到这才继续
    pthread_barrier_wait(&barrier);
    
    // 阶段2: 所有线程初始化完毕后开始协作
    cooperative_work();
    return NULL;
}
```

### Q61: 自旋锁(pthread_spinlock)？

> 🧠 **秒懂：** 自旋锁在获取失败时循环忙等(不让出CPU)。适合临界区极短(微秒级)且不会睡眠的场景。多核系统避免上下文切换开销——但单核上自旋浪费CPU，不如用互斥锁。

自旋锁在用户态的实现(适合极短临界区)：

```c
pthread_spinlock_t spinlock;
pthread_spin_init(&spinlock, PTHREAD_PROCESS_PRIVATE);

pthread_spin_lock(&spinlock);
// 极短临界区(几十ns以内)
counter++;
pthread_spin_unlock(&spinlock);

pthread_spin_destroy(&spinlock);
```

### Q62: 原子操作(GCC内建)？

> 🧠 **秒懂：** __sync_fetch_and_add等GCC内建原子操作，或C11的<stdatomic.h>。编译器保证操作的原子性。用于无锁计数器、标志位等简单场景，比锁开销小得多。

GCC内置原子操作用于无锁编程：

```c
// C11标准原子类型
#include <stdatomic.h>
atomic_int counter = 0;
atomic_fetch_add(&counter, 1);
atomic_store(&counter, 0);
int val = atomic_load(&counter);

// GCC内建(兼容旧版本)
int val = __sync_fetch_and_add(&counter, 1);
__sync_lock_test_and_set(&lock, 1);  // 设置并返回旧值
__sync_lock_release(&lock);           // 释放
bool ok = __sync_bool_compare_and_swap(&val, expected, desired);
```

### Q63: 死锁分析与避免实践？

> 🧠 **秒懂：** 死锁分析：①画出锁的获取依赖图找环 ②用GDB的info threads查看线程状态和持有锁 ③TSAN(ThreadSanitizer)编译检测。预防：固定加锁顺序、使用trylock、减少锁粒度。

多线程开发中死锁定位和避免：

```bash
# 检测死锁: GDB
(gdb) thread apply all bt   # 查看所有线程调用栈
# 死锁特征: 多个线程都阻塞在pthread_mutex_lock

# 检测工具: Valgrind Helgrind
valgrind --tool=helgrind ./multi_thread_app
```

```c
// 避免规则
// 1. 锁序一致(全局规定加锁顺序)
void safe_transfer(Account *a, Account *b, int amount) {
    Account *first = (a < b) ? a : b;   // 地址小的先锁
    Account *second = (a < b) ? b : a;
    pthread_mutex_lock(&first->lock);
    pthread_mutex_lock(&second->lock);
    // ... 转账操作 ...
    pthread_mutex_unlock(&second->lock);
    pthread_mutex_unlock(&first->lock);
}
```

### Q64: 线程与信号的交互？

> 🧠 **秒懂：** 多线程中信号发送给进程的任意线程(不确定哪个收到)。最佳实践：主线程设置信号掩码(所有线程继承)→专门一个线程sigwait处理信号。其他线程不受信号干扰。

多线程中信号处理需要特别注意：

```c
// 问题: 信号被随机线程处理

// 解决: 指定线程处理信号
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGUSR1);

// 主线程阻塞信号
pthread_sigmask(SIG_BLOCK, &set, NULL);

// 创建专门的信号处理线程
void *signal_thread(void *arg) {
    int sig;
    while (1) {
        sigwait(&set, &sig);  // 同步等待信号
        printf("Got signal %d\n", sig);
    }
}
```

### Q65: CPU亲和性(CPU Affinity)？


> 🧠 **秒懂：** CPU亲和性把线程绑定到特定CPU核心：sched_setaffinity。减少Cache失效(线程不会在核心间迁移)。嵌入式多核系统中把实时任务绑定到专用核心提高确定性。

绑定线程到特定CPU核，减少迁移开销：

```c
#define _GNU_SOURCE
#include <sched.h>

cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(0, &cpuset);  // 绑定到CPU0

// 设置线程亲和性
pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

// 或用sched_setaffinity设置进程
sched_setaffinity(0, sizeof(cpuset), &cpuset);
```

**嵌入式应用：** 大小核架构中将实时任务绑定到大核，将非关键任务绑定到小核。

### Q66: 线程安全的单例模式实现？

> 🧠 **秒懂：** 双检锁(DCLP)+volatile/atomic：if(!inst){lock(); if(!inst){inst=new Singleton;} unlock();}。C++11更简单：局部static(Meyers' Singleton)保证线程安全。

C语言实现线程安全单例(无C++的static初始化保证)：

```c
// 方法1: pthread_once(推荐)
static pthread_once_t once = PTHREAD_ONCE_INIT;
static Config *instance = NULL;

void init_config(void) {
    instance = malloc(sizeof(Config));
    load_config(instance, "/etc/app.conf");
}

Config *get_config(void) {
    pthread_once(&once, init_config);
    return instance;
}

// 方法2: 静态初始化(简单场景)
static Config config = { .inited = 0 };
static pthread_mutex_t cfg_lock = PTHREAD_MUTEX_INITIALIZER;

Config *get_config(void) {
    if (!config.inited) {
        pthread_mutex_lock(&cfg_lock);
        if (!config.inited) {  // double-check
            load_config(&config);
            config.inited = 1;
        }
        pthread_mutex_unlock(&cfg_lock);
    }
    return &config;
}
```

### Q67: 无锁队列(Lock-free Queue)的原理？

> 🧠 **秒懂：** 无锁队列用CAS(Compare-And-Swap)原子操作替代互斥锁。入队/出队时先读指针→计算新指针→CAS更新(失败则重试)。避免锁的开销和死锁风险，但编程复杂。

无锁编程在高性能嵌入式系统中使用：

```c
// 单生产者-单消费者无锁环形队列(最常用)
#define QUEUE_SIZE 1024  // 必须是2的幂

typedef struct {
    int buffer[QUEUE_SIZE];
    volatile unsigned int head;  // 消费者移动
    volatile unsigned int tail;  // 生产者移动
} SPSCQueue;

int enqueue(SPSCQueue *q, int item) {
    unsigned int next = (q->tail + 1) & (QUEUE_SIZE - 1);
    if (next == q->head) return -1;  // 满
    q->buffer[q->tail] = item;
    __sync_synchronize();  // 内存屏障
    q->tail = next;
    return 0;
}

int dequeue(SPSCQueue *q, int *item) {
    if (q->head == q->tail) return -1;  // 空
    *item = q->buffer[q->head];
    __sync_synchronize();
    q->head = (q->head + 1) & (QUEUE_SIZE - 1);
    return 0;
}
```

### Q68: eventfd用于线程/进程间通知？

> 🧠 **秒懂：** eventfd创建事件通知fd→write写入计数→read读出并清零(或减)。可以配合epoll使用。比pipe效率更高，是Linux下线程/进程间轻量级通知的推荐方式。

eventfd是Linux轻量级通知机制(替代pipe)：

```c
#include <sys/eventfd.h>

int efd = eventfd(0, EFD_NONBLOCK | EFD_SEMAPHORE);

// 通知(写)
uint64_t val = 1;
write(efd, &val, sizeof(val));

// 等待(读) - 可配合epoll
uint64_t val;
read(efd, &val, sizeof(val));  // val=通知次数

// 配合epoll使用(高效等待)
struct epoll_event ev = { .events = EPOLLIN, .data.fd = efd };
epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &ev);
```

### Q69: timerfd定时器的使用？

> 🧠 **秒懂：** timerfd_create创建定时器fd→timerfd_settime设置定时→read阻塞等待到期(返回到期次数)。可以放入epoll统一管理。比signal方式更优雅、更可控。

timerfd将定时器封装为文件描述符，可与epoll集成：

```c
#include <sys/timerfd.h>

int tfd = timerfd_create(CLOCK_MONOTONIC, 0);

// 设置1秒后首次触发，之后每500ms触发一次
struct itimerspec its;
its.it_value.tv_sec = 1;
its.it_value.tv_nsec = 0;
its.it_interval.tv_sec = 0;
its.it_interval.tv_nsec = 500000000;  // 500ms
timerfd_settime(tfd, 0, &its, NULL);

// 等待定时器(配合epoll更佳)
uint64_t expirations;
read(tfd, &expirations, sizeof(expirations));
printf("Timer fired %lu times\n", expirations);
```

### Q70: signalfd信号处理？


> 🧠 **秒懂：** signalfd将信号转换为文件描述符→用read读取信号信息→可以放入epoll统一管理。解决了信号处理函数中不能调用非异步安全函数的问题。现代Linux编程推荐。

signalfd将信号转为文件描述符(可与epoll统一事件处理)：

```c
#include <sys/signalfd.h>

sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);
sigaddset(&mask, SIGTERM);
sigprocmask(SIG_BLOCK, &mask, NULL);

int sfd = signalfd(-1, &mask, 0);

// 用read读取信号信息(而非异步signal handler)
struct signalfd_siginfo info;
read(sfd, &info, sizeof(info));
printf("Got signal %d from PID %d\n", info.ssi_signo, info.ssi_pid);
```

---

## 五、网络编程（Q71~Q80）

### Q71: socket编程基本流程(TCP)？

> 🧠 **秒懂：** TCP服务器：socket→bind→listen→accept→read/write→close。TCP客户端：socket→connect→read/write→close。TCP面向连接可靠传输。是网络编程的基本功。

TCP客户端-服务器是嵌入式网络编程的基础：

```c
// 服务端
int sfd = socket(AF_INET, SOCK_STREAM, 0);
int opt = 1;
setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port = htons(8080),
    .sin_addr.s_addr = INADDR_ANY
};
bind(sfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sfd, 5);

int cfd = accept(sfd, NULL, NULL);
char buf[1024];
int n = read(cfd, buf, sizeof(buf));
write(cfd, buf, n);  // 回显
close(cfd);
close(sfd);

// 客户端
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in srv = {
    .sin_family = AF_INET,
    .sin_port = htons(8080)
};
inet_pton(AF_INET, "192.168.1.1", &srv.sin_addr);
connect(fd, (struct sockaddr *)&srv, sizeof(srv));
write(fd, "hello", 5);
```


> 💡 **面试追问：** TCP为什么要三次握手？两次行不行？TIME_WAIT是什么？为什么需要？
> 
> 🔧 **嵌入式建议：** 服务端必须设SO_REUSEADDR(避免TIME_WAIT绑定失败);嵌入式TCP要设keepalive检测断线。

### Q72: UDP编程与TCP的区别？

> 🧠 **秒懂：** UDP无连接：sendto/recvfrom直接发送/接收(无需connect/accept)。比TCP快但不保证可靠——可能丢包、乱序、重复。嵌入式中用于实时性要求高但允许少量丢包的场景。

UDP是无连接的传输层协议，适合实时性要求高的嵌入式通信：

```c
// UDP发送
int fd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in dest = {
    .sin_family = AF_INET,
    .sin_port = htons(9000)
};
inet_pton(AF_INET, "192.168.1.100", &dest.sin_addr);
sendto(fd, "data", 4, 0, (struct sockaddr *)&dest, sizeof(dest));

// UDP接收
bind(fd, (struct sockaddr *)&local, sizeof(local));
struct sockaddr_in from;
socklen_t len = sizeof(from);
int n = recvfrom(fd, buf, sizeof(buf), 0, 
                 (struct sockaddr *)&from, &len);
```

| TCP | UDP |
|-----|-----|
| 面向连接 | 无连接 |
| 可靠(重传/排序) | 不可靠(可能丢/乱序) |
| 字节流 | 数据报(有边界) |
| 适合文件传输 | 适合音视频/传感器数据 |

### Q73: 网络字节序转换？

> 🧠 **秒懂：** 网络字节序是大端(Big-Endian)。htonl/htons(主机→网络)、ntohl/ntohs(网络→主机)。发送前转网络序，接收后转主机序。不转换在小端机器上收发的数据会乱。

网络通信中字节序问题是嵌入式常见bug来源：

```c
#include <arpa/inet.h>

// 主机→网络
uint16_t net_port = htons(8080);       // host to network short
uint32_t net_addr = htonl(0xC0A80101); // host to network long

// 网络→主机
uint16_t host_port = ntohs(net_port);
uint32_t host_addr = ntohl(net_addr);

// IP地址转换
struct in_addr addr;
inet_pton(AF_INET, "192.168.1.1", &addr);  // 字符串→二进制
char str[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &addr, str, sizeof(str)); // 二进制→字符串
```

### Q74: setsockopt常用选项？

> 🧠 **秒懂：** SO_REUSEADDR(地址复用，避免TIME_WAIT)、SO_KEEPALIVE(TCP保活)、TCP_NODELAY(关闭Nagle)、SO_RCVBUF/SO_SNDBUF(缓冲区大小)。嵌入式网络服务必须设SO_REUSEADDR。

socket选项设置是网络编程必须掌握的：

```c
int opt = 1;
// 地址复用(避免TIME_WAIT导致bind失败)
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

// 设置发送/接收缓冲区
int bufsize = 65536;
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));

// TCP_NODELAY(禁用Nagle算法，减少延迟)
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));

// 设置超时
struct timeval tv = { .tv_sec = 5, .tv_usec = 0 };
setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));

// Keepalive(检测断连)
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
```

### Q75: 非阻塞IO与epoll结合？

> 🧠 **秒懂：** 非阻塞socket+epoll_wait等待事件→读/写直到EAGAIN→继续等待。epoll_ctl添加/修改/删除监控的fd。是Linux高性能网络服务器的标准范式。

这是嵌入式高性能网络服务器的核心模式：

```c
// 设置非阻塞
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// epoll + 非阻塞(reactor模式)
int epfd = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN | EPOLLET, .data.fd = fd };
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

struct epoll_event events[128];
while (1) {
    int n = epoll_wait(epfd, events, 128, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listen_fd) {
            accept_new_connection(listen_fd);
        } else if (events[i].events & EPOLLIN) {
            // 非阻塞读(ET模式必须循环读)
            while (1) {
                int ret = read(events[i].data.fd, buf, sizeof(buf));
                if (ret < 0 && errno == EAGAIN) break;
                if (ret <= 0) { close_connection(); break; }
                process_data(buf, ret);
            }
        }
    }
}
```

### Q76: TCP粘包问题和解决方案？

> 🧠 **秒懂：** TCP是字节流，没有消息边界。粘包：多个消息粘在一起。解决方案：①固定长度 ②消息头+长度字段+数据 ③特殊分隔符(如\n)。嵌入式自定义协议通常用方案②。

TCP是字节流协议，没有消息边界，需要应用层分包：

```c
// 方案1: 固定长度
#define MSG_LEN 128
char buf[MSG_LEN];
int total = 0;
while (total < MSG_LEN) {
    int n = read(fd, buf + total, MSG_LEN - total);
    if (n <= 0) break;
    total += n;
}

// 方案2: 长度+数据(最常用)
typedef struct {
    uint32_t len;      // 网络字节序
    char data[0];      // 柔性数组
} Packet;

// 发送
uint32_t len = htonl(data_len);
write(fd, &len, 4);
write(fd, data, data_len);

// 接收
uint32_t len;
read_n(fd, &len, 4);
len = ntohl(len);
char *data = malloc(len);
read_n(fd, data, len);

// 方案3: 分隔符(如\n)
// 适合文本协议(HTTP, SMTP)
```

### Q77: 多路复用实现简单HTTP服务器？

> 🧠 **秒懂：** epoll管理所有连接fd+listen fd→新连接到来accept并加入epoll→数据到来read并处理→发送响应write→关闭连接。单线程即可处理上千连接。

嵌入式设备常需要提供Web配置页面：

```c
void handle_http(int fd) {
    char buf[4096];
    int n = read(fd, buf, sizeof(buf) - 1);
    buf[n] = '\0';
    
    // 简易HTTP响应
    const char *response =
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/html\r\n"
        "Content-Length: 13\r\n"
        "\r\n"
        "Hello World!\n";
    write(fd, response, strlen(response));
    close(fd);
}
```

### Q78: 多播(Multicast)编程？

> 🧠 **秒懂：** 多播让一个数据包同时发送给一组接收者(一对多)。setsockopt设置IP_ADD_MEMBERSHIP加入多播组。嵌入式中用于设备发现(mDNS)、实时数据分发。

多播用于一对多通信(如设备发现、传感器广播)：

```c
// 发送多播
int fd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in mcast_addr = {
    .sin_family = AF_INET,
    .sin_port = htons(5000)
};
inet_pton(AF_INET, "239.1.1.1", &mcast_addr.sin_addr);
sendto(fd, "discover", 8, 0, (struct sockaddr *)&mcast_addr, sizeof(mcast_addr));

// 接收多播
struct ip_mreq mreq;
inet_pton(AF_INET, "239.1.1.1", &mreq.imr_multiaddr);
mreq.imr_interface.s_addr = INADDR_ANY;
setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));
bind(fd, (struct sockaddr *)&local, sizeof(local));
recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
```

### Q79: raw socket原始套接字？

> 🧠 **秒懂：** 原始套接字(SOCK_RAW)绕过TCP/UDP直接操作IP层或链路层。可以构造自定义协议包、抓包分析。嵌入式中用于实现自定义网络协议或网络调试。

原始套接字可以自定义IP包头(网络调试/协议开发)：

```c
// 创建原始套接字(需root)
int fd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);

// 接收ICMP包(如ping回复)
char buf[4096];
struct sockaddr_in from;
socklen_t len = sizeof(from);
int n = recvfrom(fd, buf, sizeof(buf), 0, (struct sockaddr *)&from, &len);
// buf包含IP头+ICMP头+数据
```

### Q80: 网络调试工具和方法？


> 🧠 **秒懂：** 工具：ping(连通性)、traceroute(路由)、netstat/ss(连接状态)、tcpdump(抓包)、wireshark(分析)、nc(网络调试瑞士军刀)、iperf(带宽测试)。嵌入式网络调试利器。

嵌入式网络问题调试常用工具：

```bash
# tcpdump - 抓包(嵌入式最常用)
tcpdump -i eth0 port 8080 -w capture.pcap
tcpdump -i eth0 host 192.168.1.100 -nn

# netstat/ss - 连接状态
ss -tlnp          # 查看监听端口
ss -s             # 统计汇总

# ping / traceroute - 连通性
ping -c 3 192.168.1.1
traceroute 8.8.8.8

# nc(netcat) - 网络调试瑞士军刀
nc -l 8080         # 监听端口
nc 192.168.1.1 8080  # 连接
echo "test" | nc -u 192.168.1.1 9000  # UDP发送
```

---

## 六、Shell脚本与自动化（Q81~Q90）

### Q81: Shell脚本中的变量和参数？

> 🧠 **秒懂：** 变量赋值(NAME=value，无空格)、引用($NAME或${NAME})、位置参数($1-$9)、特殊变量($?返回值/$#参数个数/$@所有参数/$$当前PID)。

Shell脚本在嵌入式系统启动、测试、部署中广泛使用：

```bash
#!/bin/bash
# 位置参数
echo "脚本名: $0"
echo "第一个参数: $1"
echo "所有参数: $@"
echo "参数个数: $#"
echo "上一命令返回值: $?"
echo "当前PID: $$"

# 变量操作
STR="hello world"
echo ${#STR}         # 字符串长度: 11
echo ${STR:0:5}      # 子串: hello
echo ${STR/world/linux}  # 替换: hello linux
echo ${STR^^}        # 大写: HELLO WORLD
```

### Q82: Shell条件判断和循环？

> 🧠 **秒懂：** if [ 条件 ]; then ... fi。for i in list; do ... done。while [ 条件 ]; do ... done。case $var in pattern) ... ;; esac。注意[和]周围必须有空格。

自动化测试脚本的核心结构：

```bash
#!/bin/bash
# 文件判断
[ -f "$1" ] && echo "是普通文件"
[ -d "$1" ] && echo "是目录"
[ -x "$1" ] && echo "有执行权限"

# 字符串判断
[ -z "$VAR" ] && echo "空字符串"
[ "$A" = "$B" ] && echo "相等"

# 数值判断
[ $a -eq $b ]  # 等于
[ $a -gt $b ]  # 大于
[ $a -lt $b ]  # 小于

# for循环
for file in /dev/ttyUSB*; do
    echo "Found serial port: $file"
done

# while循环读取文件
while IFS=',' read -r name value; do
    echo "$name = $value"
done < config.csv
```

### Q83: Shell中的函数和返回值？

> 🧠 **秒懂：** function fname() { ... return N; }。return返回退出状态(0-255)，不是返回值。函数输出用echo→命令替换$(fname)获取。函数中的变量默认全局，用local声明局部。

Shell函数用于封装重复操作：

```bash
#!/bin/bash
# 函数定义
check_device() {
    local device=$1
    if [ -c "$device" ]; then
        echo "Device $device ok"
        return 0
    else
        echo "Device $device not found"
        return 1
    fi
}

# 函数调用
check_device "/dev/ttyS0"
if [ $? -eq 0 ]; then
    echo "Ready to communicate"
fi

# 获取函数输出
get_ip() {
    ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
}
MY_IP=$(get_ip)
echo "My IP: $MY_IP"
```

### Q84: Shell文本处理(grep/sed/awk)？

> 🧠 **秒懂：** grep -r 'pattern' dir(搜索文件内容)。sed 's/old/new/g' file(批量替换)。awk '{print $1,$3}' file(按列处理)。三剑客组合能处理几乎所有文本分析需求。

嵌入式日志分析和数据处理的核心工具：

```bash
# grep - 搜索匹配
grep -r "error" /var/log/          # 递归搜索
grep -n "pattern" file              # 显示行号
grep -c "warning" log.txt           # 计数
grep -E "err|warn|fail" log.txt     # 正则

# sed - 流编辑
sed -i 's/old/new/g' file          # 原地替换
sed -n '10,20p' file               # 打印10-20行
sed '/^#/d' config.txt              # 删除注释行

# awk - 列处理
awk '{print $1, $3}' file          # 打印第1和第3列
awk -F: '{print $1}' /etc/passwd   # 指定分隔符
awk '$3 > 100 {print $0}' data     # 条件过滤
awk '{sum+=$1} END{print sum}' file  # 求和
```

### Q85: Shell脚本实现自动化测试？

> 🧠 **秒懂：** Shell脚本实现自动化测试：运行程序→对比输出和期望值→统计通过/失败数→生成报告。嵌入式中用于自动化烧录测试、串口回归测试、批量固件验证。

嵌入式设备自动化测试脚本示例：

```bash
#!/bin/bash
# 串口通信测试
DEVICE="/dev/ttyUSB0"
BAUD=115200
PASS=0
FAIL=0

test_uart() {
    local cmd=$1
    local expect=$2
    
    # 发送命令并等待回复
    echo "$cmd" > "$DEVICE"
    sleep 0.5
    local response=$(timeout 2 cat "$DEVICE")
    
    if echo "$response" | grep -q "$expect"; then
        echo "[PASS] $cmd -> $expect"
        ((PASS++))
    else
        echo "[FAIL] $cmd -> got: $response"
        ((FAIL++))
    fi
}

# 配置串口
stty -F "$DEVICE" $BAUD cs8 -parenb -cstopb

# 执行测试
test_uart "AT" "OK"
test_uart "AT+VERSION?" "V1."
test_uart "AT+BAUD?" "$BAUD"

echo "Results: $PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] && exit 0 || exit 1
```

### Q86: Makefile自动化构建？

> 🧠 **秒懂：** Makefile高级：伪目标(.PHONY)、条件编译(ifeq)、include包含、自动依赖生成(gcc -MMD)、多目录编译。嵌入式项目模板：定义CROSS_COMPILE、CFLAGS、链接脚本路径。

Makefile是嵌入式项目构建的标准工具：

```makefile
# 嵌入式项目典型Makefile
CROSS_COMPILE ?= arm-linux-gnueabihf-
CC = $(CROSS_COMPILE)gcc  # 编译器
CFLAGS = -Wall -O2 -g  # 编译选项
LDFLAGS = -lpthread  # 链接选项

SRC = $(wildcard src/*.c)  # 源文件
OBJ = $(SRC:.c=.o)  # 目标文件
TARGET = myapp  # 输出文件名

all: $(TARGET)  # 默认构建目标

$(TARGET): $(OBJ)
	$(CC) $(LDFLAGS) $^ -o $@

%.o: %.c  # 构建目标: %.o
	$(CC) $(CFLAGS) -c $< -o $@

clean:  # 清理构建产物
	rm -f $(OBJ) $(TARGET)

install: $(TARGET)  # 默认构建目标
	cp $(TARGET) /opt/rootfs/usr/bin/

.PHONY: all clean install  # 声明伪目标
```

### Q87: CMake在嵌入式中的使用？

> 🧠 **秒懂：** CMake是跨平台构建系统：CMakeLists.txt定义项目→cmake生成Makefile→make编译。嵌入式中设置toolchain file指定交叉编译器。比手写Makefile更易维护和移植。

现代嵌入式项目越来越多使用CMake管理构建：

```cmake
cmake_minimum_required(VERSION 3.10)
project(EmbeddedApp C)

# 交叉编译工具链
set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

# 编译选项
add_compile_options(-Wall -O2)

# 源文件
file(GLOB SOURCES "src/*.c")

# 生成可执行文件
add_executable(myapp ${SOURCES})
target_link_libraries(myapp pthread)

# 安装规则
install(TARGETS myapp DESTINATION /opt/rootfs/usr/bin)
```

### Q88: 嵌入式系统启动脚本编写？

> 🧠 **秒懂：** 启动脚本(/etc/init.d/或systemd的.service文件)控制系统服务的启动/停止/重启。嵌入式中编写启动脚本：挂载文件系统→配置网络→启动看门狗→启动应用程序。

嵌入式Linux系统启动脚本(init.d风格)：

```bash
#!/bin/sh
# /etc/init.d/myservice
DAEMON=/usr/bin/myapp
PIDFILE=/var/run/myapp.pid
NAME="myapp"

start() {
    echo "Starting $NAME..."
    start-stop-daemon -S -b -m -p $PIDFILE -x $DAEMON
}

stop() {
    echo "Stopping $NAME..."
    start-stop-daemon -K -p $PIDFILE
    rm -f $PIDFILE
}

case "$1" in
    start) start ;;
    stop) stop ;;
    restart) stop; sleep 1; start ;;
    *) echo "Usage: $0 {start|stop|restart}" ;;
esac
```

### Q89: Python在嵌入式测试中的应用？

> 🧠 **秒懂：** Python在嵌入式测试中：pyserial串口通信、pexpect自动化交互、pytest测试框架、matplotlib数据可视化。PC端的测试上位机常用Python快速开发。

Python脚本用于嵌入式设备测试和数据分析：

```python
import serial
import time

# 串口通信测试
ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=2)

def send_cmd(cmd, expect='OK'):
    ser.write((cmd + '\r\n').encode())
    time.sleep(0.5)
    response = ser.read(ser.in_waiting).decode()
    assert expect in response, f"Expected '{expect}', got '{response}'"
    print(f"[PASS] {cmd}")

# 执行测试
send_cmd('AT')
send_cmd('AT+RST', 'ready')
send_cmd('AT+GMR', 'V1.')

ser.close()
print("All tests passed!")
```

### Q90: 嵌入式日志系统设计？


> 🧠 **秒懂：** 日志系统设计：分级(ERROR/WARN/INFO/DEBUG)→带时间戳→支持输出到串口/文件/网络→环形缓冲区→可运行时调整日志级别。嵌入式中日志是定位现场问题的关键手段。

嵌入式系统日志是调试和运维的重要工具：

```c
// 关键系统调用示例(见各函数注释)
#include <stdarg.h>
#include <time.h>

typedef enum { LOG_DEBUG, LOG_INFO, LOG_WARN, LOG_ERROR } LogLevel;  // 枚举定义
static LogLevel g_level = LOG_INFO;
static FILE *g_logfile = NULL;

void log_init(const char *path, LogLevel level) {
    g_logfile = fopen(path, "a");
    g_level = level;
}

void log_msg(LogLevel level, const char *fmt, ...) {
    if (level < g_level) return;
    
    const char *tags[] = {"DBG", "INF", "WRN", "ERR"};
    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    
    fprintf(g_logfile, "[%02d:%02d:%02d][%s] ",
            t->tm_hour, t->tm_min, t->tm_sec, tags[level]);
    
    va_list ap;
    va_start(ap, fmt);
    vfprintf(g_logfile, fmt, ap);
    va_end(ap);
    fprintf(g_logfile, "\n");
    fflush(g_logfile);
}

#define LOG_I(fmt, ...) log_msg(LOG_INFO, fmt, ##__VA_ARGS__)
#define LOG_E(fmt, ...) log_msg(LOG_ERROR, fmt, ##__VA_ARGS__)
```

---

## 七、高级主题（Q91~Q125）

### Q91: Linux下串口编程？

> 🧠 **秒懂：** 打开串口open(/dev/ttyS0)→tcgetattr获取属性→cfsetispeed/cfsetospeed设波特率→设数据位/停止位/校验→tcsetattr应用→read/write通信→close。

嵌入式最常用的通信接口编程：

```c
#include <termios.h>
#include <fcntl.h>

int uart_open(const char *dev, int baud) {
    int fd = open(dev, O_RDWR | O_NOCTTY);
    if (fd < 0) return -1;
    
    struct termios tio;
    tcgetattr(fd, &tio);
    
    // 设置波特率
    cfsetispeed(&tio, baud);
    cfsetospeed(&tio, baud);
    
    // 8N1
    tio.c_cflag &= ~PARENB;   // 无校验
    tio.c_cflag &= ~CSTOPB;   // 1停止位
    tio.c_cflag &= ~CSIZE;
    tio.c_cflag |= CS8;        // 8数据位
    tio.c_cflag |= CLOCAL | CREAD;
    
    // 原始模式(非规范)
    tio.c_lflag &= ~(ICANON | ECHO | ECHOE | ISIG);
    tio.c_iflag &= ~(IXON | IXOFF | IXANY);
    tio.c_oflag &= ~OPOST;
    
    // 超时设置
    tio.c_cc[VMIN] = 0;
    tio.c_cc[VTIME] = 10;  // 1秒超时
    
    tcsetattr(fd, TCSANOW, &tio);
    tcflush(fd, TCIOFLUSH);
    return fd;
}
```


> 💡 **面试追问：** termios的cflag/iflag/oflag/lflag各控制什么？怎么设置非规范模式？
> 
> 🔧 **嵌入式建议：** 串口是嵌入式Linux最常用的通信,调试/AT命令/传感器通信都靠它。推荐非规范模式+raw设置。

### Q92: I2C设备编程(Linux用户态)？

> 🧠 **秒懂：** 打开I2C设备(/dev/i2c-N)→ioctl设置从机地址(I2C_SLAVE)→write发送数据/read接收数据。或使用i2c_smbus_read/write系列函数。嵌入式Linux下访问传感器的标准方式。

通过/dev/i2c-x接口访问I2C设备：

```c
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>

int i2c_read_reg(int fd, uint8_t addr, uint8_t reg, uint8_t *val) {
    if (ioctl(fd, I2C_SLAVE, addr) < 0) return -1;  // 设备控制操作
    if (write(fd, &reg, 1) != 1) return -1;
    if (read(fd, val, 1) != 1) return -1;
    return 0;
}

int main() {
    int fd = open("/dev/i2c-1", O_RDWR);  // 打开设备/文件
    uint8_t val;
    i2c_read_reg(fd, 0x68, 0x75, &val);  // 读MPU6050 WHO_AM_I
    printf("WHO_AM_I: 0x%02x\n", val);
    close(fd);  // 关闭文件描述符
}
```

### Q93: SPI设备编程(Linux用户态)？

> 🧠 **秒懂：** 打开SPI设备(/dev/spidevX.Y)→ioctl设置模式/速率/位宽(SPI_IOC_WR_MODE)→ioctl(SPI_IOC_MESSAGE)执行全双工传输。用户态SPI访问简单但实时性不如内核驱动。

通过/dev/spidevX.Y接口访问SPI设备：

```c
#include <linux/spi/spidev.h>
#include <sys/ioctl.h>

int spi_transfer(int fd, uint8_t *tx, uint8_t *rx, int len) {
    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = len,
        .speed_hz = 1000000,
        .bits_per_word = 8,
    };
    return ioctl(fd, SPI_IOC_MESSAGE(1), &tr);  // 设备控制操作
}

int main() {
    int fd = open("/dev/spidev0.0", O_RDWR);  // 打开设备/文件
    uint8_t mode = SPI_MODE_0;
    ioctl(fd, SPI_IOC_WR_MODE, &mode);  // 设备控制操作
    
    uint8_t tx[] = {0x9F};  // 读Flash JEDEC ID
    uint8_t rx[4] = {0};
    spi_transfer(fd, tx, rx, sizeof(tx));
    printf("JEDEC: %02x %02x %02x\n", rx[1], rx[2], rx[3]);
    close(fd);  // 关闭文件描述符
}
```

### Q94: GPIO操作(sysfs和libgpiod)？

> 🧠 **秒懂：** 旧方式：echo导出/设方向/读写sysfs文件(/sys/class/gpio/)。新方式：libgpiod(gpiod_chip_open→gpiod_line_request→set/get_value)。libgpiod更高效且支持事件监控。

嵌入式Linux中GPIO控制的两种方式：

```c
// 方法1: sysfs(旧接口, 简单但将被废弃)
// echo 18 > /sys/class/gpio/export
// echo out > /sys/class/gpio/gpio18/direction
// echo 1 > /sys/class/gpio/gpio18/value

// 方法2: libgpiod(新接口, 推荐)
#include <gpiod.h>

struct gpiod_chip *chip = gpiod_chip_open("/dev/gpiochip0");
struct gpiod_line *line = gpiod_chip_get_line(chip, 18);

// 输出
gpiod_line_request_output(line, "myapp", 0);
gpiod_line_set_value(line, 1);  // 高电平

// 输入(带中断等待)
gpiod_line_request_rising_edge_events(line, "myapp");
struct gpiod_line_event event;
gpiod_line_event_wait(line, NULL);
gpiod_line_event_read(line, &event);

gpiod_line_release(line);
gpiod_chip_close(chip);
```

### Q95: PWM控制(Linux sysfs)？

> 🧠 **秒懂：** 通过sysfs操作PWM：echo N > export→设置period和duty_cycle(纳秒单位)→echo 1 > enable。Linux PWM子系统统一管理，应用层操作简单。

嵌入式中PWM用于电机控制、LED调光等：

```bash
# sysfs PWM控制
echo 0 > /sys/class/pwm/pwmchip0/export
echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period      # 周期1ms(1kHz)
echo 500000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle   # 占空比50%
echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable
```

```c
// C代码操作PWM
void pwm_set(int chip, int channel, int period_ns, int duty_ns) {
    char path[128];
    sprintf(path, "/sys/class/pwm/pwmchip%d/pwm%d/period", chip, channel);
    int fd = open(path, O_WRONLY);
    dprintf(fd, "%d", period_ns);
    close(fd);
    
    sprintf(path, "/sys/class/pwm/pwmchip%d/pwm%d/duty_cycle", chip, channel);
    fd = open(path, O_WRONLY);
    dprintf(fd, "%d", duty_ns);
    close(fd);
}
```

### Q96: Linux下CAN总线编程？

> 🧠 **秒懂：** socketCAN：socket(PF_CAN, SOCK_RAW, CAN_RAW)→bind绑定can0接口→write发送CAN帧→read接收CAN帧。用ip link set can0 type can bitrate 500000配置波特率。

SocketCAN是Linux标准CAN接口：

```c
#include <linux/can.h>
#include <net/if.h>
#include <sys/ioctl.h>

int can_init(const char *ifname) {
    int fd = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    struct ifreq ifr;
    strncpy(ifr.ifr_name, ifname, IFNAMSIZ);
    ioctl(fd, SIOCGIFINDEX, &ifr);  // 设备控制操作
    
    struct sockaddr_can addr = {
        .can_family = AF_CAN,
        .can_ifindex = ifr.ifr_ifindex
    };
    bind(fd, (struct sockaddr *)&addr, sizeof(addr));  // 绑定地址
    return fd;
}

// 发送CAN帧
struct can_frame frame;
frame.can_id = 0x123;
frame.can_dlc = 8;
memcpy(frame.data, "\x01\x02\x03\x04\x05\x06\x07\x08", 8);  // 内存拷贝
write(fd, &frame, sizeof(frame));

// 接收CAN帧
struct can_frame rx;
read(fd, &rx, sizeof(rx));
printf("ID: 0x%X, Data[0]: 0x%X\n", rx.can_id, rx.data[0]);
```

### Q97: 内存映射IO(寄存器访问)？

> 🧠 **秒懂：** mmap映射/dev/mem或设备文件到用户空间→通过指针直接读写寄存器：volatile uint32_t *reg = mmap(...)。嵌入式Linux用户态调试硬件的快捷方法，但注意安全风险。

嵌入式Linux中通过/dev/mem或UIO直接访问硬件寄存器：

```c
#include <sys/mman.h>
#include <fcntl.h>

#define GPIO_BASE 0x3F200000  // 树莓派GPIO基地址
#define BLOCK_SIZE 4096

int fd = open("/dev/mem", O_RDWR | O_SYNC);
volatile uint32_t *gpio = mmap(NULL, BLOCK_SIZE, 
                                PROT_READ | PROT_WRITE,
                                MAP_SHARED, fd, GPIO_BASE);

// 设置GPIO4为输出
gpio[0] |= (1 << 12);  // GPFSEL0: GPIO4 = output

// 输出高电平
gpio[7] = (1 << 4);    // GPSET0: GPIO4 = 1

// 输出低电平
gpio[10] = (1 << 4);   // GPCLR0: GPIO4 = 0

munmap((void *)gpio, BLOCK_SIZE);
close(fd);
```

### Q98: watchdog看门狗编程？

> 🧠 **秒懂：** 打开/dev/watchdog→定期write喂狗→关闭时未写'V'则不停止看门狗(保持保护)。ioctl设超时时间(WDIOC_SETTIMEOUT)。嵌入式Linux系统可靠性的保障。

Linux下硬件看门狗的使用：

```c
#include <linux/watchdog.h>
#include <sys/ioctl.h>

int wdt_fd = open("/dev/watchdog", O_WRONLY);

// 设置超时时间(秒)
int timeout = 10;
ioctl(wdt_fd, WDIOC_SETTIMEOUT, &timeout);

// 喂狗(必须定期调用)
while (running) {
    ioctl(wdt_fd, WDIOC_KEEPALIVE, NULL);
    // 或者 write(wdt_fd, "1", 1);
    sleep(5);  // 喂狗间隔 < 超时时间
}

// 关闭看门狗(写入magic字符'V')
write(wdt_fd, "V", 1);
close(wdt_fd);
```

### Q99: D-Bus进程间通信？

> 🧠 **秒懂：** D-Bus是Linux桌面/嵌入式系统的进程间通信总线。系统总线(system bus)和会话总线(session bus)。嵌入式中用于系统服务间通信(如NetworkManager通知网络状态变化)。

D-Bus是Linux系统服务间通信的标准机制：

```c
// 简化的D-Bus通信概念
// SystemD使用D-Bus管理服务
// 嵌入式中用于系统组件间消息传递

// 命令行测试D-Bus:
// dbus-send --system --dest=org.freedesktop.NetworkManager //   /org/freedesktop/NetworkManager //   org.freedesktop.DBus.Properties.Get //   string:"org.freedesktop.NetworkManager" string:"State"
```

### Q100: Linux输入子系统(input)？


> 🧠 **秒懂：** Linux input子系统统一处理键盘/鼠标/触摸屏/按键等输入设备。应用层读取/dev/input/eventN→解析struct input_event(type/code/value)。嵌入式中用于按键和触摸屏。

嵌入式设备的按键、触摸屏、传感器数据通过input子系统上报：

```c
// 关键系统调用示例(见各函数注释)
#include <linux/input.h>

int fd = open("/dev/input/event0", O_RDONLY);
struct input_event ev;

while (read(fd, &ev, sizeof(ev)) == sizeof(ev)) {
    if (ev.type == EV_KEY) {
        printf("Key %d %s\n", ev.code,
               ev.value ? "pressed" : "released");
    } else if (ev.type == EV_ABS) {
        printf("Abs axis %d value %d\n", ev.code, ev.value);
    }
}
```

### Q101: V4L2视频采集编程？

> 🧠 **秒懂：** V4L2(Video for Linux 2)是Linux视频采集接口：open设备→VIDIOC_QUERYCAP查能力→设格式/缓冲区→STREAMON开始采集→mmap读取帧→STREAMOFF停止。摄像头程序的标准框架。

Video4Linux2是Linux视频采集标准接口(摄像头)：

```c
#include <linux/videodev2.h>

int fd = open("/dev/video0", O_RDWR);

// 查询设备能力
struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);

// 设置格式
struct v4l2_format fmt;
fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
fmt.fmt.pix.width = 640;
fmt.fmt.pix.height = 480;
fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
ioctl(fd, VIDIOC_S_FMT, &fmt);

// 请求缓冲区(mmap方式)
struct v4l2_requestbuffers req = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
    .count = 4
};
ioctl(fd, VIDIOC_REQBUFS, &req);
// ... mmap缓冲区, 入队, 开始采集, 出队获取帧 ...
```

### Q102: ALSA音频编程？

> 🧠 **秒懂：** ALSA是Linux音频子系统：snd_pcm_open→设参数(采样率/位宽/通道)→snd_pcm_writei播放或snd_pcm_readi录音。嵌入式音频产品(语音交互/音响)的底层接口。

嵌入式音频播放/录制使用ALSA接口：

```c
#include <alsa/asoundlib.h>

snd_pcm_t *pcm;
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);

// 设置参数
snd_pcm_set_params(pcm,
    SND_PCM_FORMAT_S16_LE,    // 16位小端
    SND_PCM_ACCESS_RW_INTERLEAVED,
    2,                         // 双声道
    44100,                     // 采样率
    1,                         // 允许重采样
    100000);                   // 延迟100ms

// 播放PCM数据
short buffer[1024];
snd_pcm_writei(pcm, buffer, 512);  // 写512帧

snd_pcm_close(pcm);
```

### Q103: Linux电源管理(suspend/resume)？

> 🧠 **秒懂：** Linux电源管理：echo mem > /sys/power/state进入挂起。驱动需实现suspend/resume回调保存/恢复硬件状态。嵌入式设备省电的关键——不使用时挂起外设或整个系统。

嵌入式设备的低功耗管理：

```bash
# 查看支持的睡眠模式
cat /sys/power/state
# 输出: freeze mem disk

# 进入睡眠
echo mem > /sys/power/state    # 挂起到RAM(S3)
echo freeze > /sys/power/state  # 冻结(省电但不关外设)

# 设置唤醒源
echo enabled > /sys/devices/.../power/wakeup

# 用户空间电源管理(嵌入式应用)
# - 监控电池电量
# - 空闲时进入低功耗
# - GPIO中断唤醒
```

### Q104: syslog日志系统？

> 🧠 **秒懂：** syslog是Linux标准日志系统：openlog→syslog(priority, message)→closelog。日志写入/var/log/syslog。嵌入式中配置rsyslog远程发送日志到服务器便于集中分析。

嵌入式Linux使用syslog进行系统日志管理：

```c
#include <syslog.h>

// 打开日志
openlog("myapp", LOG_PID | LOG_CONS, LOG_DAEMON);

// 写日志(不同级别)
syslog(LOG_INFO, "Application started, version %s", VERSION);
syslog(LOG_WARNING, "Temperature high: %d C", temp);
syslog(LOG_ERR, "Failed to open device: %s", strerror(errno));

closelog();
```

```bash
# 查看日志
tail -f /var/log/syslog
journalctl -u myservice -f   # systemd系统
```

### Q105: 嵌入式文件系统操作(mount/umount)？

> 🧠 **秒懂：** mount -t type device dir挂载文件系统。嵌入式中开机脚本mount SD卡/NFS/tmpfs等。umount卸载前确保无进程使用(lsof/fuser检查)。remount可以改变挂载选项。

程序中动态挂载文件系统：

```c
#include <sys/mount.h>

// 挂载SD卡
mount("/dev/mmcblk0p1", "/mnt/sd", "vfat", 
      MS_NOEXEC | MS_NOSUID, "utf8");

// 挂载tmpfs(内存文件系统)
mount("tmpfs", "/var/run", "tmpfs", 0, "size=10M");

// 卸载
umount("/mnt/sd");

// 同步文件系统缓存到磁盘
sync();
```

### Q106: netlink与内核通信？

> 🧠 **秒懂：** Netlink是内核与用户空间的通信机制(socket接口)。用于获取/配置网络接口(NETLINK_ROUTE)、监听内核事件(udev用NETLINK_KOBJECT_UEVENT)。比ioctl更灵活。

Netlink是Linux用户态与内核的通信通道(网络事件、设备热插拔等)：

```c
#include <linux/netlink.h>

// 监听内核uevent(设备热插拔)
int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
struct sockaddr_nl sa = {
    .nl_family = AF_NETLINK,
    .nl_groups = 1  // 内核广播组
};
bind(fd, (struct sockaddr *)&sa, sizeof(sa));

char buf[4096];
while (1) {
    int len = recv(fd, buf, sizeof(buf), 0);
    printf("Event: %s\n", buf);
    // 典型输出: add@/devices/usb/... (USB设备插入)
}
```

### Q107: 定时器的使用(timer_create/setitimer)？

> 🧠 **秒懂：** POSIX定时器timer_create→timer_settime设置(一次性/重复)→到期发信号或通知线程。旧接口setitimer更简单但功能少。也可以用timerfd配合epoll(更现代)。

Linux下定时器的多种实现方式：

```c
// 方法1: POSIX定时器(推荐)
#include <signal.h>
#include <time.h>

void timer_handler(int sig, siginfo_t *si, void *uc) {
    printf("Timer expired!\n");
}

timer_t timerid;
struct sigevent sev = {
    .sigev_notify = SIGEV_SIGNAL,
    .sigev_signo = SIGRTMIN
};
timer_create(CLOCK_MONOTONIC, &sev, &timerid);

struct itimerspec its = {
    .it_value = { .tv_sec = 1 },     // 首次1秒后
    .it_interval = { .tv_nsec = 500000000 }  // 之后每500ms
};
timer_settime(timerid, 0, &its, NULL);

// 方法2: timerfd(可配合epoll，见Q69)
```

### Q108: 动态库的加载(dlopen/dlsym)？

> 🧠 **秒懂：** dlopen加载动态库→dlsym获取函数/变量地址→调用函数→dlclose关闭。实现插件机制：运行时根据配置加载不同模块。嵌入式中用于功能扩展和热更新。

运行时动态加载共享库(插件机制)：

```c
#include <dlfcn.h>

// 加载动态库
void *handle = dlopen("./libplugin.so", RTLD_LAZY);
if (!handle) {
    fprintf(stderr, "dlopen: %s\n", dlerror());
    return -1;
}

// 获取函数指针
typedef int (*PluginFunc)(const char *);
PluginFunc func = dlsym(handle, "plugin_process");
if (!func) {
    fprintf(stderr, "dlsym: %s\n", dlerror());
}

// 调用
func("hello plugin");

// 卸载
dlclose(handle);
```

### Q109: 进程间文件描述符传递(SCM_RIGHTS)？

> 🧠 **秒懂：** 通过Unix Domain Socket的辅助数据(ancillary data)传递fd：sendmsg发送SCM_RIGHTS消息→recvmsg接收。实现不同进程间共享文件描述符。用于进程间共享设备/连接。

通过Unix域套接字传递fd(高级IPC技巧)：

```c
// 发送fd
void send_fd(int sock, int fd_to_send) {
    struct msghdr msg = {0};
    struct cmsghdr *cmsg;
    char buf[CMSG_SPACE(sizeof(int))];
    
    msg.msg_control = buf;
    msg.msg_controllen = sizeof(buf);
    cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;
    cmsg->cmsg_len = CMSG_LEN(sizeof(int));
    *(int *)CMSG_DATA(cmsg) = fd_to_send;
    
    struct iovec iov = { .iov_base = "x", .iov_len = 1 };
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    sendmsg(sock, &msg, 0);
}
```

### Q110: Linux cron定时任务？


> 🧠 **秒懂：** crontab -e编辑定时任务：分/时/日/月/周 命令。嵌入式中用于定期日志清理、健康检查、数据上报。注意cron环境变量与交互式Shell不同。

嵌入式设备中的定时执行任务：

```bash
# crontab格式: 分 时 日 月 周 命令
# 每5分钟采集一次温度
*/5 * * * * /usr/bin/read_temp >> /var/log/temp.log

# 每天凌晨3点清理日志
0 3 * * * /usr/sbin/logrotate /etc/logrotate.conf

# 每小时同步时间
0 * * * * ntpdate pool.ntp.org

# 开机时执行
@reboot /usr/bin/myapp &
```

### Q111: select实现超时读取？

> 🧠 **秒懂：** fd_set集合→FD_SET添加fd→select(maxfd+1, &readfds, NULL, NULL, &timeout)→返回就绪fd数→FD_ISSET检查哪个fd就绪。超时返回0，错误返回-1。

在嵌入式中经常需要带超时的IO操作：

```c
#include <sys/select.h>

int read_timeout(int fd, char *buf, int len, int timeout_ms) {
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(fd, &fds);
    
    struct timeval tv;
    tv.tv_sec = timeout_ms / 1000;
    tv.tv_usec = (timeout_ms % 1000) * 1000;
    
    int ret = select(fd + 1, &fds, NULL, NULL, &tv);
    if (ret > 0 && FD_ISSET(fd, &fds)) {
        return read(fd, buf, len);
    } else if (ret == 0) {
        return 0;  // 超时
    }
    return -1;  // 错误
}
```

### Q112: 嵌入式Linux内存优化？

> 🧠 **秒懂：** 嵌入式Linux内存优化：精简内核配置(make menuconfig)→使用busybox(替代完整命令)→静态链接减少库→strip去符号→调小栈大小→使用内存池→避免内存泄漏。

嵌入式设备内存资源有限，需要优化：

```bash
# 查看内存使用
free -m
cat /proc/meminfo

# 清除缓存(释放页缓存)
echo 3 > /proc/sys/vm/drop_caches

# 关闭swap(嵌入式通常不用swap)
swapoff -a

# 使用busybox替代完整工具链(节省空间)
# 使用musl-libc替代glibc(更小)
# strip去除调试信息
arm-linux-gnueabihf-strip myapp
```

```c
// 代码级优化
// 1. 避免内存碎片：使用内存池
// 2. 减少堆分配：尽量栈上分配或静态分配
// 3. 使用mmap映射大文件(按需加载)
// 4. 共享库减少内存占用(多进程共享代码段)
```

### Q113: 嵌入式设备热插拔处理(udev)？

> 🧠 **秒懂：** udev是Linux设备管理器：设备插拔时内核发送uevent→udev规则匹配→自动创建/删除设备节点、加载驱动、执行脚本。嵌入式中用于热插拔USB/SD卡的自动处理。

处理USB/SD卡等设备的动态插拔：

```bash
# udev规则示例 /etc/udev/rules.d/99-mydevice.rules
# USB设备插入时执行脚本
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="1234", \
  RUN+="/usr/bin/handle_usb.sh"

# 串口设备固定名称
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", SYMLINK+="my_serial"
```

```c
// 程序中监听udev事件
#include <libudev.h>
struct udev *udev = udev_new();
struct udev_monitor *mon = udev_monitor_new_from_netlink(udev, "udev");
udev_monitor_filter_add_match_subsystem_devtype(mon, "usb", NULL);
udev_monitor_enable_receiving(mon);
int fd = udev_monitor_get_fd(mon);
// 配合epoll等待事件...
```

### Q114: Linux时间相关API？

> 🧠 **秒懂：** time(秒级)、gettimeofday(微秒级)、clock_gettime(纳秒级，CLOCK_MONOTONIC不受NTP影响)。嵌入式中测量时间间隔用CLOCK_MONOTONIC，墙钟时间用CLOCK_REALTIME。

嵌入式系统中时间处理的各种方式：

```c
#include <time.h>

// 1. 系统时间(日历时间)
time_t now = time(NULL);
struct tm *t = localtime(&now);
printf("%04d-%02d-%02d %02d:%02d:%02d\n",
       t->tm_year+1900, t->tm_mon+1, t->tm_mday,
       t->tm_hour, t->tm_min, t->tm_sec);

// 2. 单调时钟(不受NTP/手动调时影响，适合计时)
struct timespec ts;
clock_gettime(CLOCK_MONOTONIC, &ts);

// 3. 高精度计时
struct timespec start, end;
clock_gettime(CLOCK_MONOTONIC, &start);
do_work();
clock_gettime(CLOCK_MONOTONIC, &end);
long elapsed_ns = (end.tv_sec - start.tv_sec) * 1000000000L
                + (end.tv_nsec - start.tv_nsec);

// 4. 睡眠
usleep(1000);        // 微秒(已废弃)
nanosleep(&ts, NULL); // 纳秒(推荐)
clock_nanosleep(CLOCK_MONOTONIC, 0, &ts, NULL); // 指定时钟源
```

### Q115: 嵌入式网络配置编程？

> 🧠 **秒懂：** 编程方式配置网络：ioctl(SIOCSIFADDR)设IP、socket路由表操作、system('ifconfig')/system('ip')调用命令。现代方式用Netlink(RTNETLINK)操作更规范。

程序中配置网络接口：

```c
#include <net/if.h>
#include <sys/ioctl.h>

// 获取IP地址
int fd = socket(AF_INET, SOCK_DGRAM, 0);
struct ifreq ifr;
strncpy(ifr.ifr_name, "eth0", IFNAMSIZ);
ioctl(fd, SIOCGIFADDR, &ifr);
struct sockaddr_in *addr = (struct sockaddr_in *)&ifr.ifr_addr;
printf("IP: %s\n", inet_ntoa(addr->sin_addr));

// 设置IP地址(需root)
addr->sin_addr.s_addr = inet_addr("192.168.1.100");
ioctl(fd, SIOCSIFADDR, &ifr);
close(fd);
```

### Q116: Linux下多线程性能调优？

> 🧠 **秒懂：** 减少锁竞争(细粒度锁/读写锁/无锁)、降低伪共享(Cache Line对齐)、合理线程数(不超过CPU核数)、CPU亲和性绑核、减少上下文切换。用perf/valgrind分析瓶颈。

嵌入式Linux多线程程序性能优化：

```c
// 1. 减少锁竞争
// - 缩小临界区
// - 读写锁(读多写少)
// - 无锁数据结构(环形队列)
// - 分段锁(将一个大锁拆成多个小锁)

// 2. 避免false sharing(伪共享)
struct __attribute__((aligned(64))) ThreadData {
    int counter;    // 每个线程独立, 各占一个缓存行
    char pad[60];
};

// 3. 使用thread pool(避免频繁创建/销毁)

// 4. 绑定CPU(减少迁移开销)
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(core_id, &cpuset);
pthread_setaffinity_np(tid, sizeof(cpuset), &cpuset);
```

### Q117: Linux安全编程实践？

> 🧠 **秒懂：** 输入验证(防注入)、最小权限原则、不用system/popen(Shell注入)、检查返回值、避免缓冲区溢出(用snprintf不用sprintf)、使用安全函数、编译启用安全选项(-fstack-protector)。

嵌入式设备安全编程的基本原则：

```c
// 1. 输入验证
int process_cmd(const char *input, size_t len) {
    if (len > MAX_CMD_LEN) return -1;  // 长度检查
    if (!is_valid_chars(input)) return -1;  // 字符白名单
}

// 2. 缓冲区安全
char buf[64];
snprintf(buf, sizeof(buf), "%s", input);  // 不用sprintf

// 3. 整数溢出检查
if (a > 0 && b > 0 && a > INT_MAX - b) {
    // 溢出!
}

// 4. 文件操作安全
// 检查文件权限、使用O_NOFOLLOW防止符号链接攻击
int fd = open(path, O_RDONLY | O_NOFOLLOW);

// 5. 权限最小化
// 完成需要root的操作后立即降权
setuid(unprivileged_uid);
```

### Q118: 嵌入式Linux性能分析工具？

> 🧠 **秒懂：** perf(CPU热点分析)、strace(系统调用跟踪)、valgrind(内存检测)、gprof(函数耗时)、ftrace(内核函数跟踪)、top/htop(资源监控)。嵌入式性能优化先分析后优化。

性能分析是优化的基础：

```bash
# perf - Linux性能分析首选
perf stat ./myapp            # 统计CPU事件
perf record -g ./myapp       # 记录采样(含调用栈)
perf report                  # 分析采样结果

# top/htop - 实时资源监控
top -H -p <pid>              # 查看线程级CPU使用

# strace -c - 系统调用统计
strace -c ./myapp

# valgrind - 内存分析
valgrind --tool=massif ./myapp   # 内存使用分析
valgrind --tool=callgrind ./myapp  # 函数调用分析

# time - 简单计时
time ./myapp
# real(总时间) user(用户态CPU) sys(内核态CPU)
```

### Q119: Linux线程与进程的选择？

> 🧠 **秒懂：** 多进程：隔离性好(一个崩溃不影响其他)、在多CPU上并行好。多线程：共享数据方便、创建开销小。嵌入式选择：稳定性要求高用多进程，通信频繁用多线程。

面试常问"什么时候用多线程，什么时候用多进程"：

| 场景 | 推荐 | 原因 |
|------|------|------|
| 共享大量数据 | 多线程 | 线程天然共享地址空间 |
| 需要隔离(一个崩不影响另一个) | 多进程 | 地址空间独立 |
| 高并发(>1000) | 多线程/协程 | 进程创建开销大 |
| 利用多核 | 都行 | 都能并行 |
| 嵌入式实时 | 多线程 | 切换快、共享方便 |

### Q120: 嵌入式配置文件解析(JSON/INI)？


> 🧠 **秒懂：** JSON解析：cJSON(轻量C库)、json-c。INI解析：iniparser或手写(逐行读取、按=分割)。嵌入式中配置文件用JSON(结构化好)或INI(简单直观)。注意文件不存在时的默认值处理。

嵌入式设备配置方案：

```c
// 简易INI文件解析
int parse_ini(const char *path, const char *key, char *value, int len) {
    FILE *fp = fopen(path, "r");
    if (!fp) return -1;
    
    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        // 跳过注释和空行
        if (line[0] == '#' || line[0] == '\n') continue;
        
        char *eq = strchr(line, '=');
        if (!eq) continue;
        *eq = '\0';
        
        // 去除空格后比较键名
        char *k = line, *v = eq + 1;
        while (*k == ' ') k++;
        while (*v == ' ') v++;
        v[strcspn(v, "\r\n")] = '\0';
        
        if (strcmp(k, key) == 0) {
            strncpy(value, v, len - 1);
            value[len - 1] = '\0';
            fclose(fp);
            return 0;
        }
    }
    fclose(fp);
    return -1;
}
```

### Q121: Linux进程内存泄漏检测？

> 🧠 **秒懂：** Valgrind(最全面但慢5-10倍)、AddressSanitizer(编译时插桩，开销小)、mtrace(glibc内置)、/proc/PID/smaps监控内存增长。嵌入式中长期运行监控RSS增长趋势。

嵌入式长运行程序的内存泄漏排查：

```bash
# 方法1: 定期采样/proc/<pid>/status
watch -n 5 "grep VmRSS /proc/<pid>/status"

# 方法2: Valgrind(开发阶段)
valgrind --leak-check=full --show-reachable=yes ./myapp

# 方法3: mtrace(glibc内置)
# 代码中:
#include <mcheck.h>
mtrace();  // 开始跟踪
// ... 程序运行 ...
muntrace(); // 结束跟踪
# 设置环境变量: MALLOC_TRACE=trace.log ./myapp
# 分析: mtrace ./myapp trace.log
```

### Q122: 多线程程序中的错误处理？

> 🧠 **秒懂：** 多线程错误处理：返回错误码(不用errno，因为errno是线程本地的)、不在持锁期间调用可能失败的函数、使用RAII确保锁被释放、日志记录线程ID便于调试。

多线程错误处理与单线程不同：

```c
// 1. errno是线程安全的(每个线程有独立errno)
// 但strerror()不是! 用strerror_r()

char errbuf[128];
strerror_r(errno, errbuf, sizeof(errbuf));  // 线程安全

// 2. 线程内设置退出码
void *thread_func(void *arg) {
    int *result = malloc(sizeof(int));
    *result = do_work();
    if (*result < 0) {
        *result = -1;
        return result;
    }
    return result;
}

// 3. 全局错误状态需要保护
pthread_mutex_lock(&err_lock);
g_last_error = error_code;
pthread_mutex_unlock(&err_lock);
```

### Q123: Linux能力(capabilities)机制？

> 🧠 **秒懂：** capabilities将root权限细分为多个独立能力(如CAP_NET_RAW网络原始权限、CAP_SYS_BOOT重启)。进程只授予需要的能力而非full root。嵌入式中减少安全攻击面。

精细化权限控制(替代root全有或全无)：

```bash
# 查看程序capabilities
getcap /usr/bin/ping
# 输出: /usr/bin/ping = cap_net_raw+ep (可以发原始包,不需要root)

# 设置capabilities
setcap cap_net_bind_service+ep myapp  # 允许绑定<1024端口

# 代码中降低权限
#include <sys/capability.h>
cap_t caps = cap_get_proc();
cap_clear(caps);
cap_set_flag(caps, CAP_PERMITTED, 1, &cap_net_raw, CAP_SET);
cap_set_proc(caps);
cap_free(caps);
```

### Q124: 嵌入式OTA升级(Linux层)？

> 🧠 **秒懂：** 嵌入式Linux OTA：A/B双分区→下载新固件到备用分区→校验(CRC/签名)→修改启动标志→重启进入新分区→启动成功确认→失败自动回退。SWUpdate/RAUC是成熟的OTA框架。

Linux设备远程升级实现：

```bash
# 典型方案: SWUpdate
# 1. 双rootfs分区(A/B切换)
# 2. 下载更新包(.swu)
# 3. 校验签名
# 4. 写入非活动分区
# 5. 切换启动标志(如U-Boot环境变量)
# 6. 重启验证

# 升级脚本示例
#!/bin/bash
CURRENT=$(cat /proc/cmdline | grep -o "root=[^ ]*" | cut -d= -f2)
if [ "$CURRENT" = "/dev/mmcblk0p2" ]; then
    TARGET="/dev/mmcblk0p3"
else
    TARGET="/dev/mmcblk0p2"
fi

dd if=rootfs.img of=$TARGET bs=4M
fw_setenv bootpart $TARGET
reboot
```

### Q125: 嵌入式系统安全加固？


> 🧠 **秒懂：** 安全加固：最小化系统(去掉不需要的服务和命令)→安全启动(Secure Boot验证签名)→文件系统只读(squashfs)→SELinux/AppArmor访问控制→加密通信(TLS)→禁用调试接口。

嵌入式Linux产品安全加固措施：

```bash
# 1. 关闭不需要的服务
systemctl disable telnet ssh(生产环境)

# 2. 文件系统只读
mount -o remount,ro /

# 3. 限制root登录
# /etc/securetty 为空

# 4. 内核加固
echo 1 > /proc/sys/kernel/randomize_va_space  # ASLR
echo 0 > /proc/sys/kernel/kptr_restrict        # 隐藏内核地址

# 5. 安全启动链
# BootROM验证U-Boot签名 → U-Boot验证内核签名 → 内核验证rootfs

# 6. 应用层
# - 最小权限原则(不以root运行)
# - 输入验证
# - 加密敏感数据
# - 安全通信(TLS)
```

---

> 本文档由公众号：**嵌入式校招菌** 原创整理，欢迎关注获取更多嵌入式校招信息
>
> 📢 **嵌入式校招excel** 全年更新中，实习/秋招/春招都包含，纯人工维护，已支持24/25/26届同学毕业，减少招聘信息差，你没听过的公司只要有嵌入式岗位我都会收集，一次付费永久有效，更有嵌入式微信/飞书交流群；用过的同学都说好！
>
> 需要的可以关注公众号：**嵌入式校招菌**，对话框回复：**加群**


---

## ★ 面经高频补充题（来源：GitHub面经仓库/牛客讨论区/大厂真题整理）

### Q126: 多线程编程中如何避免死锁？

> 🧠 **秒懂：** ①固定加锁顺序(所有线程按相同顺序获取锁) ②使用pthread_mutex_trylock(失败后释放已有锁重试) ③减少锁的持有时间和范围。

> 💡 **面试高频** | 牛客面经高频 | 追问"手写死锁检测"是加分项

**死锁四个必要条件（缺一不死锁）：**
1. 互斥: 资源独占
2. 持有并等待: 持有A等B
3. 不可剥夺: 不能强制抢
4. 循环等待: A等B, B等A

**嵌入式避免死锁的实践：**

```c
// 方法1: 固定加锁顺序(打破循环等待)
// 规则: 永远先锁mutex_A再锁mutex_B
pthread_mutex_lock(&mutex_A);  // 所有线程都先锁A
pthread_mutex_lock(&mutex_B);  // 再锁B
// ... 临界区 ...
pthread_mutex_unlock(&mutex_B);
pthread_mutex_unlock(&mutex_A);

// 方法2: 超时锁(trylock)
if (pthread_mutex_trylock(&mutex_B) != 0) {
    pthread_mutex_unlock(&mutex_A);  // 拿不到B就释放A
    usleep(1000);  // 退避重试
    continue;
}

// 方法3: 减少锁粒度(用多个小锁代替大锁)
```

---

### Q127: fork后子进程和父进程的文件描述符关系？

> 🧠 **秒懂：** fork后子进程继承父进程打开的文件描述符副本(指向相同的file结构)。父子进程共享文件偏移量。关闭一方的fd不影响另一方。管道通信就利用了这一特性。

> 💡 **面试高频** | 牛客面经常见 | 追问"多进程写同一文件"

```text
fork之前:
  父进程 fd=3 → [文件表项: offset=100, flags] → [inode]

fork之后:
  父进程 fd=3 ──┐
                 ├→ [同一文件表项: offset=100] → [inode]
  子进程 fd=3 ──┘

关键点:
  1. 父子进程共享同一文件表项(共享偏移量!)
  2. 父进程写入后offset变化,子进程也看到(因为共享)
  3. 这就是为什么重定向在fork后对两个进程都生效
  4. 如果子进程exec了,fd默认继承(除非FD_CLOEXEC)
```

> 💡 **面试追问：** "两个进程分别open同一文件呢？" → 各有独立的文件表项(不同offset),互不影响。

---

### Q128: 静态库和动态库的区别？

> 🧠 **秒懂：** 静态库在链接时全部打包进可执行文件，动态库运行时加载。静态库用ar打包(.a)，动态库用gcc -shared编译(.so)。嵌入式小系统多用静态链接。

> 💡 **面试高频** | 编译链接必考 | 牛客面经高频

| 对比项 | 静态库(.a) | 动态库(.so) |
|--------|-----------|-------------|
| 链接时机 | 编译时复制进可执行文件 | 运行时动态加载 |
| 文件大小 | 大(库代码打包进去) | 小(只存引用) |
| 内存占用 | 每个进程一份副本 | 多进程共享一份 |
| 更新 | 要重新编译 | 替换.so即可 |
| 依赖 | 无外部依赖 | 依赖.so存在 |
| 嵌入式选择 | **MCU多用★**(简单可控) | Linux应用多用 |

```bash
# 创建静态库
gcc -c mylib.c -o mylib.o
ar rcs libmy.a mylib.o
gcc main.c -L. -lmy -o main  # 链接

# 创建动态库
gcc -fPIC -shared mylib.c -o libmy.so
gcc main.c -L. -lmy -o main
export LD_LIBRARY_PATH=.  # 运行时要找到.so
```

---

### Q129: 如何实现一个线程安全的单例模式？

> 🧠 **秒懂：** C++11 Meyers' Singleton(局部static变量)最简洁安全。C语言用pthread_once保证初始化只执行一次。双检锁(DCLP)需要配合内存屏障/atomic。

> 💡 **面试高频** | C/C++设计模式题 | 牛客面经常见

```c
// C语言实现线程安全单例(pthread_once)
#include <pthread.h>

typedef struct {
    int data;
    // ... 其他成员
} Singleton;

static Singleton *instance = NULL;
static pthread_once_t once_control = PTHREAD_ONCE_INIT;

static void init_singleton(void) {
    instance = (Singleton*)malloc(sizeof(Singleton));
    instance->data = 0;
}

Singleton* get_instance(void) {
    pthread_once(&once_control, init_singleton);
    return instance;
}
```

---

### Q130: 动态加载和静态加载的区别？与动态库/静态库有什么关系？

> 🧠 **秒懂：** 静态加载：链接时确定库的位置(编译期绑定)。动态加载：运行时dlopen按需加载(运行期绑定)。静态库只能静态加载，动态库两种都支持。dlopen实现的插件机制最灵活。

**答：**

**三种加载方式对比：**

| 方式 | 静态加载(Static Loading) | 动态加载-隐式(Implicit) | 动态加载-显式(Explicit) |
|------|------------------------|----------------------|----------------------|
| 对应库 | 静态库(.a) | 动态库(.so) | 动态库(.so) |
| 链接时机 | **编译时**链接器复制代码 | **启动时**动态链接器加载 | **运行时**程序主动dlopen |
| 链接命令 | `gcc -static -lxxx` | `gcc -lxxx`(默认) | 编译时不链接,代码中dlopen |
| 可执行文件 | 大(包含库代码) | 小(只存符号引用) | 小 |
| 运行依赖 | 无(自包含) | 要求.so在LD_LIBRARY_PATH | 要求.so运行时可找到 |
| 更新库 | 重新编译 | 替换.so重启程序 | 替换.so,无需重启 |
| 典型场景 | 嵌入式MCU/容器部署 | Linux应用程序默认方式 | **插件系统/热更新** |

**完整代码示例：**

```c
// ===== 1. 静态加载 =====
// 编译: gcc main.c -L. -lmath_static -static -o app
// 库代码在编译时已嵌入app, 运行时无需任何.so
#include "mymath.h"
int main() {
    int r = add(1, 2);  // 直接调用,函数地址编译时确定
    return 0;
}

// ===== 2. 动态加载-隐式 =====
// 编译: gcc main.c -L. -lmath -o app
// 运行: LD_LIBRARY_PATH=. ./app
// 启动时ld-linux.so自动加载libmath.so
#include "mymath.h"
int main() {
    int r = add(1, 2);  // 通过PLT/GOT间接调用(运行时解析真实地址)
    return 0;
}

// ===== 3. 动态加载-显式(dlopen) =====
// 编译: gcc main.c -ldl -o app  (注意:不需要-lmath)
#include <dlfcn.h>
int main() {
    // 运行时手动加载
    void *handle = dlopen("./libmath.so", RTLD_LAZY);
    if (!handle) { fprintf(stderr, "%s\n", dlerror()); return 1; }

    // 通过名字查找函数地址
    typedef int (*add_func)(int, int);
    add_func add = (add_func)dlsym(handle, "add");

    int r = add(1, 2);   // 通过函数指针调用

    dlclose(handle);      // 卸载
    return 0;
}
```

**ELF可执行文件中的区别：**

```bash
# 静态链接: 无动态段
$ file app_static
app_static: ELF 64-bit, statically linked

# 动态链接: 依赖.so
$ file app_dynamic
app_dynamic: ELF 64-bit, dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2

$ ldd app_dynamic
    libmath.so => ./libmath.so
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
    /lib64/ld-linux-x86-64.so.2
```

**PLT/GOT延迟绑定(隐式动态加载的核心)：**

```text
程序调用add()
    │
    ▼
PLT[add]  (Procedure Linkage Table,程序链接表)
    │  第一次调用 → 跳到动态链接器解析真实地址,写入GOT
    │  后续调用 → 直接从GOT读地址跳转(已缓存)
    ▼
GOT[add]  (Global Offset Table,全局偏移表)
    │  存放add()在.so中的真实虚拟地址
    ▼
libmath.so中的add()真实代码
```

> 💡 **面试追问：** 动态库的显式加载(dlopen)在嵌入式Linux中的实际应用场景？→ ①插件系统(加载不同传感器驱动.so)；②OTA热更新(替换.so不重启主程序)；③减少启动内存(按需加载功能模块)；④A/B版本切换(dlopen不同版本的算法.so)。


---

> 本文档由公众号：**嵌入式校招菌** 原创整理，欢迎关注获取更多嵌入式校招信息
>
> 📢 **嵌入式校招excel** 全年更新中，实习/秋招/春招都包含，纯人工维护，已支持24/25/26届同学毕业，减少招聘信息差，你没听过的公司只要有嵌入式岗位我都会收集，一次付费永久有效，更有嵌入式微信/飞书交流群；用过的同学都说好！
>
> 需要的可以关注公众号：**嵌入式校招菌**，对话框回复：**加群**
