# qemu调试linux内核环境搭建



参考链接：https://blog.csdn.net/qq_33095733/article/details/123631561



## 1、编译linux内核

从 https://mirrors.edge.kernel.org/pub/linux/kernel/ 下载linux内核，这里我下载的linux-5.16.14.tar.xz。



依次执行以下命令：

```shell
tar -xf linux-5.16.14.tar.xz

cd linux-5.16.14/

export ARCH=x86                #可以不执行这步，会根据主机平台自动选择

make x86_64_defconfig          #也可以复制已有配置config文件来生成对应配置

make menuconfig
```



在配置菜单中，启用内核debug，关闭地址随机化，不然断点处无法停止。

```shell
#开启kernel debug
Kernel hacking  --->
	[*] Kernel debugging
	Compile-time checks and compiler options  --->
		[*] Compile the kernel with debug info
		[*]   Provide GDB scripts for kernel debuggin

#关闭地址随机化
Processor type and features  --->
	[] Randomize the address of the kernel image (KASLR)
```



编译：

```shell
make -j 8
```

编译后生成bzImage文件：arch/x86/boot/bzImage



## 2、编译busybox

从 https://busybox.net/ 下载busybox，这里我下载的busybox-1.35.0.tar.bz2。

依次执行以下命令：

```shell
tar -jxf busybox-1.35.0.tar.bz2

cd busybox-1.35.0/

make menuconfig
```

把busybox配置为静态编译。

```shel
Setting  --->
	[*] Build Busybox as a static binary (no shared libs)
```

然后执行 make 编译。【可能需要root权限】



## 3、制作rootfs

接下来制作rootfs镜像文件，并把busybox安装到其中。



执行以下命令，使用dd命令创建文件，并格式化为ext4文件系统：

```shell
dd if=/dev/zero of=rootfs.img bs=1M count=100   #100M

mkfs.ext4 rootfs.img
```



执行以下命令挂载rootfs.img，并将busybox安装到里面：

```shell
mkdir fs

sudo mount -o loop rootfs.img fs

sudo make install CONFIG_PREFIX=./fs
```

继续执行以下命令：

```shell
cd fs

sudo mkdir dev proc etc home mnt

sudo cp -r ../examples/bootfloppy/etc/* etc/

cd ..

sudo chmod -R 777 fs

sudo umount fs  #取消挂载
```

制作完成的rootfs目录如下：

```shell
test@hello:~/xxxx/busybox-1.35.0$ ls fs/
bin  dev  etc  home  linuxrc  lost+found  mnt  proc  sbin  usr
```

至此，一个带有rootfs的磁盘镜像制作完成。



**注：以上制作的rootfs中没有lib、lib64等目录，因此编写的应用程序需要静态编译才能在里面运行。**

​        **如果需要以动态链接的方式编译应用程序运行，应该需要将宿主机的lib、lib64目录下的内容复制到rootfs.img中。【暂未验证】**



## 4、用qemu启动linux内核

### 4.1 安装qemu

安装qemu有两种方式：

1、大多数发行版linux都可以直接安装qemu，具体这里不介绍了。

2、下载qemu源码编译安装，暂忽略。

### 4.2 启动qemu

使用如下命令启动没有GUI的qemu：

```shell
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -hda rootfs.img --append "root=/dev/sda console=ttyS0" -nographic
```

参数解释：

-kernel：指定编译好的内核镜像

-hda：指定硬盘

-append “root=/dev/sda”：指示根文件系统 

console=ttyS0：把QEMU的输入输出定向到当前终端上

-nographic：不使用图形输出窗口

**注：另外也可以添加vnc参数，就可以使用vnc客户端来连接虚拟机**



启动后如下图：

![img](C:\Users\zhuobx\AppData\Local\Temp\企业微信截图_16564846323823.png)

这时候根文件系统是以只读模式挂载的，可以执行 mount -o remount rw / 重新以读写模式挂载。

退出qemu使用 **ctrl+A，再按x**。



##### 调试内核

若要调试内核，需要以如下命令启动：

```shell
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -hda rootfs.img --append "root=/dev/sda console=ttyS0" -s -S -smp 1 -nographic
```

新增了 -s -S -smp 1参数：

-S：让系统暂停

-s：是-gdb tcp::1234缩写，监听1234端口，在GDB中可以通过target remote localhost:1234连接

-smp 1：指定一个CPU，这个参数不是必须的



以调试方式启动qemu后linux内核会暂停，这时候在另一个终端中使用gdb连接。

```shel
xxx$ gdb

(gdb) file linux-5.16.14/vmlinux    #加载linux内核符号

(gdb) target remote localhost:1234  #连接gdb服务

(gdb) c  #让linux内核继续运行
```

![img](C:\Users\zhuobx\AppData\Local\Temp\企业微信截图_16564852959734.png)



##### 下断点调试内核函数

如给copy_process函数下断点：

```shell
(gdb) b copy_process
Breakpoint 2 at 0xffffffff81064290: file kernel/fork.c, line 1932.
(gdb) c
Continuing.
```

然后在qemu虚拟机中输入ls，这时就会断在copy_process函数。

![img](C:\Users\zhuobx\AppData\Local\Temp\企业微信截图_1656485783658.png)

![img](C:\Users\zhuobx\AppData\Local\Temp\企业微信截图_16564854924521.png)

## 5、添加共享磁盘

有时候需要在宿主机和qemu虚拟机之间共享文件，添加一个共享磁盘将有助于该项工作。

**注：这种方式添加的共享磁盘不是实时的，每次有文件增删改都需要重启虚拟机、重新挂载才生效。**



创建100MB磁盘镜像文件，并格式化为ext4，作为共享磁盘备用。

```shell
dd if=/dev/zero of=share.img bs=1M count=100

mkfs.ext4 share.img

mkdir share

sudo mount -o loop share.img share  #挂载

sudo cp test share/   #复制test文件到share.img中，test是我写的一个程序，打印hello world，静态编译的。

sudo umount share  #取消挂载
```



然后修改qemu启动命令，增加-hdb参数添加一个磁盘：

```shell
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -hda rootfs.img --hdb share.img --append "root=/dev/sda console=ttyS0" -s -S -smp 1 -nographic
```

启动后在qemu虚拟机中挂载/dev/sdb到mnt目录：

```shell
/ # mount /dev/sdb mnt    #挂载共享磁盘
[   60.081712] EXT4-fs (sdb): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
[   60.083665] ext4 filesystem being mounted at /mnt supports timestamps until 2038 (0x7fffffff)
/ #
/ # ls mnt/      #可以看到共享磁盘中的test程序
lost+found  test
/ #
/ # ./mnt/test   #执行test程序，打印出hello world
hello world
[   85.495545] test (81) used greatest stack depth: 13512 bytes left
/ #
```

