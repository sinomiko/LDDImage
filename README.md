#深入浅出Linux 设备驱动编程 

# 1.安装开发包
安装的过程中应该选中“开发工具”和“内核开发”二项（如果本文的例程要在特定的嵌入式系统中运行，还应安装相应的交叉编译器，并
包含相应的Linux 源代码），如下图：

![image](https://github.com/sinomiko/LDDImage/blob/master/res/kernel_package.png?raw=true)

# 2.Linux 内核模块
Linux 设备驱动属于内核的一部分，Linux 内核的一个模块可以以两种方式被编译和加
载：
1. 直接编译进Linux 内核，随同Linux 启动时加载；
2. 编译成一个可加载和删除的模块，使用insmod 加载（modprobe 和insmod 命令类
似，但依赖于相关的配置文件），rmmod 删除。这种方式控制了内核的大小，而模块一旦被
插入内核，它就和内核其他部分一样。

一个内核模块的例子：
```c
#include <linux/module.h> //所有模块都需要的头文件
#include <linux/init.h> // init&exit 相关宏
MODULE_LICENSE("GPL");
static int __init hello_init (void)
{
    printk("Hello module init\n");
return 0;
}
static void __exit hello_exit (void)
{
    printk("Hello module exit\n");
}
module_init(hello_init)
module_exit(hello_exit);
```

Linux 内核模块需包含模块初始化和模块卸载函数，
前者在insmod 的时候运行，后者在rmmod 的时候运行;
初始化与卸载函数必须在宏module_init和module_exit 使用前定义，否则会出现编译错误。

程序中的 MODULE_LICENSE("GPL")用于声明模块的许可证
如果要把上述程序编译为一个运行时加载和删除的模块，则编译命令为：
```
gcc –D__KERNEL__ -DMODULE –DLINUX –I /usr/local/src/linux2.4/include -c –o hello.o hello.c
```

Linux 内核模块的编译需要给gcc 指示–D__KERNEL__ -DMODULE –DLINUX 参数 -I 选项跟着Linux 内核源代码中Include 目录的路径。

加载 hello 模块：
```
insmod ./hello.o
```

下列命令完成相反过程：
```
rmmod hello
```

如果要将其直接编译入Linux 内核，则需要将源代码文件拷贝入Linux 内核源代码的相
应路径里，并修改Makefile。



##  Linux 内核编程的一些基本知识：
###   1.内存
在 Linux 内核模式下，我们不能使用用户态的malloc()和free()函数申请和释放内存。进行内核编程时，最常用的内存申请和释放函数为在include/linux/kernel.h 文件中声明的kmalloc()和kfree()，其原型为：
```c
void *kmalloc(unsigned int len, int priority);
void kfree(void *__ptr);
```

kmalloc 的priority 参数通常设置为==GFP_KERNEL==

**如果在中断服务程序里申请内存则要用GFP_ATOMIC 参数，因为使用GFP_KERNEL 参数可能会引起睡眠，不能用于非进程上下文中（在中断中是不允许睡眠的）。**


###   2.内核态和用户态数据交换
由于内核态和用户态使用不同的内存定义，所以二者之间不能直接访问对方的内存。而应该使用Linux 中的用户和内核态内存交互函数（这些函数在include/asm/uaccess.h 中被声明）：
```c
unsigned long copy_from_user(void *to, const void *from, unsigned long n);
unsigned long copy_to_user (void * to, void * from, unsigned long len);
```

copy_from_user、copy_to_user 函数返回不能被复制的字节数，因此，如果完全复制成功，返回值为0。

include/asm/uaccess.h 中定义的put_user 和get_user 用于内核空间和用户空间的单值交互（如char、int、long）。

这里给出的仅仅是关于内核中内存管理的皮毛，关于 Linux 内存管理的更多细节知识，我们会在本文第9 节《内存与I/O 操作》进行更加深入地介绍。

###  3.输出
==在内核编程中，我们不能使用用户态 C库函数中的printf()函数输出信息，而只能使用printk()。==

但是，**内核中printk()函数的设计目的并不是为了和用户交流，它实际上是内核的一种日志机制，用来记录下日志信息或者给出警告提示。**

每个printk 都会有个优先级，内核一共有8 个优先级，它们都有对应的宏定义。如果未指定优先级，内核会选择默认的优先级DEFAULT_MESSAGE_LOGLEVEL。

如果优先级数字比int console_loglevel 变量小的话，消息就会打印到控制台上。

**如果syslogd 和klogd 守护进程在运行的话，则不管是否向控制台输出，消息都会被追加进/var/log/messages 文件。**

==klogd只处理内核消息，syslogd 处理其他系统消息，比如应用程序。==

###   4.模块参数
2.4 内核下，include/linux/module.h 中定义的宏MODULE_PARM(var,type) 用于向模块传递命令行参数。

var 为接受参数值的变量名， type 为采取如下格式的字符串[min[-max]]{b,h,i,l,s}。

min 及max 用于表示当参数为数组类型时，

允许输入的数组元素的个数范围；b：byte；h：short；i：int；l：long；s：string。

在装载内核模块时，用户可以向模块传递一些参数：insmod modname var=value如果用户未指定参数，var 将使用模块内定义的缺省值。

#   3.字符设备驱动程序
Linux 下的设备驱动程序被组织为一组完成不同任务的函数的集合，通过这些函数使得Linux的设备操作犹如文件一般。

在应用程序看来，硬件设备只是一个设备文件，应用程序可以象操作普通文件一样对硬件设备进行操作，如open ()、close ()、read ()、write () 等。

==Linux 主要将设备分为二类：字符设备和块设备。==

字符设备是指设备发送和接收数据以字符的形式进行；而块设备则以整个数据缓冲区的形式进行。字符设备的驱动相对比较简单。

下面我们来假设一个非常简单的虚拟字符设备：这个设备中只有一个4 个字节的全局变量int global_var，而这个设备的名字叫做“gobalvar”。对“gobalvar”设备的读写等操作即是对其中全局变量global_var 的操作。

驱动程序是内核的一部分，因此我们需要给其添加模块初始化函数，该函数用来完成对所控设备的初始化工作，并调用register_chrdev() 函数注册字符设备：

```c
static int __init gobalvar_init(void)
{
    if (register_chrdev(MAJOR_NUM, " gobalvar ", &gobalvar_fops))
    {
        //…注册失败
    }
    else
    {
        //…注册成功
    }
}
```

其中，register_chrdev 函数中的参数MAJOR_NUM 为主设备号,“gobalvar”为设备名，gobalvar_fops 为包含基本函数入口点的结构体，类型为file_operations。

**当gobalvar 模块被加载时，gobalvar_init 被执行，它将调用内核函数register_chrdev，把驱动程序的基本入口点指针存放在内核的字符设备地址表中，在用户进程对该设备执行系统调用时提供入口地址。**

与模块初始化函数对应的就是模块卸载函数，需要调用
```c
register_chrdev()的“反函数unregister_chrdev()：
static void __exit gobalvar_exit(void)
{
    if (unregister_chrdev(MAJOR_NUM, " gobalvar "))
    {
        //…卸载失败
    }
    else
    {
        //…卸载成功
    }
}	
```

随着内核不断增加新的功能，file_operations 结构体已逐渐变得越来越大，

==但是大多数的驱动程序只是利用了其中的一部分。对于字符设备来说，要提供的主要入口有：open ()、release ()、read ()、write ()、ioctl ()、llseek()、poll()等。==

####  1.open()函数 
对设备特殊文件进行open()系统调用时，将调用驱动程序的open () 函数：
```c
int (*open)(struct inode * ,struct file *);
```

其中参数inode 为设备特殊文件的inode (索引结点) 结构的指针，参数file 是指向这一设备的文件结构的指针。

open()的主要任务是确定硬件处在就绪状态、验证次设备号的合法性

(次设备号可以用MINOR(inode-> i - rdev) 取得)、控制使用设备的进程数、根据执行情况返回状态码(0 表示成功，负数表示存在错误) 等；

####  2.release()函数
 当最后一个打开设备的用户进程执行close ()系统调用时，内核将调用驱动程序的release () 函数：
 ```c
void (*release) (struct inode * ,struct file *) ;
```

release 函数的主要任务是清理未结束的输入/输出操作、释放资源、用户自定义排他标志的复位等。

####  3.read()函数
 当对设备特殊文件进行read() 系统调用时，将调用驱动程序read() 函数：
 ```c
ssize_t (*read) (struct file *, char *, size_t, loff_t *);
```

用来从设备中读取数据。

当该函数指针被赋为NULL 值时，将导致read 系统调用出错并返回-EINVAL（“Invalid argument，非法参数”）。

函数返回非负值表示成功读取的字节数
（返回值为“signed size”数据类型，通常就是目标平台上的固有整数类型）。


globalvar_read 函数中内核空间与用户空间的内存交互需要借助第2 节所介绍的函数
```c
static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
    …
    copy_to_user(buf, &global_var, sizeof(int));
    …
}
```

####  4.write( ) 函数 
当设备特殊文件进行write () 系统调用时，将调用驱动程序的write () 函
数：
```c
ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
```

向设备发送数据。

如果没有这个函数，write 系统调用会向调用程序返回一个-EINVAL。

如果返回值非负，则表示成功写入的字节数。

globalvar_write 函数中内核空间与用户空间的内存交互需要借助第2 节所介绍的函数：

```c
static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len, loff_t *off)
{
    …
    copy_from_user(&global_var, buf, sizeof(int));
    …
}
```

####  5.ioctl() 函数
该函数是特殊的控制函数，可以通过它向设备传递控制信息或从设备取得状态信息，函数原型为：
```c
int (*ioctl) (struct inode * ,struct file * ,unsigned int ,unsigned long);
```

unsigned int 参数为设备驱动程序要执行的命令的代码，由用户自定义，unsigned long参数为相应的命令提供参数，类型可以是整型、指针等。

==如果设备不提供ioctl 入口点，则对于任何内核未预先定义的请求，ioctl 系统调用将返回错误（-ENOTTY，“No such ioctl fordevice，该设备无此ioctl 命令”）。==

如果该设备方法返回一个非负值，那么该值会被返回给调用程序以表示调用成功。

####  6.llseek()函数

该函数用来修改文件的当前读写位置，并将新位置作为（正的）返回值返回，原型为：
```c
loff_t (*llseek) (struct file *, loff_t, int);
```

####  7.poll()函数
poll 方法是poll 和select **这两个系统调用的后端实现，用来查询设备是否可读或可写，或是否处于某种特殊状态**，

原型为：
```c
unsigned int (*poll) (struct file *, struct poll_table_struct *);
```

我们将在“设备的阻塞与非阻塞操作”一节对该函数进行更深入的介绍。设备“gobalvar”的驱动程序的这些函数应分别命名为gobalvar_open、gobalvar_ release、gobalvar_read、gobalvar_write、gobalvar_ioctl，因此设备“gobalvar”的基本入口点结构变量
gobalvar_fops 赋值如下：
```c
struct file_operations gobalvar_fops = {
read: gobalvar_read,
write: gobalvar_write,
};
```

上述代码中对gobalvar_fops 的初始化方法并不是标准C 所支持的，属于GNU 扩展语
法。

完整的 globalvar.c 文件源代码如下：
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
MODULE_LICENSE("GPL");
#define MAJOR_NUM 254 //主设备号
static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);
//初始化字符设备驱动的file_operations 结构体
struct file_operations globalvar_fops =
{
    read: globalvar_read, write: globalvar_write,
};
static int global_var = 0; //“globalvar”设备的全局变量
static int __init globalvar_init(void)
{
    int ret;
    //注册设备驱动
    ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
    if (ret)
    {
        printk("globalvar register failure");
    }
    else
    {
        printk("globalvar register success");
    }
    return ret;
}

static void __exit globalvar_exit(void)
{
    int ret;
    //注销设备驱动
    ret = unregister_chrdev(MAJOR_NUM, "globalvar");
    if (ret)
    {
        printk("globalvar unregister failure");
    }
    else
    {
        printk("globalvar unregister success");
    }
}

static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
    //将global_var 从内核空间复制到用户空间
    if (copy_to_user(buf, &global_var, sizeof(int)))
    {
        return - EFAULT;
    }
    return sizeof(int);
}

static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len, loff_t *off)
{
    //将用户空间的数据复制到内核空间的global_var
    if (copy_from_user(&global_var, buf, sizeof(int)))
    {
        return - EFAULT;
    }
    return sizeof(int);
}

module_init(globalvar_init);
module_exit(globalvar_exit);
```

运行
```bash
gcc –D__KERNEL__ -DMODULE –DLINUX –I /usr/local/src/linux2.4/include -c –o globalvar.o globalvar.c
```

编译代码，运
```bash
inmod globalvar.o
```

加载globalvar 模块，再运行
```bash
cat /proc/devices
```

发现其中多出了“254 globalvar”一行，如下图：
![image](https://github.com/sinomiko/LDDImage/blob/master/res/globalvar_fops%20.png?raw=true)

接着我们可以运行：
```bash
mknod /dev/globalvar c 254 0
```

# 4.设备驱动中的并发控制

**在驱动程序中，当多个线程同时访问相同的资源时（驱动程序中的全局变量是一种典型的共享资源），可能会引发“竞态”，因此我们必须对共享资源进行并发控制。**

**Linux 内核中解决并发控制的最常用方法是自旋锁与信号量（绝大多数时候作为互斥锁使用）。**

自旋锁与信号量“类似而不类”，类似说的是它们功能上的相似性，“不类”指代它们在本质和实现机理上完全不一样，不属于一类。

==++自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环查看是否该自旋锁的保持者已经释放了锁，“自旋”就是“在原地打转”。++==

==++而信号量则引起调用者睡眠，它把进程从运行队列上拖出去，除非获得锁。这就是它们的“不类”。++==

但是，无论是信号量，还是自旋锁，==**在任何时刻，最多只能有一个保持者，即在任何时刻最多只能有一个执行单元获得锁**==。这就是它们的“类似”。

鉴于自旋锁与信号量的上述特点，一般而言，**自旋锁适合于保持时间非常短的情况，它可以在任何上下文使用；信号量适合于保持时间较长的情况，会只能在进程上下文使用**。如果被保护的共享资源只在进程上下文访问，则可以以信号量来保护该共享资源，如果对共享资源的访问时间非常短，自旋锁也是好的选择。**但是，如果被保护的共享资源需要在中断上下文访问（包括底半部即中断处理句柄和顶半部即软中断），就必须使用自旋锁。**

与信号量相关的 API 主要有：

-  定义信号量
```c
    struct semaphore sem;
```

- 初始化信号量
```c
void sema_init (struct semaphore *sem, int val);
```

该函数初始化信号量，并设置信号量sem 的值为val
```c
void init_MUTEX (struct semaphore *sem);
```

该函数用于初始化一个互斥锁，即它把信号量sem 的值设置为1，等同于sema_init (structsemaphore *sem, 1)；
```c
void init_MUTEX_LOCKED (struct semaphore *sem);
```

该函数也用于初始化一个互斥锁，但它把信号量sem 的值设置为0，等同于sema_init(struct semaphore *sem, 0)；

-   获得信号量
```c
void down(struct semaphore * sem);
```
**该函数用于获得信号量sem，它会导致睡眠，因此不能在中断上下文使用；**
```c
int down_interruptible(struct semaphore * sem);
```
**该函数功能与down 类似，不同之处为，down 不能被信号打断，但down_interruptible能被信号打断；**

-   获得信号量
```c
void down(struct semaphore * sem);
```
**该函数用于获得信号量sem，它会导致睡眠，因此不能在中断上下文使用；**
```c
int down_interruptible(struct semaphore * sem);
```
**该函数功能与down 类似，不同之处为，down 不能被信号打断，但down_interruptible
能被信号打断；**
```c
int down_trylock(struct semaphore * sem);
```
该函数尝试获得信号量sem，如果能够立刻获得，它就获得该信号量并返回0，否则，
返回非0 值。

它不会导致调用者睡眠，可以在中断上下文使用。

-   释放信号量
```c
void up(struct semaphore * sem);
```
该函数释放信号量sem，唤醒等待者。
与自旋锁相关的 API 主要有：

-   定义自旋锁
```c
spinlock_t spin;
```

-   初始化自旋锁
```c
spin_lock_init(lock)
```

该宏用于动态初始化自旋锁lock
-   获得自旋锁

```c
spin_lock(lock)
```
该宏用于获得自旋锁lock，如果能够立即获得锁，它就马上返回，否则，它将自旋在那里，直到该自旋锁的保持者释放；

```c
spin_trylock(lock)
```
该宏尝试获得自旋锁lock，如果能立即获得锁，它获得锁并返回真，否则立即返回假，
实际上不再“在原地打转”；

-   释放自旋锁
```c
spin_unlock(lock)
```
**该宏释放自旋锁lock，它与spin_trylock 或spin_lock 配对使用；**
除此之外，还有一组自旋锁使用于中断情况下的 API。

下面进入对并发控制的实战。首先，在 globalvar 的驱动程序中，我们可以通过信号量来控制对int global_var 的并发访问，下面给出源代码：

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
#include <asm/semaphore.h>
MODULE_LICENSE("GPL");
#define MAJOR_NUM 254
static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);
struct file_operations globalvar_fops =
{
    read: globalvar_read, write: globalvar_write,
};
static int global_var = 0;
static struct semaphore sem;
static int __init globalvar_init(void)
{
    int ret;
    ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
    if (ret)
    {
        printk("globalvar register failure");
    }
    else
    {
        printk("globalvar register success");
        init_MUTEX(&sem);
    }
    return ret;
}

static void __exit globalvar_exit(void)
{
    int ret;
    ret = unregister_chrdev(MAJOR_NUM, "globalvar");
    if (ret)
    {
        printk("globalvar unregister failure");
    }
    else
    {
        printk("globalvar unregister success");
    }
}

static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
    //获得信号量
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS;
    }
    //将global_var 从内核空间复制到用户空间
    if (copy_to_user(buf, &global_var, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    //释放信号量
    up(&sem);
    return sizeof(int);
}

ssize_t globalvar_write(struct file *filp, const char *buf, size_t len, loff_t* off)
{
    //获得信号量
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS;
    }
    //将用户空间的数据复制到内核空间的global_var
    if (copy_from_user(&global_var, buf, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    //释放信号量
    up(&sem);
    return sizeof(int);
}
module_init(globalvar_init);
module_exit(globalvar_exit);
```

接下来，我们给globalvar 的驱动程序增加open()和release()函数，并在其中借助自旋锁来保护对全局变量int globalvar_count（记录打开设备的进程数）的访问来实现设备只能被一个进程打开（必须确保globalvar_count 最多只能为1）

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
#include <asm/semaphore.h>

MODULE_LICENSE("GPL");

#define MAJOR_NUM 254


static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);
static int globalvar_open(struct inode *inode, struct file *filp);
static int globalvar_release(struct inode *inode, struct file *filp);

struct file_operations globalvar_fops =
{
    read: globalvar_read, write: globalvar_write, open: globalvar_open, release:
    globalvar_release,
};


static int global_var = 0;
static int globalvar_count = 0;
static struct semaphore sem;
static spinlock_t spin = SPIN_LOCK_UNLOCKED;


static int __init globalvar_init(void)
{
    int ret;
    ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
    if (ret)
    {
        printk("globalvar register failure");
    }
    else
    {
        printk("globalvar register success");
        init_MUTEX(&sem);
    }
    return ret;
}

static void __exit globalvar_exit(void)
{
    int ret;
    ret = unregister_chrdev(MAJOR_NUM, "globalvar");
    if (ret)
    {
        printk("globalvar unregister failure");
    }
    else
    {
        printk("globalvar unregister success");
    }
}


static int globalvar_open(struct inode *inode, struct file *filp)
{
    //获得自选锁
    spin_lock(&spin);
    
    //临界资源访问
    if (globalvar_count)
    {
        spin_unlock(&spin);
       return - EBUSY;
    }
    globalvar_count++;
    
    //释放自选锁
    spin_unlock(&spin);
    
    return 0;
}

static int globalvar_release(struct inode *inode, struct file *filp)
{
    globalvar_count--;
    return 0;
}

static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS;
    }
    if (copy_to_user(buf, &global_var, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    up(&sem);
    return sizeof(int);
}

static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len,loff_t *off)
{
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS
    }
    if (copy_from_user(&global_var, buf, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    up(&sem);
    return sizeof(int);
}
module_init(globalvar_init);
module_exit(globalvar_exit);

```

为了上述驱动程序的效果，我们启动两个进程分别打开/dev/globalvar。在两个终端中调
用./globalvartest.o 测试程序，当一个进程打开/dev/globalvar 后，另外一个进程将打开失败，
输出“device open failure”，如下图：
![image](https://github.com/sinomiko/LDDImage/blob/master/res/device_open_failure.png?raw=true)

# 5.设备的阻塞与非阻塞操作
**阻塞操作是指，在执行设备操作时，若不能获得资源，则进程挂起直到满足可操作的条件再进行操作。**

**非阻塞操作的进程在不能进行设备操作时，并不挂起。**

**被挂起的进程进入sleep 状态，被从调度器的运行队列移走，直到等待的条件被满足。**


==在 Linux 驱动程序中，我们可以使用等待队列（wait queue）来实现阻塞操作。wait queue 很早就作为一个基本的功能单位出现在Linux内核里了，它以队列为基础数据结构，与进程调度机制紧密结合，能够用于实现核心的异步事件通知机制。等待队列可以用来同步对系统资源的访问，上节中所讲述Linux信号量在内核中也是由等待队列来实现的。==

下面我们重新定义设备“globalvar”，它可以被多个进程打开，但是每次只有当一个进
程写入了一个数据之后本进程或其它进程才可以读取该数据，否则一直阻塞。

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <asm/uaccess.h>
#include <linux/wait.h>
#include <asm/semaphore.h>

MODULE_LICENSE("GPL");

#define MAJOR_NUM 254

static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);

struct file_operations globalvar_fops =
{
    read: globalvar_read, write: globalvar_write,
};

static int global_var = 0;
static struct semaphore sem;
static wait_queue_head_t outq;
static int flag = 0;

static int __init globalvar_init(void)
{
    int ret;
    ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
    if (ret)
    {
        printk("globalvar register failure");
    }
    else
    {
        printk("globalvar register success");
        init_MUTEX(&sem);
        init_waitqueue_head(&outq);
    }
    return ret;
}

static void __exit globalvar_exit(void)
{
    int ret;
    ret = unregister_chrdev(MAJOR_NUM, "globalvar");
    if (ret)
    {
        printk("globalvar unregister failure");
    }
    else
    {
        printk("globalvar unregister success");
    }
}

static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
    //等待数据可获得
    if (wait_event_interruptible(outq, flag != 0))
    {
        return - ERESTARTSYS;
    }
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS;
    }
    flag = 0;
    if (copy_to_user(buf, &global_var, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    up(&sem);
    return sizeof(int);
}

static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len,loff_t *off)
{
    if (down_interruptible(&sem))
    {
        return - ERESTARTSYS;
    }
    if (copy_from_user(&global_var, buf, sizeof(int)))
    {
        up(&sem);
        return - EFAULT;
    }
    up(&sem);
    flag = 1;
    
    //通知数据可获得
    wake_up_interruptible(&outq);
    return sizeof(int);
}


module_init(globalvar_init);
module_exit(globalvar_exit);
```

编写两个用户态的程序来测试，第一个用于阻塞地读/dev/globalvar，另一个用于写/dev/globalvar。只有当后一个对/dev/globalvar 进行了输入之后，前者的read 才能返回。
读的程序为：

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <fcntl.h>
main()
{
    int fd, num;
    fd = open("/dev/globalvar", O_RDWR, S_IRUSR | S_IWUSR);
    if (fd != - 1)
    {
        while (1)
        {
            read(fd, &num, sizeof(int)); //程序将阻塞在此语句，除非有针对globalvar 的输入
            printf("The globalvar is %d\n", num);
            
            //如果输入是0，则退出
            if (num == 0)
            {
                close(fd);
                break;
            }
        }
    }
    else
    {
        printf("device open failure\n");
    }
}
```

写的程序为：

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <fcntl.h>
main()
{
    int fd, num;
    fd = open("/dev/globalvar", O_RDWR, S_IRUSR | S_IWUSR);
    if (fd != - 1)
    {
        while (1)
        {
            printf("Please input the globalvar:\n");
            scanf("%d", &num);
            write(fd, &num, sizeof(int));
            
            //如果输入0，退出
            if (num == 0)
            {
                close(fd);
                break;
            }
        }
    }
    else
    {
        printf("device open failure\n");
    }
}
```


打开两个终端，分别运行上述两个应用程序，发现当在第二个终端中没有输入数据时，第一个终端没有输出（阻塞），每当我们在第二个终端中给globalvar 输入一个值，第一个终端就会输出这个值，如下图：
