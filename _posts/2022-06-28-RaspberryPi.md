---
title: 树莓派开发 - 字符设备驱动开发
tags: RaspberryPi-use
article_header:
  type: cover
  image:
    src: https://photo-hq.oss-cn-hangzhou.aliyuncs.com/cover/15.png
---



------

# 驱动开发知识

### 指令

| 指令                  | 功能               |
| --------------------- | ------------------ |
| `lsmod`               | 列出系统的驱动模块 |
| `sudo mknod hq c 8 1` | 手动生成设备指令   |
| `insmod xxx.ko`       | 加载驱动模块       |
| `rmmod xxx`           | 删除驱动模块       |
|                       |                    |
|                       |                    |
|                       |                    |
|                       |                    |

### 地址

**总线地址**

地址总线属于一种电脑总线，是由CPU或者有DMA能力的单元，用来沟通这些单元想要存取（读取/写入）电脑内存元件/地方的实体位址。

简单来讲就是，CPU能够访问内存的范围。32位系统最大能识别的内存4G

**物理地址**

硬件实际地址或绝对地址。

**虚拟地址**

逻辑地址（基于算法的地址，软件层面的地址）。

### 树莓派CPU

`cat /proc/cpuinfo`

> ![image-20220507144222593](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220507144222593.png)

><img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220507143708678.png" alt="image-20220507143708678" style="zoom:67%;" />

### 树莓派芯片手册

查询芯片手册第六章 GPIO 配置，得知下面寄存器配置信息

> 也可通过官网查询各个引脚信息 pinout.xyz/pinout/

```
GPFSEL0 GPIO Function Select 0 
功能选择 输出/输入（GPIO Function Select Registers） 32位
14-12   001 = GPIO Pin4 is an output

GPSET0 GPIO Pin Output Set 0  输出1
0 = No effect
1 = Set GPIO pin n

GPCLR0 GPIO Pin Output Clear 0 清0
0 = No effect
1 = Clear GPIO pin n
```





# 驱动开发介绍

#### 驱动开发方式

1. 直接将驱动编译进内核，即内核（zImage）包含了驱动。

2. **以模块方式生成驱动文件xxx.ko（通过内核的交叉编译工具链进行编译生成），嵌入式系统启动后，通过命令inmosd xxx.ko 加载。（本笔记采用这种方法）**

   > ==采用这种方法，一定要确保交叉编译内核版本（ubuntu中内核版本）与安装模块的树莓派内核版本一致。==
   >
   > 如果不一致，将会出现下面问题：
   >
   > ![Snipaste_2022-06-17_13-59-04](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_13-59-04.png)
   >
   > ![Snipaste_2022-06-17_13-58-43](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_13-58-43.png)

3. 通过修改设备树，将驱动加入内核。

==驱动代码的编译需要一个提前编译好的内核，编译内核就必须配置，配置的最终目标会生成 .config文件，该文件指导Makefile去把有用东西组织成内核==，厂家配linux内核源码，比如说买了树莓派，树莓派linux内核源码。

#### 驱动开发流程

1. 运行应用程序之前，通过insmod将驱动模块加载到内核（执行pin4_init函数，将pin4驱动结构体加到内核的驱动链表中），然后在/dev下生成pin4。
2. 应用调用open函数，触发异常，进入内核态，调用sys_call系统调用，再调用sys_open，根据文件名找到相关的设备号，根据设备号从驱动链表里面找出驱动，如果找到然后返回fd句柄，与此同时也会执行驱动里面的open函数。

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useIMG_20220507_110949.jpg" alt="IMG_20220507_110949" style="zoom:67%;" />

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useIMG_20220507_085153.jpg" alt="IMG_20220507_085153" style="zoom:67%;" />

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useIMG_20220507_085140.jpg" alt="IMG_20220507_085140" style="zoom:67%;" />





# 交叉编译驱动模块 - 简要（推荐）

### 步骤

将模块 xxx.c 文件放到树莓派内核源码中（推荐放到如图所示的目录中）

> ![Snipaste_2022-06-17_16-56-53](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-56-53.png)

修改Makefile文件，在顶部添加下面内容（obj-m意为将驱动编译成模块） 

> ![Image](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useImage-16394677376565.png)

执行命令开始交叉编译（make modules只生成驱动模块），生成.ko驱动模块

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- KERNEL=kernel7 make modules
```

> ![Snipaste_2022-06-17_16-57-04](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-57-04.png)

使用scp指令拷贝到树莓派中

> ![Snipaste_2022-06-17_16-57-36](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-57-36.png)

lsmod加载驱动模块（`sudo insmod xxx.ko`），可能需要添加权限 `sudo chmod 666 /dev/pin4`

>![Snipaste_2022-06-17_16-58-37](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-58-37.png)

编写驱动测试应用

> ![Snipaste_2022-06-17_16-59-56](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-59-56.png)

通过 demsg 查看内核打印的输出（包括printk函数的输出）

>![Snipaste_2022-06-17_16-33-14](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-33-14.png)
>
>![image-20220617173207528](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220617173207528.png)





# 交叉编译驱动模块 - 详细

### 测试环境

硬件：树莓派3B/3B+
系统：Raspberry Debian 9 / Debian 10

### 准备工作

首先，按本公众号文章《**【树莓派】Linux内核编译**》方法对要使用的内核进行编译及安装。

### 编写驱动源码

创建驱动源码目录, 并新建源码文件和Makefile文件

```
linux@ubuntu:~$ mkdir  first_driver
linux@ubuntu:~$ cd first_driver/
linux@ubuntu:~/first_driver$ touch firstdriv.c  Makefile
```

下面是一个最简单的hello world驱动模块测试程序：

```c
//vi firstdriv.c 
#include <linux/init.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

static int __init driv_init(void)
{
        printk(KERN_INFO "hello world!\n");
        return 0;
}

static void __exit driv_exit(void)
{
        printk(KERN_INFO "bye world!\n");
}

module_init(driv_init);
module_exit(driv_exit);
```

### 编写Makefile文件

```makefile
#vi Makefile
#交叉编译工具, 根据实际使用的编译工具指定， 有可能是arm-none-linux-gnueabi-、arm-eabi-gcc，aarch64-linux-gnu-等等
CROSS_COMPILE := arm-linux-gnueabihf-

CC := $(CROSS_COMPILE)gcc
LD := $(CROSS_COMPILE)ld
 #指定要编译的板子内核为arm 32位版本，如果是64位系统，则修改为arm64
ARCH := arm

obj-m := firstdriv.o

#使用的内核路径
LINUXDIR=/home/linux/linux-rpi-4.14.y

PWD=$(shell pwd)  #当前路径

modules:        
        $(MAKE) ARCH=$(ARCH) -C $(LINUXDIR) M=$(PWD) modules 
#       $(MAKE) -C $(LINUXDIR) M=$(PWD) modules
clean:
        rm -rf *.o *.ko *.mod.* *.symvers *.order

.PHONY: modules clean
```

 **Makefile 中一些变量含义：**
（1）CROSS_COMPILE 指定要使用的交叉编译工具，需要根据实际使用的工具链进行修改，如果是在板子上编译，可以不需要该变量。

（2）ARCH 指定了当前的架构， 如果是直接在arm板子上编译，可以不需要该变量。

（3）LINUXDIR变量便是当前内核的源代码目录， 需要根据自己实际存放位置进行修改。

（4）`make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules`

这句话就是编译驱动模块。（**注意:**这句话最前面的空行其实是一个***Tab键\***，不是空格)

### 编译

运行make命令进行编译

```
linux@ubuntu:~/first_driver$ make
make ARCH=arm   -C /home/linux/linux-rpi-4.14.y M=/home/linux/first_driver   modules 
make[1]: Entering directory '/home/linux/linux-rpi-4.14.y'
  CC [M]  /home/linux/first_driver/firstdriv.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/linux/first_driver/firstdriv.mod.o
  LD [M]  /home/linux/first_driver/firstdriv.ko
make[1]: Leaving directory '/home/linux/linux-rpi-4.14.y'
```

如果没有报错，通过ls命令可以看到目录下生成了ko文件

```
linux@ubuntu:~/first_driver$ ls
firstdriv.c  firstdriv.ko  firstdriv.mod.c  firstdriv.mod.o  firstdriv.o  Makefile  modules.order  Module.symvers
```

如果需要清理，则运行 make clean即可

```
linux@ubuntu:~/first_driver$ make clean
rm -rf *.o *.ko *.mod.* *.symvers *.order
linux@ubuntu:~/first_driver$ ls
firstdriv.c  Makefile
linux@ubuntu:~/first_driver$ 
```

### 测试

#### 1、驱动加载

此时即可将ko文件拷贝树莓派中，然后通过运行 "sudo  insmod firstdriv.ko "命令加载驱动

```
sudo  insmod firstdriv.ko
```

通过dmesg命令，可从内核日志中看到，有"hello world!"字段输出。

#### 2、卸载驱动

通过"sudo rmmod  firstdriv"命令卸载驱动

```
sudo rmmod firstdriv
```

通过"dmesg"命令，可从内核日志中看到，有"bye world!"输出。

(**PS:**以上测试代码及Makfile可在公众号后台回复"**firstdriv**"获取)





# 驱动例程

### 驱动例程 - 简单测试程序

#### driver_test.c

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>

int main()
{
    int fd;
    fd = open("/dev/pin4", O_RDWR);
    if (fd < 0) {
        printf("open failed\n");
        perror("reason:");
    } else {
        printf("open success\n");
    }
    fd = write(fd, '1', 1);  
    close(fd);
}
```

#### first_driver.c

```c
#include <linux/init.h>
#include <linux/module.h>

MODULE_LICENSE("GPL");

static int __init driv_init(void)
{
	printk(KERN_INFO "hello world!\n");
	return 0;
}

static void __exit driv_exit(void)
{
	printk(KERN_INFO "bye world!");
}

module_init(driv_init);
module_exit(driv_exit);
```

#### Makefile

```makefile
#vi Makefile
##交叉编译工具, 根据实际使用的编译工具指定， 有可能是arm-none-linux-gnueabi-、arm-eabi-gcc，aarch64-linux-gnu-等等
CROSS_COMPILE := arm-linux-gnueabihf-

CC := $(CROSS_COMPILE)gcc 
LD := $(CROSS_COMPILE)ld  
 #指定要编译的板子内核为arm 32位版本，如果是64位系统，则修改为arm64
ARCH := arm  

obj-m := firstdriv.o  

#使用的内核路径
LINUXDIR=/home/linux/linux-rpi-4.14.y

PWD=$(shell pwd)  #当前路径

modules:	
	$(MAKE) ARCH=$(ARCH) -C $(LINUXDIR) M=$(PWD) modules 
#       $(MAKE) -C $(LINUXDIR) M=$(PWD) modules
clean:
	rm -rf *.o *.ko *.mod.* *.symvers *.order

.PHONY: modules clean
```





### 驱动框架（推荐）

```c
#include <linux/fs.h>      //file_operations声明
#include <linux/module.h>  //module_init  module_exit声明
#include <linux/init.h>    //__init  __exit 宏定义声明
#include <linux/device.h>  //class  devise声明
#include <linux/uaccess.h> //copy_from_user 的头文件
#include <linux/types.h>   //设备号  dev_t 类型声明
#include <asm/io.h>        //ioremap iounmap的头文件

static struct class *pin4_class;
static struct device *pin4_class_dev;

static dev_t devno;                //设备号
static int major = 231;            //主设备号
static int minor = 0;              //次设备号
static char *module_name = "pin4"; //模块名

/* led_open函数 */
static ssize_t pin4_open(struct inode *inode, struct file *file)
{
    printk("pin4_open\n"); //内核的打印函数和printf类似
    return 0;
}

/* led_read函数 */
static ssize_t pin4_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
    printk("pin4_open\n"); //内核的打印函数和printf类似
    return 0;
}

/* led_write函数 */
static ssize_t pin4_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    printk("pin4_write\n");
    return 0;
}

/* 创建驱动对象 */
static struct file_operations pin4_fops = {
    .owner = THIS_MODULE,
    .open = pin4_open,
    .write = pin4_write,
    .read = pin4_read,
};

/* 真实驱动入口 */
int __init pin4_drv_init(void)  
{
    int ret;
    devno = MKDEV(major, minor);                           //2.创建设备号
    ret = register_chrdev(major, module_name, &pin4_fops); //3.注册驱动  告诉内核，把这个驱动加入到内核驱动的链表中

    //4.创建设备文件，让代码在dev下自动生成设备
    pin4_class = class_create(THIS_MODULE, "myfirstdemo"); 
    pin4_class_dev = device_create(pin4_class, NULL, devno, NULL, module_name); 
    return 0;
}

/* 驱动出口 */
void __exit pin4_drv_exit(void)
{
    device_destroy(pin4_class, devno);
    class_destroy(pin4_class);
    unregister_chrdev(major, module_name); //卸载驱动
}

module_init(pin4_drv_init); //1.入口，内核加载该驱动的时候，这个宏会被展开，然后调用pin4_drv_init
module_exit(pin4_drv_exit);
MODULE_LICENSE("GPL v2");
```





### 驱动例程 - 控制I/O（推荐）

#### 一个bug的解决过程

原本问题是驱动程序和测试程序都能够正确编译，但是运行结果存在问题，

测试应用程序能够正常调用驱动模块，调用write也正常，驱动copy_from_user也能够读取到应用的数据，但是驱动读取到的数据是错误的，1变为了-214602047，0变为了-2146402048

> ![Snipaste_2022-06-17_16-33-14](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-33-14.png)
>
> ![Snipaste_2022-06-17_16-32-41](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_16-32-41.png)

经排查发现是变量未赋初值的问题，修改程序后，重新编译模块，测试正确

> ![Snipaste_2022-06-17_17-06-11](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useSnipaste_2022-06-17_17-06-11.png)

>![image-20220617173147549](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220617173147549.png)
>
>![image-20220617173207528](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220617173207528.png)

#### driver_test.c

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main()
{
    int fd;
    int cmd = 0;
       
    fd = open("/dev/pin4", O_RDWR);
    if (fd < 0) {
        printf("open /dev/pin4 failed!\n");
        perror("reason:");
    } else {
        printf("open /dev/pin4 success!\n");
    }
    printf("Please input the command: 1/0\n1 : set pin4 high\n0 : set pin4 low\n");
    scanf("%d", &cmd);
       
    if (cmd == 1) {   
        printf("You entered : %d\n", cmd);
        fd = write(fd, &cmd, 1);
    } else if (cmd == 0) {
 		printf("You entered : %d\n", cmd);
        fd = write(fd, &cmd, 1);
    } else {
        printf("You entered : %dis error!\n", cmd);
    }
    close(fd);
}
```

#### gpio_control_driver.c

```c
#include <linux/fs.h>      //file_operations声明
#include <linux/module.h>  //module_init  module_exit声明
#include <linux/init.h>    //__init  __exit 宏定义声明
#include <linux/device.h>  //class  devise声明
#include <linux/uaccess.h> //copy_from_user 的头文件
#include <linux/types.h>   //设备号  dev_t 类型声明
#include <asm/io.h>        //ioremap iounmap的头文件

static struct class *pin4_class;
static struct device *pin4_class_dev;

static dev_t devno;                //设备号
static int major = 231;            //主设备号
static int minor = 0;              //次设备号
static char *module_name = "pin4"; //模块名

/* 寄存器 */
volatile unsigned int *GPFSEL0 = NULL;
volatile unsigned int *GPSET0 = NULL;
volatile unsigned int *GPCLR0 = NULL;

/* led_open函数 */
static ssize_t pin4_open(struct inode *inode, struct file *file)
{
    printk("pin4_open\n"); //内核的打印函数和printf类似
    //配置pin4引脚为输出引脚，pin 12-14配置成001
    *GPFSEL0 &= ~(0x6 << 12);  //把bit13-14配置成0
    *GPFSEL0 |= (0x1 << 12);   //把bit12配置成1
    return 0;
}

///* led_read函数 */
//static ssize_t pin4_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
//{
//     printk("pin4_open\n"); //内核的打印函数和printf类似
//     return 0;
//}

/* led_write函数 */
static ssize_t pin4_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
    int user_cmd = 0;  //这里赋值为0非常重要!!!!!
    int n;
    //printk("pin4_write\n");
    // 获取上层write函数的值
    n = copy_from_user(&user_cmd, buf, count);
    if (n != 0) {
        printk("copy form user error\n");
    } else {
        printk("copy form user success\n");
    }
    //根据值来操作IO口，高电平或者低电平
    printk("%d\n", user_cmd);
    if (user_cmd == 1) {
        printk("set pin4 success\n");
        *GPSET0 |= 0x1 << 4;  //将GPSET0寄存器的bit4置1
    } else if (user_cmd == 0) {
        printk("clear pin4 success\n");
        *GPCLR0 |= 0x1 << 4;  //将GPCLR0寄存器的bit4置1
    } else {
        printk("Control pin4 failed\n");
    }
    return 0;
}

/* 创建驱动对象 */
static struct file_operations pin4_fops = {
    .owner = THIS_MODULE,
    .open = pin4_open,
    .write = pin4_write,
    //.read = pin4_read,
};

/* 真实驱动入口 */
int __init pin4_drv_init(void)
{
    int ret;
    printk("Insmod driver pin4 success\n");
    devno = MKDEV(major, minor);                           // 2.创建设备号
    ret = register_chrdev(major, module_name, &pin4_fops); // 3.注册驱动  告诉内核，把这个驱动加入到内核驱动的链表中

    // 4.创建设备文件，让代码在dev下自动生成设备
    pin4_class = class_create(THIS_MODULE, "pin4driver");
    pin4_class_dev = device_create(pin4_class, NULL, devno, NULL, module_name);

    //将寄存器的物理地址转换为虚拟地址，IO口寄存器映射成普通内存单元进行访问
    GPFSEL0 = (volatile unsigned int *) ioremap(0x3f200000, 4);
    GPSET0 = (volatile unsigned int *) ioremap(0x3f20001c, 4);
    GPCLR0 = (volatile unsigned int *) ioremap(0x3f200028, 4);

    return 0;
}

/* 驱动出口 */
void __exit pin4_drv_exit(void)
{
    // 将虚拟地址解绑
    iounmap(GPFSEL0);
    iounmap(GPSET0);
    iounmap(GPCLR0);

    device_destroy(pin4_class, devno);
    class_destroy(pin4_class);
    unregister_chrdev(major, module_name); //卸载驱动
}

module_init(pin4_drv_init); // 1.入口，内核加载该驱动的时候，这个宏会被展开，然后调用pin4_drv_init
module_exit(pin4_drv_exit);
MODULE_LICENSE("GPL v2");
```





### 驱动例程 - LED驱动 - 来自网络

https://blog.csdn.net/yang562887291/article/details/76945391

编写LED驱动前需要移植树莓派内核，以获得内核源码。

**我们在编写驱动程序的时候，IO空间的起始地址是0x3f000000,加上GPIO的偏移量0x2000000,所以GPIO的物理地址应该是从0x3f200000开始的，然后在这个基础上进行Linux系统的MMU内存虚拟化管理，映射到虚拟地址上。**

特别注意，BCM2708 和BCM2709 IO起始地址不同，BCM2708是0x20000000,BCM2709是0x3f000000,这是造成大部分人驱动出现“段错误”的原因。树莓派3B的CPU为BCM2709。

在虚拟机分别编译，将模块、测试程序拷贝至树莓派，插入模块后运行测试程序。

```
sudo insmod xxx.ko

sudo ./led_test 1 打开LED

sudo ./led_test 0 关闭LED
```

#### LED驱动代码

```c
#include <linux/miscdevice.h>    
#include <linux/delay.h>    
#include <asm/irq.h>    
#include <linux/kernel.h>    
#include <linux/module.h>    
#include <linux/init.h>    
#include <linux/mm.h>    
#include <linux/fs.h>    
#include <linux/types.h>    
#include <linux/delay.h>    
#include <linux/moduleparam.h>    
#include <linux/slab.h>    
#include <linux/errno.h>    
#include <linux/ioctl.h>    
#include <linux/cdev.h>    
#include <linux/string.h>    
#include <linux/list.h>    
#include <linux/pci.h>    
#include <asm/uaccess.h>    
#include <asm/atomic.h>    
#include <asm/unistd.h>    
#include <asm/io.h>    
#include <asm/uaccess.h>    
#include <linux/ioport.h>    
      
#define PIN					26 //GPIO26 
  
#define BCM2835_GPSET0                          0x001c  
#define BCM2835_GPFSEL0                         0x0000  
#define BCM2835_GPCLR0                          0x0028  
#define BCM2835_GPIO_FSEL_OUTP      1  
  
#define BCM2835_GPIO_BASE                   0x3f200000  
  
int open_state = 0;         //文件打开状态    
  
    
int bcm2835_gpio_fsel(uint8_t pin, uint8_t mode)    
{    
    //初始化GPIOB功能选择寄存器的物理地址    
    volatile uint32_t * bcm2835_gpio = (volatile uint32_t *)ioremap(BCM2835_GPIO_BASE, 16);    
    volatile uint32_t * bcm2835_gpio_fsel = bcm2835_gpio + BCM2835_GPFSEL0/4 + (pin/10);    
    uint8_t   shift = (pin % 10) * 3;    
    uint32_t  value = mode << shift;    
    *bcm2835_gpio_fsel = *bcm2835_gpio_fsel | value;    
    
    printk("fsel address: 0x%lx : %x\n", (long unsigned int)bcm2835_gpio_fsel, *bcm2835_gpio_fsel);    
    
    return 0;    
}    
    
int bcm2835_gpio_set(uint8_t pin)    
{    
    //GPIO输出功能物理地址    
    volatile uint32_t * bcm2835_gpio = (volatile uint32_t *)ioremap(BCM2835_GPIO_BASE, 16);    
    volatile uint32_t * bcm2835_gpio_set = bcm2835_gpio + BCM2835_GPSET0/4 + pin/32;    
    uint8_t   shift = pin % 32;    
    uint32_t  value = 1 << shift;    
    *bcm2835_gpio_set = *bcm2835_gpio_set | value;    
    
    printk("set address:  0x%lx : %x\n", (long unsigned int)bcm2835_gpio_set, *bcm2835_gpio_set);    
    
    return 0;    
}    
    
int bcm2835_gpio_clr(uint8_t pin)    
{    
   //GPIO清除功能物理地址    
    volatile uint32_t * bcm2835_gpio = (volatile uint32_t *)ioremap(BCM2835_GPIO_BASE, 16);    
    volatile uint32_t * bcm2835_gpio_clr = bcm2835_gpio + BCM2835_GPCLR0/4 + pin/32;    
    uint8_t   shift = pin % 32;    
    uint32_t  value = 1 << shift;    
    *bcm2835_gpio_clr = *bcm2835_gpio_clr | value;    
        
    printk("clr address:  0x%lx : %x\n", (long unsigned int)bcm2835_gpio_clr, *bcm2835_gpio_clr);    
    
    return 0;    
}   
  
static int leds_open(struct inode *inode, struct file *filp)    
{    
    if(open_state == 0)      
    {      
        open_state =  1;      
        printk("Open file suc!\n");      
        return 0;      
    }      
    else      
    {      
        printk("The file has opened!\n");      
        return -1;      
    }      
}    
    
static long leds_ioctl(struct file*filp, unsigned int cmd, unsigned long arg)    
{    
    switch(cmd)      
    {      
        case 0:      
            bcm2835_gpio_clr(PIN);    
            printk("LED OFF!\n");    
            break;      
        case 1:      
            bcm2835_gpio_set(PIN);    
            printk("LED ON!\n");    
            break;      
    
        default:      
            return-EINVAL;      
    }      
    
    return 0;    
}    
    
static int leds_release(struct inode *inode, struct file *filp)    
{    
    if(open_state == 1)      
    {      
        open_state =  0;      
        printk("close file suc!\n");      
        return 0;      
    }      
    else      
    {      
        printk("The file has closed!\n");      
        return -1;      
    }      
}    
    
static const struct file_operations leds_fops = {    
    .owner = THIS_MODULE,    
    .open = leds_open,    
    .unlocked_ioctl = leds_ioctl,    
    .release = leds_release,    
};    
    
static struct miscdevice misc = {    
    .minor =MISC_DYNAMIC_MINOR,    
    .name ="my_leds",    
    .fops =&leds_fops,    
};    
    
    
static int __init leds_init(void)    
{    
    int ret;    
    
    //注册混杂设备    
    ret =misc_register(&misc);    
    
    //配置功能选择寄存器为输出    
    bcm2835_gpio_fsel(PIN, BCM2835_GPIO_FSEL_OUTP);    
    
    //设置输出电平为高电平，LED亮    
    bcm2835_gpio_set(PIN);    
    
    printk("ledsinit.\n");    
    return ret;    
}    
    
static void leds_exit(void)    
{    
    //LED灭    
    bcm2835_gpio_clr(PIN);    
    
    misc_deregister(&misc);            
    
    printk("leds_exit\n");    
}    
    
module_init(leds_init);    
module_exit(leds_exit);    
    
MODULE_AUTHOR("yangwen");    
MODULE_LICENSE("GPL"); 
```

#### 测试代码

```c
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <sys/ioctl.h>  
  
int main(int argc, char **argv)  
{  
    int on;  
    int fd;  
    if (argc != 2 || sscanf(argv[1],"%d", &on) != 1 ||on < 0 || on > 1 ) {  
        fprintf(stderr, "Usage:%s 0|1\n",argv[0]);  
        exit(1);  
    }  
    fd = open("/dev/my_leds", 0);  
    if (fd < 0) {  
        perror("open device leds");  
        exit(1);  
    }  
    /*通过ioctl来控制灯的亮、灭*/  
    if(on){  
        printf("turn on leds!\n");  
        ioctl(fd, 1);  
    }  
    else {  
        printf("turn off leds!\n");  
        ioctl(fd, 0);  
    }  
    close(fd);  
    return 0;  
} 
 
```

#### Makefile

```makefile
CROSS= /home/yangwen/Raspberry/tools-master/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin/arm-linux-gnueabihf-
all: led_test
 
led_test: led_test.c
	$(CROSS)gcc -o $@ led_test.c -static
 
clean:
	@rm -rf led_test *.o
```





# 树莓派高级开发 - “IO口驱动代码的编写“ 包含总线地址、物理_虚拟地址、BCM2835芯片手册知识 - 摘录

https://mp.weixin.qq.com/s/JGKDA_Hmhtvna1m8mBCStw

### 微机总线地址

#### 地址总线：

- **百度百科解释：** 地址总线 (Address Bus；又称：位址总线) 属于一种电脑总线 （一部份），是由CPU 或有DMA 能力的单元，用来沟通这些单元想要存取（读取/写入）电脑内存元件/地方的实体位址。
- **地址总线 = cpu能够访问内存的范围**：用一个现象来解释地址总线：装了32位的win7系统，明明内存条8G，可是系统只识别了3.8G，装了64位，才能识别到8G。**32位能表示/访问 4,294,967,296 bit**

| kbit——mbit——gbit | 差1024        |
| :--------------- | :------------ |
| bit              | 4,294,967,296 |
| kbit             | 4,194,304 K   |
| mbit             | 4,096 M       |
| gbit             | 4 G           |

- **地址总线 = CPU寻找外部的内存单元靠的是地址总线传输的数据**。如果CPU有8根地址总线，每根线上传输0或1，那么传输的数据范围为`00000000~ 11111111`，每一个数值都对应内存中的一个内存单元，所以可以找到编号为`00000000~ 11111111`号的内存单元。如果传输的数据为`00110011`，那么就会找到`00110011`号内存单元，如果传输的数据为`10110111`，那么就会找到`10110111`号内存单元。编号不在`[00000000，11111111]`范围内的CPU就寻找不到，例如`100000000`号内存单元，CPU就寻找不到。**寻址能力就是计算CPU能寻找多少个内存单元**，`00000000~11111111`号内存单元，一共有256个，一个内存单元的大小为**1byte**，这256个内存单元的大小为**256byte**。
- **CPU**是**通过地址总线来指定存储单元**的。
- **==地址总线决定了cpu所能访问的最大内存空间的大小==**。\**eg\**: \**10根地址线\**能**访问的最大的内存为`1024`**（2的10次方）位二进制数据(1B)
- **地址总线是地址线数量之和**。若CPU的地址总线宽度是32位，那么CPU的寻址范围是4G（2的32次方位），所以最多支持4G内存.
- 比如上面我们说的那个现象：装了32位的win7系统，明明内存条8G，可系统只是别了3.8G ，装了64位才能识别到8G。装了32位的操作系统CPU的访问范围是**2^32bit**,就是**4194304kbit**，就是4096Mbit，等于4G。树莓派也是32位 ，一个G的内存，但它只能访问949M剩下的挪作他用。

#### 数据总线：

- CPU通过地址总线寻址，然后通过数据总线与外部设备互换信息。
- 是CPU与内存或其他器件之间的数据传送的通道。
- 数据总线的宽度决定了CPU和外界的数据传送速度。
- 每条传输线一次只能传输1位二进制数据。eg: 8根数据线一次可传送一个8位二进制数据(即一个字节)。
- 数据总线是数据线数量之和，数据总线的位数决定CPU单次通信能交换的信息数量。

#### 数据总线的宽度对CPU的性能的影响：

- 首先，**总线的速度**（即：CPU的主频，CPU的性能指标之一）决定CPU和外设互换信息的速度。
- 其次，**数据总线的宽度**也是表示CPU性能的参数之一（通常，我们说“64位的CPU”是指CPU的数据总线的宽度是64位）。如：64位数据总线的CPU一次就能取出64bit的数据，8位数据总线的CPU一次只能取出8bit的数据，在相同频率的情况下，8位数据总线的CPU就得连续取8次数据，数据量才能和64位数据总线一次取出的数据量相同，单就比较取数据的性能就相差8倍。况且，通常CPU中的寄存器的位数与数据总线的宽度一样，所以在数据处理方面，64位的CPU又比8位的CPU快很多。
- **CPU的地址总线位数和数据总线可以不同**（典型代表就是51单片机），但是一般都相同。16位机有16根数据总线，20根地址总线，能访问1M（2的20次方），32位机有32根数据总线，32根地址总线，能访问4G（2的32次方），64位机确实有64根数据总线。

### 物理地址（PA）

- **百度百科解释：****网卡物理地址存储器中存储单元对应实际地址**称物理地址，与逻辑地址相对应。网卡的物理地址通常是由网卡生产厂家写入网卡的EPROM（一种闪存芯片，通常可以通过程序擦写），它**存储的是传输数据时真正赖以标识发出数据的电脑和接收数据的主机的地址**。
- 这里说的 **物理地址是内存中的内存单元实际地址**，不是外部总线连接的其他电子元件的地址！**==物理地址属于比较好理解的，物理地址就是内存中每个内存单元的编号==**，这个编号是顺序排好的，物理地址的大小决定了内存中有多少个内存单元，物理地址的大小由地址总线的位宽决定!**物理地址是硬件实际地址或绝对地址**

### 虚拟地址（VA）

- 虚拟地址是Windows程序时运行在386保护模式下，这样**程序访问存储器所使用的==逻辑地址`（基于算法的地址[软件层面的地址：假地址]）`称为虚拟地址==\**，与实地址模式下的分段地址类似，\**虚拟地址也可以写为“段：偏移量”的形式**，这里的段是指段选择器。而linux没有各种保护模式，本来用的就是虚拟地址。
- **虚拟地址是CPU保护模式下的一个概念**，保护模式是80286系列和之后的x86兼容CPU操作模式，在CPU引导完操作系统内核后，操作系统内核会进入一种CPU保护模式，也叫虚拟内存管理，在这之后的程序在运行时都处于虚拟内存当中，虚拟内存里的所有地址都是不直接的，所以你有时候可以看到一个虚拟地址对应不同的物理地址，比如A进程里的call函数入口虚拟地址是0x001，而B也是，但是它俩对应的物理地址却是不同的，操作系统采用这种内存管理方法。
- 是防止程序对物理地址写数据造成一些不可必要的问题，比如知道了A进程的物理地址，那么向这个地址写入数据就会造成A进程出现问题，在虚拟内存中运行程序永远不知道自己处于内存中那一段的物理地址上！现在操作系统运行在保护模式下即便知道其他进程的物理地址也不允许向其写入！但是可以通过操作系统留下的后门函数获取该进程上的虚拟地址空间所有控制权限并写入指定数据。
- 虚拟内存管理采用一种拆东墙补西墙的形式，所以虚拟内存的内存会比物理内存要大许多。在进入虚拟模式之前CPU以及Bootloader（BootLoader是在操作系统内核运行之前运行。可以初始化硬件设备、建立内存空间映射图，从而将系统的软硬件环境带到一个合适状态，以便为最终调用操作系统内核准备好正确的环境），操作系统内核均运行在实模式下，直接对物理地址进行操作！虚拟内存中也有分页管理，这种管理方法是为了确保内存中不会出现内存碎片，当操作系统内核初始化完毕内存中的分页表后CPU的分页标志位会被设置，这个分页标志位是给MMU看的！
- **`MMU`是Memory Management Unit的缩写，中文名是内存管理单元，它是==中央处理器（CPU）中用来管理虚拟存储器、物理存储器的控制线路，同时也负责虚拟地址映射为物理地址，以及提供硬件机制的内存访问授权，多用户多进程操作系统==。作用有两点，地址翻译和内存保护。==MMU将虚拟地址翻译为物理地址。==**

**有关各种地址介绍的博文：**

物理地址、虚拟地址、总线地址物理地址和总线地址区别

### 页表（MMU的单元）

**分页管理：**![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640.png)

- 内存分页其实就是我们所说的4G空间，内存的所有内存被操作系统内核以4G为每页划分开，当我们程序运行时会被加载到内存中的4G空间里，其实说是有4G其实并没有真正在的4G空间，4G空间中有一小部分被映射到了物理内存中，或者被映射到了硬盘的文件上（fopen），或者没有被映射，还有一部分在内存当中就会被划分栈，堆，其中有大片大片的内存是没有被映射的，同样物理内存也是被分页了用来与虚拟内存产生映射关系。将虚拟地址映射为物理地址有一个算法（页表）决定了将虚拟地址映射到物理地址的哪个位置，页表是通过MMU（分页内存管理单元）来管理的，就是设计完页表后通过MMU来执行将虚拟地址映射为物理地址。
- 其实真正情况下只有3G用户空间，假如你的内存是4G的那么其中有1G是给操作系统内核使用的，所谓的4G空间只是操作系统基于虚拟内存这种拆东墙补西墙的形式给你一种感觉每个进程都有4G的可用空间一样！这里来说一下拆东墙补西墙，当我们程序被加载进4G空间时其实根本用不了所谓的4G空间，其中有大片内存被闲置，那么这个时候呢，其他程序被加载进来时发现内存不够了，就把其他程序里的4G空间里闲置部分拿出来给这个进程用，换之这个进程内存不够时就会把其他进程里闲置的空间拿过来给该进程使用。银行也是如此！
- 当我们要对物理地址做操作时比如if语句要根据CPU的状态标志寄存器来做不同的跳转，那么这个时候就要对CPU额状态寄存器做操作了就必须知道它的物理地址，内存中有一个电子元件叫MMU负责从操作系统已经初始化好的内存映射表里查询与虚拟地址对应的物理地址并转换，比如mov 0x4h8这个是虚拟地址，当我们要对这个虚拟地址里写数据时那么MMU会先判断CPU的分页状态寄存器里的标志状态是否被设定，如果被设定那么MMU就会捕获这个虚拟地址物理并在操作系统内核初始化好的内存映射表里查询与之对应的物理地址，并将其转换成真正的实际物理地址，然后在对这个实际的物理地址给CPU，在由CPU去执行对应的命令，相反CPU往内存里读数据时比如A进程要读取内存中某个虚拟地址的数据，A进程里的指令给的是虚拟地址，MMU首先会检查CPU的分页状态寄存器标志位是否被设置，如果被设置MMU会捕获这个虚拟地址并将其转换成相应的物理地址然后提交给CPU，在由CPU到内存中去取数据！

更详细的地址问题看这里

### BCM2835芯片手册

下面截取树莓派芯片手册的一张图：![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361421.png)**BCM2835是树莓派3B CPU的型号**，是**ARM-cotexA53架构**，**cpu Bus是地址总线**，`00000000~FFFFFFFF`是**CPU寻址的范围（4G）**。**DMA是高速拷贝单元**，CPU可以发动DMA直接让DMA进行数据拷贝，直接内存访问单元。**物理地址（PA）1G**、**虚拟地址（VA）4G**若程序大于物理地址1G,是不是就跑不了了，不是的，**它有个MMU的单元，把物理地址映射成虚拟地址**，我们操作的代码基本上都是在虚拟地址，它有一个映射页表（上面提及到过)

- 通过芯片手册了解树莓派的GPIO：有54条通用I/O GPIO行，分为两行，备用功能通常是外围IO并且可以在每个银行中出现一个外围设备，以允许灵活地选择IO电压。![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361432.png)
- GPIO有41个寄存器，所有访问都是32位的。
- Description是寄存器的功能描述。**GPFSEL0（寄存器名）**`GPIO Function Select 0`（功能选择：输入或输出）；**GPSET0 （寄存器名）**`GPIO Pin Output Set 0`（将IO口置0）；**GPSET1（寄存器名）**`GPIO Pin Output Set 1`（将IO口置1）；**GPCLR0（寄存器名）**`GPIO Pin Output Clear 0` （清0）下图的地址是：总线地址（并不是真正的物理地址）![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361433.png)
- FSELn表示GPIOn，![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361434.png)下图给出第九个引脚的功能选择示例，对寄存器的29-27进行配置，进而设置相应的功能。根据图片下方的**register 0**表示**0~9**使用的是**register 0**这个寄存器。![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361435.png)
- **输出集寄存器用于设置GPIO管脚**。SET{n}字段定义，分别对GPIO引脚进行设置，将“0”写入字段没有作用。如果GPIO管脚为在输入(默认情况下)中使用，那么SET{n}字段中的值将被忽略。然而，如果引脚随后被定义为输出，那么位将被设置根据上次的设置/清除操作。分离集和明确功能取消对读-修改-写操作的需要。**GPSETn寄存器为了使IO口设置为1，set4位设置第四个引脚，也就是寄存器的第四位。**![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361436.png)
- 输出清除寄存器用于清除GPIO管脚。CLR{n}字段定义要清除各自的GPIO引脚，向字段写入“0”没有作用。如果的在输入(默认)，然后在CLR{n}字段的值是忽略了。然而，如果引脚随后被定义为输出，那么位将被定义为输出根据上次的设置/清除操作进行设置。分隔集与清函数消除了读-修改-写操作的需要。**GPCLRn是清零功能寄存器。**![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361447.png)

配置树莓派的pin4引脚为输出引脚：

```
功能选择 输出/输入（GPIO Function Select Registers）32位
14-12    001    =   GPIO Pin4 is an output
```

只需要将GPFSL0这个寄存器的14~12位设置为001就可以了。只需要将0x6（对应的2进制是110）左移12位·然后取反再与上GPFSL0就可以将13、14这两位配置为0，然后再将0x6（对应2进制110）左移12位，然后或上GPFSL0即可将12位置1。

- 可使用copy_from_user()这个函数在驱动代码里面读取用户输入的指令，使用copy_to_user()这个函数让引脚反馈现在的状态，也就是让用户读取到。
- 

若想找树莓派引脚点这里

### 树莓派IO操控驱动代码

#### ioremap、iounmap：

一. 一般我们的外设都是通过读写设备上的寄存器来进行的，通常包括控制寄存器、状态寄存器、数据寄存器三大类。外设的寄存器通常被连续编址，并且根据CPU的体系架构不同CPU对IO端口的编制方式有两种：

- IO映射方式（IO-mapped）：比较典型的有X86处理器为外设专门实现了一个单独的地址空间，称为“IO端口空间”或者“IO地址空间”，此时CPU可以通过专门的指令（比如X86的IN和OUT）来访问这个“IO端口空间”。
- 内存映射方式（memory-mapped）：RISC指令系统的CPU一般只实现一个物理地址空间，外设IO端口成为内存的一部分。此时CPU可以访问外设的IO端口，就像访问自己的内存一样方便，不必再设置专门的指令来访问。在驱动开发过程中一般使用内存映射方式。

二、 在驱动开发过程中，一般来说外设的IO内存资源的物理地址是已知的，由硬件的设计决定。但是CPU不会为这些已知的外设IO内存资源预先指定虚拟地址的值，所以驱动程序不可以直接就通过外设的物理地址访问到IO内存，而必须要将其映射到虚拟地址空间（通过页表），然后才能根据内核映射过后的虚拟地址来通过内存指令访问这些IO内存，并对其进行操作。

三、 在Linux内核的io.h头文件中声明了ioremap（）函数，用来将IO内存资源映射到核心虚拟地址空间（3Gb~4GB）中，当然不用了可以将其取消映射iounmap（）。这两个函数在mm/ioremap.c文件中：

```
开始映射：void* ioremap（unsigned long phys_addr , unsigned long size , unsigned long flags）
//用map映射一个设备意味着使用户空间的一段地址关联到设备内存上，这使得只要程序在分配的地址范围内进行读取或写入，实际上就是对设备的访问。
第一个参数是映射的起始地址
第二个参数是映射的长度
第二个参数怎么定啊？
====================
这个由你的硬件特性决定。
比如，你只是映射一个32位寄存器，那么长度为4就足够了。
（这里树莓派IO口功能设置寄存器、IO口设置寄存器都是32位寄存器，所以分配四个字节就够了）

比如：GPFSEL0=(volatile unsigned int *)ioremap(0x3f200000,4);
   GPSET0 =(volatile unsigned int *)ioremap(0x3f20001C,4);
      GPCLR0 =(volatile unsigned int *)ioremap(0x3f200028,4);
这三行是设置寄存器的地址,volatile的作用是作为指令关键字
确保本条指令不会因编译器的优化而省略，且要求每次直接读值
ioremap函数将物理地址转换为虚拟地址，IO口寄存器映射成普通内存单元进行访问。
 
解除映射：void iounmap（void* addr）//取消ioremap所映射的IO地址
比如：
     iounmap(GPFSEL0);
        iounmap(GPSET0);
        iounmap(GPCLR0); //卸载驱动时释放地址映射
```

#### 树莓派IO口四的驱动代码：

```
#include <linux/fs.h>            //file_operations声明
#include <linux/module.h>    //module_init  module_exit声明
#include <linux/init.h>      //__init  __exit 宏定义声明
#include <linux/device.h>        //class  devise声明
#include <linux/uaccess.h>   //copy_from_user 的头文件
#include <linux/types.h>     //设备号  dev_t 类型声明
#include <asm/io.h>          //ioremap iounmap的头文件


static struct class *pin4_class;
static struct device *pin4_class_dev;

static dev_t devno;                //设备号
static int major =231;             //主设备号
static int minor =0;               //次设备号
static char *module_name="pin4";   //模块名

volatile unsigned int* GPFSEL0 = NULL;
volatile unsigned int* GPSET0   = NULL;
volatile unsigned int* GPCLR0   = NULL;
//这三行是设置寄存器的地址
//volatile的作用是作为指令关键字，确保本条指令不会因编译器的优化而省略，且要求每次直接读值

//led_open函数
static int pin4_open(struct inode *inode,struct file *file)
{
        printk("pin4_open\n");  //内核的打印函数和printf类似
       
        //配置pin4引脚为输出引脚        
        *GPFSEL0 &=~(0x6 <<12); // 把bit13 、bit14置为0  
        //0x6是110  <<12左移12位 ~取反 &按位与
        *GPFSEL0 |=~(0x1 <<12); //把12置为1   |按位或
        
        return 0;

}
//read函数
static int pin4_read(struct file *file,char __user *buf,size_t count,loff_t *ppos)
{
        printk("pin4_read\n");  //内核的打印函数和printf类似

        return 0;
}

//led_write函数
static ssize_t pin4_write(struct file *file,const char __user *buf,size_t count, loff_t *ppos)
{
        int usercmd;
        printk("pin4_write\n");  //内核的打印函数和printf类似
        
        //获取上层write函数的值                
        copy_from_user(&usercmd,buf,count); //将应用层用户输入的指令读如usercmd里面
        //根据值来操作io口，高电平或者低电平
        if(usercmd == 1){
                printk("set 1\n");
                *GPSET0 |= 0x01 << 4;
        }
        else if(usercmd == 0){
                printk("set 0\n");
                *GPCLR0 |= 0x01 << 4;
        }
        else{
                printk("undo\n");
        }
        return 0;
}

static struct file_operations pin4_fops = {

        .owner = THIS_MODULE,
        .open  = pin4_open,
        .write = pin4_write,
        .read  = pin4_read,
};

//static限定这个结构体的作用，仅仅只在这个文件。
int __init pin4_drv_init(void)   //真实的驱动入口
{
        int ret;
        devno = MKDEV(major,minor);  //创建设备号
        ret   = register_chrdev(major, module_name,&pin4_fops);  //注册驱动  告诉内核，把这个驱动加入到内核驱动的链表中
        pin4_class=class_create(THIS_MODULE,"myfirstdemo");//让代码在dev下自动>生成设备
        pin4_class_dev =device_create(pin4_class,NULL,devno,NULL,module_name);  //创建设备文件
        
        GPFSEL0=(volatile unsigned int *)ioremap(0x3f200000,4);
        GPSET0 =(volatile unsigned int *)ioremap(0x3f20001C,4);
        GPCLR0 =(volatile unsigned int *)ioremap(0x3f200028,4);

        printk("insmod driver pin4 success\n");
        return 0;
}

void __exit pin4_drv_exit(void)
{

        iounmap(GPFSEL0);
        iounmap(GPSET0);
        iounmap(GPCLR0); //卸载驱动时释放地址映射

        device_destroy(pin4_class,devno);
        class_destroy(pin4_class);
        unregister_chrdev(major, module_name);  //卸载驱动
}
module_init(pin4_drv_init);  //入口，内核加载驱动的时候，这个宏会被调用，去调用pin4_drv_init这个函数
module_exit(pin4_drv_exit);
MODULE_LICENSE("GPL v2");
```

#### 1. 设置寄存器的地址

设置寄存器的地址，但是这样写是有问题的，我们上面讲到了在内核里代码和上层代码访问的是虚拟地址（VA），而现在设置的是物理地址，**==必须把物理地址转换成虚拟地址==**

```
//这三行是设置寄存器的地址
volatile unsigned int* GPFSEL0 = volatile (unsigned int *)0x3f200000;
volatile unsigned int* GPSET0  = volatile (unsigned int *)0x3f20001C;
volatile unsigned int* GPCLR0  = volatile (unsigned int *)0x3f200028;
//volatile的作用是作为指令关键字，确保本条指令不会因编译器的优化而省略，且要求每次直接读值
```

我们先把地址初始

```
volatile unsigned int* GPFSEL0 = NULL;
volatile unsigned int* GPSET0   = NULL;
volatile unsigned int* GPCLR0   = NULL;
```

在初始化`int __init pin4_drv_init(void) //真实的驱动入口`里赋值。

> //整数11 //0xb 11 00010001 即便是16进制也是整数，左边是`volatile unsigned int* GPFSEL0` 右边也强制转换成`（volatile unsigned int*）`
>
> **`volatile`的作用是作为指令关键字，确保本条 ==指令不会因编译器的优化而省略==，==且要求每次直接读值==**因为它是地址我希望它是无符号的`unsigned`

我们在编写驱动程序的时候，IO空间的起始地址是`0x3f000000`,加上GPIO的偏移量`0x2000000`,所以GPIO的物理地址应该是从`0x3f200000`开始的![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361448.png)然后在这个基础上进行Linux系统的MMU内存虚拟化管理，映射到虚拟地址上。用到了一个函数`ioremap`

```
//物理地址转换成虚拟地址，io口寄存器映射成普通内存单元进行访问
 GPFSEL0=(volatile unsigned int *)ioremap(0x3f200000,4);
 GPSET0 =(volatile unsigned int *)ioremap(0x3f20001C,4);
 GPCLR0 =(volatile unsigned int *)ioremap(0x3f200028,4);   //4是4个字节
```

#### 2. 配置pin4引脚为输出引脚

![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16546102361449.png)**配置pin4引脚为输出引脚    bit 12-14  配置成001**

```
31 30 ······14 13 12 11 10 9 8 7 6 5 4 3 2 1 
0  0  ······0  0  1  0  0  0 0 0 0 0 0 0 0 0 
 //配置pin4引脚为输出引脚      bit 12-14  配置成001  
  *GPFSEL0 &=~(0x6 <<12); // 把bit13 、bit14置为0  
 //0x6是110  <<12左移12位 ~取反 &按位与
  *GPFSEL0 |=~(0x1 <<12); //把12置为1   |按位或
```

忘记按位与 按位或 点这里

#### 3. 获取上层write函数的值，根据值来操作io口，高电平或者低电平

用`copy_form_user(char *buf , user_buf , count)`获取上层write函数的值

```
int usercmd;
copy_from_user(&usercmd,buf,count); //将应用层用户输入的指令读如usercmd里面
       
 //根据值来操作io口，高电平或者低电平
 printk("get value\n");
        if(usercmd == 1){
                printk("set 1\n");        //置1
                *GPSET0 |= 0x01 << 4;      //用 | 或操作  目的是不影响其他位
                //写1 是让寄存器    开启置1  让bit4为高电平
        }
        else if(usercmd == 0){           
                printk("set 0\n");        //清0
                *GPCLR0 |= 0x01 << 4;      //用 | 或操作  目的是不影响其他位
                //写1 是让清0寄存器 开启置0 让bit4为低电平
        }
        else{
                printk("undo\n");  //提示不支持该指令
        }
```

#### 4. 解除映射

解除映射：`void iounmap（void* addr）；`//取消ioremap所映射的IO地址

```
void __exit pin4_drv_exit(void)
{
        iounmap(GPFSEL0);   //解除映射 GPFSEL0
        iounmap(GPSET0);    //解除映射 GPSET0
        iounmap(GPCLR0);   //解除映射 GPCLR0

        device_destroy(pin4_class,devno);//先销毁设备
        class_destroy(pin4_class);//再销毁类
        unregister_chrdev(major, module_name);  //卸载驱动

}
```

**上层测试代码：**

```
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>
#include <unistd.h>
#include<stdlib.h>
#include<stdio.h>

int main()
{
   int fd;
   int cmd;
   int data;

   fd = open("/dev/pin4",O_RDWR);
   if(fd<0){
           printf("open failed\n");
   }else{
    printf("open success\n");
   }
   
   printf("input commnd:1/0 \n 1:set pin4 high \n 0 :set pin4 low\n");
   scanf("%d",&cmd);

   printf("cmd = %d\n",cmd);
   fd = write(fd, &cmd,4); //cmd类型是int  所以 写4
}
```

#### 驱动卸载

在装完驱动后可以使用指令：`sudo rmmod +驱动名`（不需要写ko）将驱动卸载。

### IO口驱动代码编译

1. 首先在系统目录`/SYSTEM/linux-rpi-4.14.y`下使用指令：`ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- KERNEL=kernel7 make modules`对**驱动模块进行编译**生成`.ko`文件.
2. 然后将编译后的驱动发送到树莓派：`scp ./drivers/char/pin4driver.ko pi@192.168.0.104:/home/pi`，然后再将**上层代码进行编译**：`arm-linux-gnueabihf-gcc pin4test.c -o realtest`，然后再将测试代码传到树莓派：`scp realtest pi@192.168.43.136:/home/pi/`
3. 然后在树莓派上面使用指令：`insmod pin4drive.ko`进行**加载驱动**（然后`lsmod`即可查看到该驱动），
4. 然后使用指令：`sudo chmod 666 /dev/pin4`给予**pin4**这个**设备可访问权限**，还可以在虚拟机上面使用mk5sum查看驱动文件的值，并在树莓派上面使用该指令进行查看该驱动文件的值，看是否一致。
5. `dmesg`查看内核打印的信息，如下图所示：![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-165461023614410.png)
6. 然后**运行测试代码**，在新建一个窗口，使用指令`gpio readall`可以看到`BCM`下面的`4`号引脚模式是**输出模式**，电平是低电平或高电平（根据输入的上层代码而定，输入0就是低电平，输入1就是高电平），这里我输入的是0，如下图所示：![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-165461023614411.png)

### 有关驱动代码里面GPIO口地址的问题

**有关驱动代码里面GPIO口地址的问题：**![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-165461023614412.png)![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-165461023614513.png)

- 7Ennnnn意思是7E00000到7EFFFFFF，F2000000是3F000000映射的虚拟地址，然后7E00000和F200000对应，芯片手册里面使用的是和虚拟地址F200000有着对应关系的地址——7E00000，芯片手册上面地址偏移多少物理地址就偏移多少
- 根据上方图片描述，外设的物理地址范围是m 0x3F000000 to 0x3FFFFFFF，所以你看到的7E200000对应的实际物理地址应该是0x3F000000 + （7E200000-7E000000）
- GPFSEL0=(volatile unsigned int *)ioremap(0x3f200000,4); GPSET0  =(volatile unsigned int *)ioremap(0x3f20001C,4); GPCLR0 =(volatile unsigned int *)ioremap(0x3f200028,4);
- 0x3f200000，0x3f20001C，0x3f200028是物理地址，树莓派的外设空间的起始地址是0x3f000000，根据芯片手册可知对应寄存器的偏移量为,比如GPFSEL0寄存器的实际地址是0x3f200000=0x3F000000  + （7E200000-7E000000）
- 然后通过数据手册可以看到，树莓派相关寄存器的总线地址（和映射的虚拟地址有某种对应关系的地址）进而可得知偏移量，如下图所示：![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-165461023614514.png)





































































































































































































































































































































































































