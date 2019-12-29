# PINTOS 实验报告

> 10185101210 陈俊潼, East China Normal University

## 准备工作

### 安装与调试

为了方便使用 VSCode 做实验，避免安装一个繁重 Ubuntu 虚拟机，便尝试直接在 macOS 上安装 pintos。使用的 pintos 来自：

[https://github.com/maojie/pintos_mac](https://github.com/maojie/pintos_mac)

下载后使用以下 port 命令安装依赖库、gdb和 bochs：

```bash
sudo port install i386-elf-binutils
sudo port install i386-elf-gcc
sudo port install sdl
sudo port install gdb # 用于调试，安装后需要使用命令 ggdb 调试
sudo port install bochs -smp +gdbstub
```

其中 `sudo port install bochs -smp +gdbstub` 后面的两个参数是为了开启 gdb 调试。因为 port 默认安装的 pintos 没有 --enable-gdb-stub 参数（可以通过查阅 port 的 variant 得到，如下图）。

![](Report.assets/image-20191205183338082.png)

为了能够直接输入 gdb 运行 ggdb，可以`vim ~/.bash_profile`，加入一行 `alias gdb='ggdb';`。

接着讲将 pintos 放入任意目录，在终端中将 utils 目录 export PATH:

```bash
# 后面的目录是 utils 所在的目录
export PATH=$PATH:~/OneDrive/Workspace/LearningRepo/Course/OSConcepts/pintos/utils
```

进入 threads 目录运行 make 后，可以尝试运行：

![](Report.assets/2019-12-05-09-44-33.png)

发现无法找到内核。修改 kernal.o 和 loader.o 的位置。在 /utils/pintos 的第 256 行：

![](Report.assets/2019-12-05-09-37-02.png)

/utils/pintos.pm 第 362 行：

![](Report.assets/2019-12-05-10-19-58.png)

/utils/pintos-gdb 第 4 行，调整GDBMACROS的目录：

![](Report.assets/2019-12-05-10-26-06.png)

接着测试调试。

输入命令

```bash
sudo pintos --gdb -s -- run alarm-multiple
```

新建终端，进入 threads/build 目录下输入命令：

```bash
ggdb kernel.o # 使用 port 安装的 mac 下的 gdb 应输入 ggdb
target remote localhost:1234
```

回到 Bochs 运行界面，可以发现已经连接成功：

![image-20191205120711836](Report.assets/image-20191205120711836.png)

这时已经可以在 gdb 中输入命令 `make check`查看检查信息，得到以下反馈：

```
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x00000000 in ?? ()
(gdb) make check
pass tests/threads/alarm-single
pass tests/threads/alarm-multiple
pass tests/threads/alarm-simultaneous
FAIL tests/threads/alarm-priority
pass tests/threads/alarm-zero
pass tests/threads/alarm-negative
FAIL tests/threads/priority-change
FAIL tests/threads/priority-donate-one
FAIL tests/threads/priority-donate-multiple
FAIL tests/threads/priority-donate-multiple2
FAIL tests/threads/priority-donate-nest
FAIL tests/threads/priority-donate-sema
FAIL tests/threads/priority-donate-lower
FAIL tests/threads/priority-fifo
FAIL tests/threads/priority-preempt
FAIL tests/threads/priority-sema
FAIL tests/threads/priority-condvar
FAIL tests/threads/priority-donate-chain
FAIL tests/threads/mlfqs-load-1
FAIL tests/threads/mlfqs-load-60
FAIL tests/threads/mlfqs-load-avg
FAIL tests/threads/mlfqs-recent-1
pass tests/threads/mlfqs-fair-2
pass tests/threads/mlfqs-fair-20
FAIL tests/threads/mlfqs-nice-2
FAIL tests/threads/mlfqs-nice-10
FAIL tests/threads/mlfqs-block
20 of 27 tests failed.
make: *** [check] Error 1
```

确认环境配置完成，实验正式开始。

在 pintos 运行时，会在 build 目录下创建 bochsrc.txt 文件，用于给 bochs 虚拟机提供配置文件。运行时的输出都会输出在终端窗口中。可以使用 intos run alar0m-multiple > log 将输出重定向到文本文件保存。

### 代码规范

官方文档推荐在开始实现之前，阅读附录中的代码规范。指出项目应当遵循 `GNU Coding Standards`：

![image-20191218234232505](Report.assets/image-20191218234232505.png)

规范指出：

- 不应当使任意一行代码超过 79 个字符
- 支持 C99 标准库中的新特性
- 应当为每一个函数写明注释
- 不要使用 `strcpy()`、`strcat()`、`sprintf()`等不安全的函数

## Project 1

### Overview

在开始之前，初步了解 PintOS 目录下的几个文件夹的内容：

```
threads/：内核的源代码
userprog/：用户程序加载代码
vm/：虚拟内存目录
filesys/：文件系统目录
devics/：I/O 设备驱动目录
lib/：包含部分标准 C 语言的函数
lib/kernel：部分只在 Pintos 中有的 C 语言函数
lib/user：包含一些头文件，只在 Pintos 中有的一些 C 语言函数
tests/：每个 Project 的测试案例
examples/：在 Projcet 2 的一些案例
misc/ & utils/：官方不推荐修改的两个文件夹
```

而浏览 Pintos 的官方文档可以了解到 thread 目录下具有的文件包含以下功能：

//todo

除此之外，浏览官方文档的附录 Debugging Tools，了解到两个常用的调试工具的用法。第一个是`ASSERT`，官方的描述如下：

![image-20191218232404670](Report.assets/image-20191218232404670.png)

`ASSERT`的作用是测试括号内的表达式，如果表达式不为真，这会出现 kernel panic，将出现错误的详细信息打印在屏幕上。

第二个调试工具是 `printf()`，使用方法同 C 标准库函数。

### Mission 1: Alarm Clock (忙等待问题)

#### Requirement

在这个任务中需要重新实现 devices/timer/c 目录下的 timer_sleep()。虽然当前代码提供了一个实现方式，但它的实现方为忙等待，即它在循环中检查当前时间是否已经过去`ticks`个时钟，并循环调用`thread_yield()`直到循环结束。**需要重新实现这个函数来避免忙等待。**

对于 `timer_msleep()`、`timer_usleep()`、`timer_nsleep()`等函数，将会自动定期调用`timer_sleep()`，所有不需要修改。

#### Analysis

这个函数将会在 pintos 原先的实现中， timer_sleep() 的代码如下：

```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks) 
{
  int64_t start = timer_ticks ();

  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks) 
    thread_yield ();
}
```

1. 首先，`start` 记录了进入这个函数的当前时间。
2. 然后判断 `intr_get_level()`的返回值是否为真，即是否启用了中断，如果没有则 kernel panic。
3. 利用 `timer_elapsed ()` 判断当前经过的时间和`start`之间的差值。
4. 判断当前流逝的时间是否超过给定的 `ticks`，如果没有，执行 `thread_yield ()`，让线程休眠。

接下来查找`thread_yield()`函数。这个函数位于threads/threads.c：

```c
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void
thread_yield (void) 
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;
  
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread) 
    list_push_back (&ready_list, &cur->elem);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
```

1. 首先通过 `thread_current()`获得当前正在运行的线程。这个结构体指针中的进程结构体包含以下字段：

   ```c
   struct thread
     {
       /* Owned by thread.c. */
       tid_t tid;                          /* Thread identifier. */
       enum thread_status status;          /* Thread state. */
       char name[16];                      /* Name (for debugging purposes). */
       uint8_t *stack;                     /* Saved stack pointer. */
       int priority;                       /* Priority. */
       struct list_elem allelem;           /* List element for all threads list. */
   
       /* Shared between thread.c and synch.c. */
       struct list_elem elem;              /* List element. */
   
   #ifdef USERPROG
       /* Owned by userprog/process.c. */
       uint32_t *pagedir;                  /* Page directory. */
   #endif
   
       /* Owned by thread.c. */
       unsigned magic;                     /* Detects stack overflow. */
     };
   ```

   包含有进程 ID，进程状态，进程名，栈指针，优先级，线程链表，线程项，页目录，和一个用于检查栈溢出的量。而其中的 `thread_status`又包含有以下几种进程状态：

   ```C
   /* States in a thread's life cycle. */
   enum thread_status
     {
       THREAD_RUNNING,     /* Running thread. */
       THREAD_READY,       /* Not running but ready to run. */
       THREAD_BLOCKED,     /* Waiting for an event to trigger. */
       THREAD_DYING        /* About to be destroyed. */
     };
   ```

   这些文件都通过 thread.c 实现。

2. 获取`old_level`，即调用函数的时候的中断状态。同时发现在 `intr_disable()`函数中会暂时禁用中断。

3. 判断当前的进程是否为 idle_thread，如果不是，则将现在这个进程 push 到 ready_list 的尾部。

4. 调度当前进程，同时将中断状态设置回刚调用`thread_yield()`时的状态。

回顾 `timer_sleep()`函数，可以发现这个函数是一个自旋锁，将会一直循环检查当前经过的时间并且调用 `thread_yield()`来让进程休眠，会出现忙等待的现象。也就是为了让进程休眠特定时间，这个函数就在这一段时间里反复把进程从运行状态丢到就绪列表的最后。当调度到来时，如果时间没到，又一次把进程放在最后，这样做效率低下。

观察到 `thread_block()`函数和`thread_unblock`函数，观察其注释：

```c
/* Puts the current thread to sleep.  It will not be scheduled
   again until awoken by thread_unblock().

   This function must be called with interrupts turned off.  It
   is usually a better idea to use one of the synchronization
   primitives in synch.h. */
void
thread_block (void) 
{
  ASSERT (!intr_context ());
  ASSERT (intr_get_level () == INTR_OFF);

  thread_current ()->status = THREAD_BLOCKED;
  schedule ();
}
```

可知如果一个进程被设置为阻塞状态后，在调用`thread_unblock（）`之前将不再会被 `schedule()`函数调度，而这正是我们需要的。

#### Solution

为了解决这个问题，可以在进程第一次调用`timer_sleep()`的时候，通过调用 `thread_block()`函数来讲进程的状态设置为阻塞，同时记录下需要阻塞的时间，至此，`timer_sleep()`就完成了自己的工作，把唤醒的工作留给其他函数。

这样的话，当在每一次中断，即调用`timer_interrupt()`的时候，都需要把所有被阻塞的进程内记录的阻塞时间信息 -1，如果减到了 0，则 unblock 进程，供后续调度。

所以需要对代码进行以下修改：

1. 在 thread 的结构体，也就是 PCB 中加入一项 `blocked_ticks`:

   ![image-20191219020458585](Report.assets/image-20191219020458585.png)

2. 在初始化进程调用`thread_create()`的时候，将 `blocked_ticks` 设置为 0：

   ![image-20191219021430365](Report.assets/image-20191219021430365.png)

3. 新建一个 `thread_check_blocked(struct thread *t)`函数检查进程的阻塞时间记录情况。之所以需要增加第二个 `void *aux`指针的原因是这个韩珊瑚将会被`thread_foreach`调用，而这个函数将会给`thread_chheck_blocked`传递一个`aux`指针。

   ![image-20191219025841037](Report.assets/image-20191219025841037.png)

   同时在`thread.h`头文件中添加：

   ```c
   void thread_check_blocked(struct thread *, void * aux UNUSED);
   ```

4. 在 `timer_interrupt()`调用的时候，对每一个进程都使用 `thread_for_each()`函数调用`thread_check_blocked()`来处理阻塞状态：

   ![image-20191219024713208](Report.assets/image-20191219024713208.png)

5. 最后修改改`timer_sleep()`函数，使进程通过阻塞休眠而不是忙等待：

   ![image-20191219030029133](Report.assets/image-20191219030029133.png)

#### Result

使用命令`pintos -- run alarm-multiple`检查运行结果，可以得到：

```
(alarm-multiple) begin
(alarm-multiple) Creating 5 threads to sleep 7 times each.
(alarm-multiple) Thread 0 sleeps 10 ticks each time,
(alarm-multiple) thread 1 sleeps 20 ticks each time, and so on.
(alarm-multiple) If successful, product of iteration count and
(alarm-multiple) sleep duration will appear in nondescending order.
(alarm-multiple) thread 0: duration=10, iteration=1, product=10
(alarm-multiple) thread 0: duration=10, iteration=2, product=20
(alarm-multiple) thread 1: duration=20, iteration=1, product=20
(alarm-multiple) thread 0: duration=10, iteration=3, product=30
(alarm-multiple) thread 2: duration=30, iteration=1, product=30
(alarm-multiple) thread 0: duration=10, iteration=4, product=40
(alarm-multiple) thread 1: duration=20, iteration=2, product=40
(alarm-multiple) thread 3: duration=40, iteration=1, product=40
(alarm-multiple) thread 0: duration=10, iteration=5, product=50
(alarm-multiple) thread 4: duration=50, iteration=1, product=50
(alarm-multiple) thread 0: duration=10, iteration=6, product=60
(alarm-multiple) thread 1: duration=20, iteration=3, product=60
(alarm-multiple) thread 2: duration=30, iteration=2, product=60
(alarm-multiple) thread 0: duration=10, iteration=7, product=70
(alarm-multiple) thread 1: duration=20, iteration=4, product=80
(alarm-multiple) thread 3: duration=40, iteration=2, product=80
(alarm-multiple) thread 2: duration=30, iteration=3, product=90
(alarm-multiple) thread 1: duration=20, iteration=5, product=100
(alarm-multiple) thread 4: duration=50, iteration=2, product=100
(alarm-multiple) thread 1: duration=20, iteration=6, product=120
(alarm-multiple) thread 2: duration=30, iteration=4, product=120
(alarm-multiple) thread 3: duration=40, iteration=3, product=120
(alarm-multiple) thread 1: duration=20, iteration=7, product=140
(alarm-multiple) thread 2: duration=30, iteration=5, product=150
(alarm-multiple) thread 4: duration=50, iteration=3, product=150
(alarm-multiple) thread 3: duration=40, iteration=4, product=160
(alarm-multiple) thread 2: duration=30, iteration=6, product=180
(alarm-multiple) thread 3: duration=40, iteration=5, product=200
(alarm-multiple) thread 4: duration=50, iteration=4, product=200
(alarm-multiple) thread 2: duration=30, iteration=7, product=210
(alarm-multiple) thread 3: duration=40, iteration=6, product=240
(alarm-multiple) thread 4: duration=50, iteration=5, product=250
(alarm-multiple) thread 3: duration=40, iteration=7, product=280
(alarm-multiple) thread 4: duration=50, iteration=6, product=300
(alarm-multiple) thread 4: duration=50, iteration=7, product=350
(alarm-multiple) end
```

duration 和 iteration 的乘积已经是不减排序了，修改完成。运行`make check`查看结果：

![image-20191219030334430](Report.assets/image-20191219030334430.png)

除了在 Mission 2 中要解决的`alarm-priority`以外都显示通过测试。

### Misson 2: Priority Scheduling (优先级调度问题)

#### Requirements

这一部分要求在 Pintos 中实现优先级调度。

在 thread  的 PCB 中已经具有了 `priority`项，优先级最低从`PRI_MIN`到最高`PRI_MAX`。当 ready list 中出现了一个比当前正在运行的进程优先级更高的进程的时候，当前的进程将会立即让出 CPU。同时，当进程在等待一个信号量的时候，最高优先级的进程将会被第一个唤醒。

除此之外，实验还要求每一个进程可以在任意时候提高或降优先级，并且在降低优先级后如果不是当前系统中优先级最高的进程，立即让出 CPU。

除此之外，需要实现的问题还有优先级倒置、优先级捐赠。需要完成`thread.c`中的`void thread_set_priority (int new_priority)`函数和`int thread_get_priority (void)`函数。

#### Analysis

操作系统目前的调度函数为`schedule()`，代码如下：

```c
/* Schedules a new process.  At entry, interrupts must be off and
   the running process's state must have been changed from
   running to some other state.  This function finds another
   thread to run and switches to it.

   It's not safe to call printf() until thread_schedule_tail()
   has completed. */
static void
schedule (void) 
{
  struct thread *cur = running_thread ();
  struct thread *next = next_thread_to_run ();
  struct thread *prev = NULL;

  ASSERT (intr_get_level () == INTR_OFF);
  ASSERT (cur->status != THREAD_RUNNING);
  ASSERT (is_thread (next));

  if (cur != next)
    prev = switch_threads (cur, next);
  thread_schedule_tail (prev);
}
```

可以看到在第 12 行，函数通过调用`next_thread_to_run()`获取要调度的下一个进程，然后在第 20 行调用 `switch_threads(cur, next)`把当前进程和要调度的下一个进程进行交换。

观察目前`next_thread_to_run()`的代码：

```c
/* Chooses and returns the next thread to be scheduled.  Should
   return a thread from the run queue, unless the run queue is
   empty.  (If the running thread can continue running, then it
   will be in the run queue.)  If the run queue is empty, return
   idle_thread. */
static struct thread *
next_thread_to_run (void) 
{
  if (list_empty (&ready_list))
    return idle_thread;
  else
    return list_entry (list_pop_front (&ready_list), struct thread, elem);
}
```

可以看到这个函数只是简单的返回 `ready_list` 中最前面的进程。如果队列为空，则返回一`idle_thread`空进程。

而观察`thread.c`中添加进程方式，发现有三个函数在向 `ready_list`中添加进程，分别是`thread_list()`、`init_thread()`、和`thread_yield()`。而这三个函数都使用了`list_push_back (&ready_list, &cur->elem);`来添加。因此，pintos 目前使用的算法是 FIFO 调度。

为了实现优先级调度，首先应当使**进程进入队列的时候按照优先级的大小插入**，而不是简单的放在后面。

发现`list.c`内已经具有了方法`list_insert_ordered()`，可以将项目按照顺序插入：

```c
/* Inserts ELEM in the proper position in LIST, which must be
   sorted according to LESS given auxiliary data AUX.
   Runs in O(n) average case in the number of elements in LIST. */
void
list_insert_ordered (struct list *list, struct list_elem *elem,
                     list_less_func *less, void *aux){...}
```

因此，对于该过程，需要：

- 自行实现 `list_less_func`用于比较优先级
- 然后把这三个具有安排线程的函数中的调度函数从`list_push_back ();`更改成``list_insert_ordered()``。

查看官方文档：

![image-20191227004845850](Report.assets/image-20191227004845850.png)

除了当新的进程出现的时候需要根据优先级进行调度和安排，当某一个进程的优先级改变并且比当前进程的优先级高的时候，也需要内核立即让出程序，即调用`thread_yield`。所以，当有新的进程出现（在`init_thread`中）或者当某一进程优先级改变的时候（在`thread_set_priority()`中），也需要抢占式地改变当前正在运行的程序。

在实验手册的 F&Q 部分也有进一步说明：

![image-20191227034737867](Report.assets/image-20191227034737867.png)

高优先级的进程在具备运行的机会时候，应当以及运行。因此，在这里需要做两个更改：

- 当新的进程建立时，判断新进程的优先级，如果高则抢占
- 更新一个进程的优先级时，判断更新后的优先级，如果高则抢占。

另外，对于锁的唤醒问题，当有一系列程序等待锁释放的时候，需要**最先唤醒优先级最高**的程序。对于这一个要求，观察现有的锁函数（位于`synch.c`）：

```c
/* Acquires LOCK, sleeping until it becomes available if
   necessary.  The lock must not already be held by the current
   thread.

   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but interrupts will be turned back on if
   we need to sleep. */
void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  sema_down (&lock->semaphore);
  lock->holder = thread_current ();
}

```

观察`sema_down()`函数的注释：

```c
/* Down or "P" operation on a semaphore.  Waits for SEMA's value
   to become positive and then atomically decrements it.

   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but if it sleeps then the next scheduled
   thread will probably turn interrupts back on. */
void
sema_down (struct semaphore *sema) 
{...}
```

可以发现	`sema_down()`是信号量的`P`操作。而对于这个信号量，原代码中的定义如下：

```c
/* A counting semaphore. */
struct semaphore 
  {
    unsigned value;             /* Current value. */
    struct list waiters;        /* List of waiting threads. */
  };
```

可以看到维持了一个等待进程的队列。但进一步观察发现所有对`waiters`队列的操作都是同样采用的 FIFO 算法，所以这里需要：

- 修改所有对`waiters`队列的，使其变成优先级队列。

对于 **优先级倒转**，官方的文档中有如下叙述：

![image-20191227033453624](Report.assets/image-20191227033453624.png)

也就是说，当一个低优先级进程占有了一个资源锁并已经在等待队列中，一个中优先级进程也在等待队列中在低优先级进程之前。如果这个时候一个高优先级的进程，也就是正在运行的进程申请使用这个资源，就会陷入死锁的状态。因为**高优先级进程为了使用资源必须要让低优先级进程释放资源锁，但低优先级进程却已经被放在了队列的后方，永远无法被调度出来。**

所以这里需要有一个优先级翻转的过程，即先让这个低优先级的线程暂时获得比较高的优先级并处理，让它把资源锁释放；释放了之后再将优先级恢复成之前的状态。恢复后再放回队列中的位置，不会影响之前已经实现了的调度策略。

通过阅读官方文档，可以使用优先级捐赠算法解决这一问题。关于优先级捐赠，官方的实验手册还有如下说明：

![image-20191227034410124](Report.assets/image-20191227034410124.png)

捐赠不会导致子进程的优先级超过捐赠者，也不会改变捐赠者自身的优先级。此外，除了单个进程对单个进程的优先级捐赠以外，还需要考虑多个进程之间的捐赠情况：

![image-20191227035001712](Report.assets/image-20191227035001712.png)

当出现了高优先级进程访问中优先级进程的锁，同时中优先级进程访问低优先级进程的锁的情况的时候，需要发生递归捐赠，低优先级的进程最终会被提升到高优先级进程的优先级，然后在锁释放后恢复原先的优先级。

如果没有递归捐赠的步骤，那么高优先级的进程会把优先级捐赠给中优先级进程，但此时之前中优先级进程为了不让低优先级进程死锁而捐赠的优先级就会没有用了。所以这个时候需要把低优先级进程的优先级也提升上来。

而现有的锁的定义为：

```c
/* Lock. */
struct lock 
  {
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
  };
```

不支持多个正在访问这个锁的进程的记录，需要稍后修改。

总结起来为了解决优先级捐赠问题需要有以下修改步骤：

- 需要修改线程的结构体定义来存储线程是否被捐赠了优先级，之前的优先级和新的优先级。
- 在一个线程获取一个锁的时候， 如果拥有这个锁的线程优先级比自己低就提高它的优先级；如果还有其他进程在使用这个锁，也会相应地提升这个进程的优先级
- 如果线程同时被多个进程捐赠优先级，线程的优先级将会变成这些捐赠进程的优先级中的最高值

#### Solution

首先为了实现`list_insert_ordered()`函数，首先需要自行编写一个比较优先级的函数。在`thread.c`加入以下比较函数：

```c
/* ++1.2 Compare priority */
bool compare_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED) {
	int pa = list_entry(a, struct thread, elem)->priority;
	int pb = list_entry(b, struct thread, elem)->priority;
	return pa > pb;
}
```

同时在`thread.h`中声明：

```c
/* +++1.2 */
bool compare_priority(const struct list_elem *, const struct list_elem *, void *);
```

修改`yield()`中的调度语句：

![image-20191227014606915](Report.assets/image-20191227014606915.png)

另两个函数`thread_unblock()`同样做相同更改：

```c
  //list_push_back (&ready_list, &t->lelem);
  /* +++1.2 Priority */
  list_insert_ordered(&ready_list, &t->elem, (list_less_func *)&compare_priority, NULL);
```
对``init_thread()`：传递的参数为`all_list`和`allelem`。
```c
//list_push_back (&all_list, &t->allelem); 
/* ++1.2 Priority */
list_insert_ordered(&all_list, &t->allelem, (list_less_func *)&compare_priority, NULL);
```

接下来修改设置优先级函数中的语句：

![image-20191227024048341](Report.assets/image-20191227024048341.png)

至此，修改了`thread.c`中的优先级调度程序，已经可以通过部分和优先级有关的测试了：

![image-20191227033105852](Report.assets/image-20191227033105852.png)

接下来对于优先级捐赠问题，首先需要`thread`定义中加入以下数据结构：

![image-20191229011917084](Report.assets/image-20191229011917084.png)

然后为`lock`的定义加入以下两个数据结构。前者是当前线程在信号量队列中位置（在原始的队列中，当前线程一定位于队列的首部），后者则是表示该锁的信号量队列中线程的最高优先级（用于优先级的捐赠）：

![image-20191229012710425](Report.assets/image-20191229012710425.png)

为了让修改的数据结构能够得到初始化，修改线程的初始化函数`init_thread`和锁的初始化函数`lock_init`：![image-20191229013249507](Report.assets/image-20191229013249507.png)

![image-20191229013515914](Report.assets/image-20191229013515914.png)

由于全程都在涉及关于锁的操作，所以先对获取锁的函数`lock_aquire()`进行修改。也就是要让每一个锁在获得的时候首先检测比较想要获得这个锁的进程的优先级是否比锁内存储的`max_priority`大，如果更大，则需要将这个锁内存储的`max_priority`设置为最大的优先级，并且对进程进行优先级捐赠操作：

```c
/* Acquires LOCK, sleeping until it becomes available if
   necessary.  The lock must not already be held by the current
   thread.

   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but interrupts will be turned back on if
   we need to sleep. */
void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  /** ++1.2 Priority Donate */
  struct thread *current_thread = thread_current();
  struct lock *l;
  enum intr_level old_level;

  if (lock->holder != NULL && !thread_mlfqs) {
	  current_thread->lock_waiting = lock;
	  l = lock;
	  while (l && current_thread->priority > l->max_priority) {
		  l->max_priority = current_thread->priority;
		  thread_donate_priority(l->holder);
		  l = l->holder->lock_waiting;
	  }
  }

  sema_down(&lock->semaphore);
  old_level = intr_disable();
  current_thread = thread_current();
  if (!thread_mlfqs) {
	  current_thread->lock_waiting = NULL;
	  lock->max_priority = current_thread->priority;
	  thread_hold_the_lock(lock);
  }
  lock->holder = current_thread;
  intr_set_level(old_level);
  // sema_down (&lock->semaphore);
  // lock->holder = thread_current ();
}
```

对应的在`thread.c`中实现`thread_donate_priority`和`thread_hold_the_lock`函数。

其中`thread_donate_priority`需要自行实现一个对`t`进行优先级更新的函数，因为被捐赠优先级的进程不一定是正在运行的进程，之前程序自带的`thread_set_priority()`只能满足更新当前进程的优先级，所以需要修改。

而`thread_hold_the_lock`则是让进程获得当前锁。由于如果进程拥有一个锁，那么进程的优先级一定要是拥有这个锁的队列中的最大值，所以如果锁的优先级大于进程的优先级，需要相应地更新进程的优先级，然后把这个锁加入到进程拥有的锁的队列中。

这两个函数的代码如下：

```c
/* ++1.2 Let thread hold a loc*/
void thread_hold_the_lock(struct lock *lock) {
	enum intr_level old_level = intr_disable();
	list_insert_ordered(&thread_current()->locks, &lock->elem, lock_cmp_priority, NULL);

	if (lock->max_priority > thread_current()->priority) {
		thread_current()->priority = lock->max_priority;
		thread_yield();
	}

	intr_set_level(old_level);
}

/* ++1.2 Donate current priority to thread t. */
void thread_donate_priority(struct thread *t) {
	enum intr_level old_level = intr_disable();
	thread_update_priority(t);

	if (t->status == THREAD_READY) {
		list_remove(&t->elem);
		list_insert_ordered(&ready_list, &t->elem, compare_priority, NULL);
	}
	intr_set_level(old_level);
}
```

接下来对于`thread_update_priority()`函数编写如下。当前已经有的修改函数`thread_set_priority()`只能用于修改当前正在运行的优先级。不过这个函数可以修改一下用于更新当前运行线程的`base_priority`并根据新的优先级来判断是否需要`yield`进程。

在这之前，先完善修改当前进程优先级的函数`thread_set_priority()`如下：

```c
/* Sets the current thread's priority to NEW_PRIORITY. */
void
thread_set_priority (int new_priority) 
{
  /* ++1.2 Handle priority */
	int old_priority = thread_current()->priority;
	enum intr_level old_level = intr_disable();
	struct thread *current_thread = thread_current();
	current_thread->base_priority = new_priority;

	if (list_empty(&current_thread->locks) || new_priority > old_priority) {
		current_thread->priority = new_priority;
		thread_yield();
	}
	intr_set_level(old_level);
	/* +++ 1.2 (OLD )Handle priority */
	// if (new_priority < old_priority) {
	// 	thread_yield();
  // }
}
```

此后再编写一个`thread_update_priority()`函数来更新当前正在运行的进程的优先级。使用这个函数来应对多个进程同时在对一个进程进行优先级捐赠的情况。为了使设置的优先级为锁的最大值，需要对某个进程所获得的锁进行排序，然后取出队列中最前的元素作为当前进程的新优先级。

```c
/* ++1.2 Used to update priority. */
void thread_update_priority(struct thread *t) {
	enum intr_level old_level = intr_disable();
	int max_priority = t->base_priority;
	int lock_priority;

	if (!list_empty(&t->locks)) {
		list_sort(&t->locks, lock_cmp_priority, NULL);
		lock_priority = list_entry(list_front(&t->locks), struct lock, elem)->max_priority;
		if (lock_priority > max_priority)
			max_priority = lock_priority;
	}

	t->priority = max_priority;
	intr_set_level(old_level);
}
```

显然为了实现关于锁的排序函数，需要对应地实现`lock_cmp_priority`函数如下：

```c
/* ++1.2 Compare priority in locks */
bool lock_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED) {
	return list_entry(a, struct lock, elem)->max_priority > list_entry(b, struct lock, elem)->max_priority;
}
```

同时在`thread.h`中加入刚刚编写的几个函数的声明：

![image-20191230011352327](Report.assets/image-20191230011352327.png)

以上实现了获取锁的逻辑。而对于锁的释放，需要先把锁从线程的`lock`队列中删除，然后将锁设置为不被任何进程占用，最后再进行信号量的`V`操作。这里编写一个`thread_remove_lock`函数来实现：

```c
/* ++1.2 Remove a lock. */
void thread_remove_lock(struct lock *lock) {
	enum intr_level old_level = intr_disable();
	list_remove(&lock->elem);
	thread_update_priority(thread_current());
	intr_set_level(old_level);
}
```

在进行`lock_release()`的时候调用该函数：

```c
  /* ++1.2 */
  if(!thread_mlfqs){
	  thread_remove_lock(lock);
  }
```

最后将剩下的队列修改为优先级队列。首先修改`synch.c`中的`cond_signal`函数：

![image-20191230010232328](Report.assets/image-20191230010232328.png)

完善其排序函数：

```c
/* ++1.2 cond sema comparation function */
bool cond_sema_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED) {
	struct semaphore_elem *sa = list_entry(a, struct semaphore_elem, elem);
	struct semaphore_elem *sb = list_entry(b, struct semaphore_elem, elem);
	return list_entry(list_front(&sa->semaphore.waiters), struct thread, elem)->priority > list_entry(list_front(&sb->semaphore.waiters), struct thread, elem)->priority;
}
```

再修改`waiters`为优先级队列。对`sema_down`和`sema_up`修改如下：

![image-20191230011140332](Report.assets/image-20191230011140332.png)

![image-20191230010730007](Report.assets/image-20191230010730007.png)

#### 实验结果

完成以上步骤以后，可以看到`priority`部分的测试已经全部通过，优先级调度部分的修改完成。

![image-20191230015304267](Report.assets/image-20191230015304267.png)

```c
7 of 27 tests failed.
```

### Mission 3: Advanced Scheduler

