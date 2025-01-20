# 嵌入式Linux学习

## vscode插件安装



1. `C/C++`,这个肯定是必须的。
2. `C/C++Snippets`,即C/C++重用代码块。
3. `C/C++Advanced Lint`,即C/C++静态检测
4. `Code Runner`,即代码运行。
5. `Include AutoComplete`,即自动头文件包含。
6. `Rainbow Brackets`,彩虹花括号,有助于阅读代码。
7. `One Dark Pro`,VSCode的主题。
8. `GBKtoUTF8,`将GBK转换为UTF8。
9. `ARM`,即支持ARM汇编语法高亮显示。
10. `Chinese(Simplified)`,即中文环境。
11. `vscode-icons,`VSCode图标插件,主要是资源管理器
12. `compareit`,比较插件,可以用于比较两个文件的差异
13. `DeviceTree`,设备树语法插件。

## imx6u和虚拟机网络配置

`开发板IP：`192.168.10.50  

`虚拟机IP：`192.168.10.100  

`电脑网口的IP：`192.168.10.200



1. 设置开发板ip

~~~shell
ifconfig eth0 192.168.10.50
~~~

2. 重启网卡

~~~shell
/etc/init.d/networking restart
~~~

3. 挂载网络文件系统

~~~shell
mount -t nfs -o nolock,nfsvers=3 192.168.10.100:/home/zc/linux/nfs get/ 
~~~





### uboot挂载设备树

1. 设置网络环境变量

~~~shell
env default -a;saveenv 
setenv ipaddr 192.168.10.50 
setenv ethaddr 00:0C:29:9C:86:28 
setenv gatewayip 192.168.10.1 
setenv netmask 255.255.255.0 
setenv serverip 192.168.10.100 
saveenv 
~~~

2. 设置bootcmd和bootargs

   ~~~shell
   setenv bootcmd 'tftp 80800000 zImage;tftp 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000'
   ~~~

   ~~~
   setenv bootargs 'console=ttymxc0,115200 rw nfsroot=192.168.10.100:/home/zc/linux/nfs/rootfs ip=192.168.10.50:192.168.10.100:192.168.10.1:255.255.255.0::eth0:0ff'
   ~~~

3. 遇到问题

   ~~~
   Starting kernel ...
   
   Booting Linux on physical CPU 0x0
   ...
   can: raw protocol (rev 20120528)
   can: broadcast manager protocol (rev 20120528 t)
   can: netlink gateway (rev 20130117) max_hops=1
   lib80211: common routines for IEEE802.11 drivers
   Key type dns_resolver registered
   Registering SWP/SWPB emulation handler
   snvs_rtc 20cc000.snvs:snvs-rtc-lp: setting system clock to 2023-03-26 19:57:45 UTC (1679860665)
   IP-Config: Failed to open eth0
   IP-Config: Device `eth0' not found
   
   无法挂载根文件系统解决
   ~~~

4. 如何解决
   * 怀疑是kernel编译的时候内核配置自动网关dchp导致nfs无法正常工作
   * 根据这篇博客，下载2.4版本后的zImage和dtb文件即可



## LINUX字符型设备驱动模块框架编写

### 步骤

1. 包含头文件

    ~~~c
    #include <linux/module.h>
    #include <linux/kernel.h>
    #include <linux/init.h>
    #include <linux/fs.h>
    
    #include <linux/slab.h>
    #include <linux/uaccess.h>
    #include <linux/io.h>
    ~~~

2. 定义驱动的名称及主设备号

    ~~~c
    #define LED_MAJOR       200 //主设备号
    #define LED_NAME        "LED"//名字
    ~~~

    > 可以使用命令 `cat /proc/devices ` 查看当前驱动主设备号

3. 编写init和exit函数，顺便标记作者和证书

    ~~~c
    module_init(LED_init);
    module_exit(LED_exit);
    
    MODULE_LICENSE("GPL");
    MODULE_AUTHOR("zc");
    ~~~

    1. init函数

        ~~~c
        //入口
        static int __init LED_init(void)
        {
            //注册
            int ret = register_chrdev(0, LED_NAME, &LED_fops);
            if (ret < 0)    
            {
                printk("regist failed!");
                return -EIO;//LINUX内部定义的错误类型
            }
            
            printk("LED init access!\r\n");
            return 0;
        }
        ~~~

        

    2. exit函数

        ~~~c
        //出口
        static int __exit LED_exit(void)
        {
            printk("LED exit access!\r\n");
            return 0;
        }
        ~~~


4. 编写**字符设备操作集**`fops`

   ~~~
   static const struct file_operations cm4000_fops = {
   	.owner	= THIS_MODULE,
   	.read	= cmm_read,
   	.write	= cmm_write,
   	.unlocked_ioctl	= cmm_ioctl,
   	.open	= cmm_open,
   	.release= cmm_close,
   	.llseek = no_llseek,
   };
   ~~~

   > 可根据需求进行删减 例如：

	~~~c
	/*字符设备操作集*/
	static const struct file_operations LED_fops = {
		.owner	= THIS_MODULE,
		.write	= LED_write,
		.open	= LED_open,
		.release= LED_close,
	};

5. 编写驱动函数：

~~~c
static ssize_t LED_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    return 0;
}
static int LED_open(struct inode *inode, struct file *filp)
{
    return 0; 
}

static int LED_close(struct inode *inode, struct file *filp)
{
    return 0;
}
~~~

### 完整框架代码

~~~c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>

#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/io.h>

#define LED_MAJOR       200 //主设备号
#define LED_NAME        "LED"//名字


static ssize_t LED_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
    return 0;
}
static int LED_open(struct inode *inode, struct file *filp)
{
    return 0; 
}

static int LED_close(struct inode *inode, struct file *filp)
{
    return 0;
}

/*字符设备操作集*/
static const struct file_operations LED_fops = {
	.owner	= THIS_MODULE,
	.write	= LED_write,
	.open	= LED_open,
	.release= LED_close,
};

//入口
static int __init LED_init(void)
{
    //注册
    int ret = register_chrdev(0, LED_NAME, &LED_fops);
    if (ret < 0)    
    {
        printk("regist failed!");
        return -EIO;
    }
    
    printk("LED init access!\r\n");
    return 0;
}

//出口
static int __exit LED_exit(void)
{
    printk("LED exit access!\r\n");
    return 0;
}


module_init(LED_init);
module_exit(LED_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zc");
~~~



## LINUX字符型设备驱动模块加载

1. 加载新模块驱动时需要先执行

    ~~~shell
    depmod
    ~~~

2. 然后加载驱动

    ~~~shell
    modprobe led.ko
    ~~~

3. 然后手动创建设备节点

    ~~~shell
    mknod  /dev/led c 200 0
    ~~~

4. 卸载驱动

    ~~~
    rmmod led.ko
    ~~~

    

## `新`LINUX字符型设备驱动模块框架编写

### 区别：

1. 在驱动模块加载的时候自动创建设备节点文件  

   >device_create

2. 不用事先查询设备号直接向Linux申请即可

   >int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)  

### 步骤：

#### 1. 分配和释放设备号

   ~~~c
   struct LED_dv
   {
       int major;       // 主设备号（主设备标识符）
       int minor;       // 次设备号（次设备标识符）
       dev_t devid;     // 设备号，包含主设备号和次设备号
   };
   
   // 检查是否已指定主设备号（LED.major）
   if (LED.major)  
   {
       // 如果主设备号已经指定（LED.major 非零），则使用该主设备号
       LED.devid = MKDEV(LED.major, 0);  // 使用主设备号和次设备号（次设备号设为0）生成设备号
       ret = register_chrdev_region(LED.devid, LED_COUNT, LED_NAME);  // 注册设备号范围（LED_COUNT个设备）
   } 
   else 
   {
       // 如果没有指定主设备号（LED.major为0），动态分配一个设备号
       ret = alloc_chrdev_region(&LED.devid, 0, LED_COUNT, LED_NAME);  // 分配设备号，返回分配的设备号到LED.devid
       LED.major = MAJOR(LED.devid);  // 从设备号中提取主设备号
       LED.minor = MINOR(LED.devid);  // 从设备号中提取次设备号
   }
   
   // 检查设备号注册是否成功
   if (ret < 0)    
   {
       printk("regist failed!");  // 如果设备号注册失败，打印错误信息
       return -EIO;  // 返回错误代码EIO（输入输出错误）
   }
   
   //注销设备号
   unregister_chrdev_region(LED.devid, LED_COUNT);
   ~~~

   

#### 2. 新的字符设备注册方法

   1. 定义结构体cdev，

      ~~~c
      /*
       * @brief: 该结构体表示一个字符设备的核心信息。
       *         它包含了字符设备的操作函数、设备号、引用计数等， 
       *         是 Linux 内核字符设备管理的重要部分。
       *
       * @fields:
       *  - kobj: 内核对象，支持 sysfs 接口，管理设备属性。
       *  - owner: 指向设备所属模块的指针，防止模块在设备使用时被卸载。
       *  - ops: 字符设备的操作函数，定义了设备的行为，如 open、read、write 等。
       *  - list: 用于将多个 cdev 结构体按链表组织，便于设备的管理和遍历。
       *  - dev: 设备号（主设备号和次设备号），唯一标识一个字符设备。
       *  - count: 引用计数，记录设备的使用次数，确保设备在使用时不会被卸载。
       */
      struct cdev {
          struct kobject kobj;              /* 内核对象，支持 sysfs 接口 */
          struct module *owner;             /* 所属模块，防止设备使用时卸载 */
          const struct file_operations *ops; /* 设备操作函数集 */
          struct list_head list;            /* 链表，用于组织多个字符设备 */
          dev_t dev;                        /* 设备号（主设备号 + 次设备号） */
          unsigned int count;               /* 引用计数，管理设备生命周期 */
      };
      ~~~

      >在 cdev 中有两个重要的成员变量： ops 和 dev，这两个就是字符设备文件操作函数集合file_operations 以及设备号 dev_t。编写字符设备驱动之前需要定义一个 cdev 结构体变量，这个变量就表示一个字符设备，如下所示：  
		~~~c
		struct LED_dv
		{
		    int major;       // 主设备号（主设备标识符）
		    int minor;       // 次设备号（次设备标识符）
		    dev_t devid;     // 设备号，包含主设备号和次设备号
		    struct cdev cdev; // 字符设备的核心结构，包含设备操作函数
		};
		~~~
	
	2. 定义好 `cdev` 变量以后就要使用 `cdev_init` 函数对其进行初始化
	
	   ~~~c
	   void cdev_init(struct cdev *cdev, const struct file_operations *fops)
	   ~~~
	
	3. 然后使用`cdev_add` 函数用于向 Linux 系统添加字符设备
	
	   ~~~c
	   int cdev_add(struct cdev *p, dev_t dev, unsigned count)
	   
	   //example
	   cdev_init(&LED.cdev, &LED_fops);
	   ret = cdev_add(&LED.cdev,LED.devid, LED_COUNT);
	   ~~~
	
	4. 最后退出的时候一定要使用 `cdev_del` 函数从Linux内核中删除相应的字符设备  
	
	   ~~~c
	   cdev_del(&testcdev); /* 删除 cdev */
	   ~~~

#### 3.自动创建设备节点

> 在 Linux 系统中，`mdev` 是一个简化版的 `udev`，主要用于嵌入式系统中实现设备文件的自动创建和删除。当我们使用 `modprobe` 加载驱动模块时，`mdev` 会自动在 `/dev` 目录下创建对应的设备文件，而当我们卸载驱动模块时，它会删除相应的设备文件。这样，无需手动使用 `mknod` 命令创建设备节点，`mdev` 会根据硬件设备的变化自动管理设备文件。通过设置 `/proc/sys/kernel/hotplug`，系统会将热插拔事件交给 `mdev` 处理，从而实现设备节点的动态管理。

1. 创建和删除类 `class_create`  参数 owner 一般为 THIS_MODULE  

   ~~~c
   struct class *class_create (struct module *owner, const char *name)
   ~~~

2. 创建设备  `device_create`

   ~~~c
   /*
    * @brief: 创建一个设备，并在系统中注册该设备。
    * 
    * @param cls: 指向设备类别的指针，表示该设备属于哪个类别。
    * @param parent: 指向父设备的指针，如果该设备没有父设备，可以传递 NULL。
    * @param devt: 设备号，通常是通过 `MKDEV` 宏生成的设备号（主设备号和次设备号）。
    * @param drvdata: 驱动程序特定的数据，可以是任何类型的指针，驱动程序可以使用该数据来存储与设备相关的信息。
    * @param fmt: 设备的名称格式字符串，该字符串用于生成设备文件的名称（例如，`/dev/led0`）。
    * @param ...: 可变参数，用于生成设备名称时使用的具体信息（例如设备的编号）。
    * 
    * @return: 成功时返回指向 `struct device` 的指针，失败时返回 `NULL`。
    */
   struct device *device_create(struct class *cls, struct device *parent,
   			     dev_t devt, void *drvdata,
   			     const char *fmt, ...);
   example：
   LED.device = device_create(LED.class,NULL,LED.devid,NULL,LED_NAME);
   ~~~
3. 删除类 ` device_destroy`

   ~~~c
   void device_destroy(struct class *class, dev_t devt)
   ~~~
4. 删除类 `class_destroy`

   ~~~c
   void class_destroy(struct class *cls);
   ~~~

### 完整框架代码

~~~c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>

#define LED_NAME        "LED"//名字
#define LED_COUNT        1  //子设备数
#define LEDOFF 					0			/* 关灯 */
#define LEDON 					1			/* 开灯 */
 
/* 寄存器物理地址 */
#define CCM_CCGR1_BASE				(0X020C406C)	
#define SW_MUX_GPIO1_IO03_BASE		(0X020E0068)
#define SW_PAD_GPIO1_IO03_BASE		(0X020E02F4)
#define GPIO1_DR_BASE				(0X0209C000)
#define GPIO1_GDIR_BASE				(0X0209C004)

/* 映射后的寄存器虚拟地址指针 */
static void __iomem *IMX6U_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

void led_switch(u8 sta)
{
	u32 val = 0;
	if(sta == LEDON) {
		val = readl(GPIO1_DR);
		val &= ~(1 << 3);	
		writel(val, GPIO1_DR);
	}else if(sta == LEDOFF) {
		val = readl(GPIO1_DR);
		val|= (1 << 3);	
		writel(val, GPIO1_DR);
	}	
}
struct LED_dv
{
    int major;       //主设备号
    int minor;      //次设备号
    dev_t devid;    //设备号
    struct cdev cdev;
	struct class *class;
	struct device *device;
};
struct LED_dv LED;

/*
 * @description		: 向设备写数据 
 * @param - filp 	: 设备文件，表示打开的文件描述符
 * @param - buf 	: 要写给设备写入的数据
 * @param - cnt 	: 要写入的数据长度
 * @param - offt 	: 相对于文件首地址的偏移
 * @return 			: 写入的字节数，如果为负值，表示写入失败
 */
static ssize_t LED_write(struct file *filp, const char __user *buf,
			 size_t count, loff_t *ppos)
{
	printk("LED_write enter!\r\n");
	int retvalue;
	unsigned char databuf[1];
	unsigned char ledstat;

	retvalue = copy_from_user(databuf, buf, count);
	if(retvalue < 0) {
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}

	ledstat = databuf[0];		/* 获取状态值 */

	if(ledstat == LEDON) {	
		printk("led on!\r\n");
		led_switch(LEDON);		/* 打开LED灯 */
	} else if(ledstat == LEDOFF) {
		printk("led off!\r\n");
		led_switch(LEDOFF);	/* 关闭LED灯 */
	}
	return 0;
}

static int LED_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &LED;	/* 设置私有数据 */
    return 0;
}

static int LED_close(struct inode *inode, struct file *filp)
{
    return 0;
}

/*字符设备操作集*/
static const struct file_operations LED_fops = {
	.owner	= THIS_MODULE,
	.write	= LED_write,
	.open	= LED_open,
	.release= LED_close,
};



//入口
static int __init LED_init(void)
{
    u32 val = 0;

	/* 初始化LED */
	/* 1、寄存器地址映射 */
  	IMX6U_CCM_CCGR1 = ioremap(CCM_CCGR1_BASE, 4);
	SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
  	SW_PAD_GPIO1_IO03 = ioremap(SW_PAD_GPIO1_IO03_BASE, 4);
	GPIO1_DR = ioremap(GPIO1_DR_BASE, 4);
	GPIO1_GDIR = ioremap(GPIO1_GDIR_BASE, 4);

	/* 2、使能GPIO1时钟 */
	val = readl(IMX6U_CCM_CCGR1);
	val &= ~(3 << 26);	/* 清楚以前的设置 */
	val |= (3 << 26);	/* 设置新值 */
	writel(val, IMX6U_CCM_CCGR1);

	/* 3、设置GPIO1_IO03的复用功能，将其复用为
	 *    GPIO1_IO03，最后设置IO属性。
	 */
	writel(5, SW_MUX_GPIO1_IO03);
	
	/*寄存器SW_PAD_GPIO1_IO03设置IO属性
	 *bit 16:0 HYS关闭
	 *bit [15:14]: 00 默认下拉
     *bit [13]: 0 kepper功能
     *bit [12]: 1 pull/keeper使能
     *bit [11]: 0 关闭开路输出
     *bit [7:6]: 10 速度100Mhz
     *bit [5:3]: 110 R0/6驱动能力
     *bit [0]: 0 低转换率
	 */
	writel(0x10B0, SW_PAD_GPIO1_IO03);

	/* 4、设置GPIO1_IO03为输出功能 */
	val = readl(GPIO1_GDIR);
	val &= ~(1 << 3);	/* 清除以前的设置 */
	val |= (1 << 3);	/* 设置为输出 */
	writel(val, GPIO1_GDIR);

	/* 5、默认关闭LED */
	val = readl(GPIO1_DR);
	val |= (1 << 3);	
	writel(val, GPIO1_DR);

    int ret = 0;
    //注册设备号
    if (LED.major)  
    {
        LED.devid = MKDEV(LED.major,0);
        ret = register_chrdev_region(LED.devid,LED_COUNT,LED_NAME);
    }else
    {
        ret = alloc_chrdev_region(&LED.devid,0,LED_COUNT,LED_NAME);
        LED.major = MAJOR(LED.devid);
        LED.minor = MINOR(LED.devid);
    }
    if (ret < 0)    
    {
        printk("regist failed!"); 
        return -EIO;
    }
    

    //cdev初始化
    cdev_init(&LED.cdev, &LED_fops);
    ret = cdev_add(&LED.cdev,LED.devid, LED_COUNT);
    
	/* 4、创建类 */
	LED.class = class_create(THIS_MODULE, LED_NAME);
	if (IS_ERR(LED.class)) {
		return PTR_ERR(LED.class);
	}

	//创建device
	LED.device = device_create(LED.class,NULL,LED.devid,NULL,LED_NAME);
	if (IS_ERR(LED.device)) {
		return PTR_ERR(LED.device);
	}

    printk("LED major is %d ,minor is %d !\r\n",LED.major,LED.minor);
    return 0;
}

//出口
static int __exit LED_exit(void)    
{
	/* 取消映射 */
	iounmap(IMX6U_CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_DR);
	iounmap(GPIO1_GDIR);

    //注销cdev
    cdev_del(&LED.cdev);
	//注销设备号
	unregister_chrdev_region(LED.devid,LED_COUNT);
	//注销device
	device_destroy(LED.class, LED.devid);
	//注销class
	class_destroy(LED.class);
    printk("LED exit access!\r\n");
    return 0;
}

//


module_init(LED_init);
module_exit(LED_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("zc");
~~~

### 问题

#### 报错：

~~~c
/lib/modules/4.1.15 # modprobe led.ko
led: disagrees about version of symbol device_create
led: Unknown symbol device_create (err -22)
led: disagrees about version of symbol device_destroy
led: Unknown symbol device_destroy (err -22)
led: disagrees about version of symbol device_create
led: Unknown symbol device_create (err -22)
led: disagrees about version of symbol device_destroy
led: Unknown symbol device_destroy (err -22)
modprobe: can't load module led.ko (led.ko): Invalid argument
~~~

#### 问题原因：

>内核版本与模块版本不一致造成

#### 问题分析：

>1. 之前加载根文件系统的时候，因为一直无法通过网络挂载根文件系统，查询解决方法是将kernel内核更换为2.4版本之后的Zimage
>2. 所以导致现在imx6ull开发板上的内核版本是2.4版本后，而编写驱动加载用的linux内核是2.4版本之前的
>3. 因为2.4版本之前的Zimage无法挂载网络根文件系统，所以现在只需将编写驱动加载用的linux内核也更改为2.4版本之后的就行

#### 解决步骤：

1.打开正点原子资料：

~~~
D:\学习\嵌入式linux\开发板资料\【正点原子】阿尔法Linux开发板（A盘）-基础资料\01、例程源码\01、例程源码\10、开发板教程对应的uboot和linux源码\02、linux\02、V2.4版本及以后版本底板使用的linux\linux-imx-rel_imx_4.1.15_2.1.1_ga_alientek_v2.4.tar.bz2
~~~

2. 将该linux原码复制到ubuntu中（FileZilla）

~~~shell
/home/zc/linux/IMX6ULL/linux/2.4
~~~

3.解压缩代码（建议创建一个文件夹 ./linux-imx-rel_imx_4.1.15_2.1.1_ga_alientek_v2.4 把这个加在后面）

~~~shell
tar -xjf linux-imx-rel_imx_4.1.15_2.1.1_ga_alientek_v2.4.tar.bz2  
~~~

3. 编译linux内核

~~~shll
./imx6ull_alientek_emmc.sh
~~~

4. 将生成的ZImage，和dtb文件挂载到TFTP中

   1. dtb路径

      ~~~
      2.4/arch/arm/boot/dts
      ~~~

   2. ZImage路径

      ~~~
      2.4/arch/arm/boot
      ~~~

5. 在Makefile和Json中修改路径

   ~~~makefile
   //Makefile
   KERNELDIR := /home/zc/linux/IMX6ULL/linux/2.4  //修改这里
   
   CURRENT_PAYH := $(shell pwd)
   
   obj-m := led.o
   
   all: app kernel_modules
   
   kernel_modules:
   	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PAYH) modules;
   	sudo cp led.ko  ledAPP /home/zc/linux/nfs/rootfs/lib/modules/4.1.15/ -f
   
   app:
   	arm-linux-gnueabihf-gcc ledAPP.c -o ledAPP
   
   clean:
   	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PAYH) clean
   	rm -f ledAPP
   
   //json
   {
       "configurations": [
           {
               "name": "Linux",
               "includePath": [
                   "${workspaceFolder}/**",
                   "/home/zc/linux/IMX6ULL/linux/2.4/include", 
                   "/home/zc/linux/IMX6ULL/linux/2.4/arch/arm/include", 
                   "/home/zc/linux/IMX6ULL/linux/2.4/arch/arm/include/generated/"
               ],
               "defines": [],
               "compilerPath": "/usr/bin/clang",
               "cStandard": "c11",
               "cppStandard": "c++17",
               "intelliSenseMode": "clang-x64"
           }
       ],
       "version": 4
   }
   ~~~

   
