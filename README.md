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
   * 根据这篇博客，下次2.4版本后的zImage和dtb文件即可



## LINUX驱动模块框架编写

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

    > 可以使用命令 `cat /proc/deveices ` 查看当前驱动主设备号

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



## LINUX驱动模块加载

1. 加载新模块驱动时需要先执行

    ~~~shell
    depmod
    ~~~

2. 然后加载驱动

    ~~~shell
    modprobe led.ko
    ~~~

3. 然后在设备树下设置设备号和子设备数量

    ~~~shell
    mknod  /dev/led c 200 0
    ~~~

4. 卸载驱动

    ~~~
    rmmod led.ko
    ~~~

    
