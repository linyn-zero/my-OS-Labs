# 中山大学本科生实验报告
实验课程：  操作系统原理
实验名称：  lab1
专业名称：  计算机科学与技术
学生姓名：  黄鑫
学生学号：  21307008
报告时间：  2023/3/3

### 实验要求
⽬标：熟悉开发操作系统的环境和⼯具，熟悉Linux 内核的编译、加载运⾏的过程，熟悉qemu模拟器的使⽤⽅法。
1. 完成Linux内核编译；
2. 完成initramfs的制作过程
3. 完成内核的装载和启动过程
4. 完成Busybox的编译、启动过程
5. 完成Busybox的远程调试
6. 完成Linux 0.11内核的编译、启动和调试

主要指令已在指导手册里给出，重点在于理解指令中各部分行为、对象与参数的含义。

### 实验过程
#### 完成Linux内核编译
1. 下载并解压Linux-5-10-170后，将内核编译settings设置为i386的32位版本。
```shell
mkdir ~/lab1
cd ~/lab1
xz -d linux-5.10.170.tar.xz
tar -xvf linux-5.10.170.tar
cd linux-5.10.170
make i386_defconfig
make menuconfig
```
在打开的图像界面中依次选择`Kernel hacking`、`Compile-time checks and compiler options`，最后在`[ ] Compile the kernel with debug info`输入`Y`勾选，保存退出。

2. 编译内核
```shell
make -j8
```

#### 完成内核的装载和启动过程
1. 启动内核
```shell
cd ~/lab1
qemu-system-i386 -kernel linux-5.10.170/arch/x86/boot/bzImage -s -S -append "console=ttyS0" -nographic
``` 
- `qemu-system-i386`是我们提前下载好的qemu对应版本的intel-x80386操作系统模拟器
- `-kernel`参数指定要引导的内核镜像路径
- `linux-5.10.169/arch/x86/boot/bzImage`是我刚才编译得到的Linux内核压缩镜像
- `-s`启动qemu的gdb服务(gdbserver) 
- `-S`将linux内核执行时悬挂（suspend），等待gdb连接
- `-append`将指定字符串作为内核的命令行参数传递给内核
- `console=ttyS0`的作用是使内核的控制台输出重定向到串行（sequential）端口0，以便我们的qemu可以通过该端口查看内核的输出
- `tty`表示终端（terminal）端口，但是缩写源自teletypewriters（电报打字机）
- `-nographic`是取消qemu的GUI界面
  
2. gdb调试
   
在另外一个Terminal下启动gdb，注意，不要关闭qemu所在的Terminal。
```shell
gdb
```
在gdb下，加载符号表
```shell
file linux-5.10.169/vmlinux
```
在gdb下，连接已经启动的qemu进行调试。
```shell
target remote:1234
```
在gdb下，为start_kernel函数设置断点(break)。
```shell
b start_kernel
```
在gdb下，输入`c`(continue)运行。
```
c
```

#### 完成initramfs的制作过程
1. helloworld.c

在`lab1`文件夹下操作。
```c
#include <stdio.h>
void main()
{
    printf("lab1: Hello World\n");
    fflush(stdout);
    /* 让程序打印完后继续维持在用户态 */
    while(1);
}
```
上述文件保存在`~/lab1/helloworld.c`中，然后用gcc将上面代码（-static）静态编译成（-m32）32位可执行文件。
```shell
gcc -o helloworld -m32 -static helloworld.c
```
   1. 用cpio打包initramfs。
```shell
echo helloworld | cpio -o --format=newc > hwinitramfs
```
这里使用了一些shell指令：
- `echo`将指定文件的内容打印出来（默认打印到console）
- `|`是管道（pipe），建立了IO通道，将左侧的指令的输出连接到右侧指令的输入
- `cpio`（copy input and output）最初功能将文件从一个设备复制到另一个。常与管道和重定向一同使用（上面指令便是一个例子）来将多个文件归档（archive）为一个文件。
- `-o`指定创建的归档文件名
- `--format`指定归档格式为`newc`，这是一种较新的兼容性较好的格式
- `>`是重定向（redirect）将左侧执行的结果输送到右侧指定的文件（或创建新文件） 
  
#### 启动内核，并加载initramfs。
```shell
qemu-system-i386 -kernel linux-5.10.170/arch/x86/boot/bzImage -initrd hwinitramfs -s -S -append "console=ttyS0 rdinit=helloworld" -nographic
```
- `-initrd hwinitramfs`指定要加载的磁盘映像为刚得到的归档文件
- `rdinit=helloworld`指定内核初始化（random disk init）时启动的(在磁盘映像中的)程序

之后重复上面的gdb的调试过程，可以看到gdb中输出了`lab1: Hello World\n`

#### 完成Busybox的编译、启动过程
下面的过程在文件夹`~/lab1`下进行。
先下载并解压busybox。
1. 编译busybox
```shell
make defconfig
make menuconfig # 进入settings  
```
设置静态构建
进入`settings`，然后在`Build BusyBox as a static binary(no shared libs)`处输入`Y`勾选，然后分别设置`() Additional CFLAGS`和`() Additional LDFLAGS`为`(-m32 -march=i386) Additional CFLAGS`和`(-m32) Additional LDFLAGS`，分别指定编译器与链接器为32位i386的版本。

保存退出，然后编译。
```shell
make -j8
make install
```
2. 制作Initramfs
将安装在_install目录下的文件和目录取出放在`~/lab1/mybusybox`处。

```shell
cd ~/lab1
mkdir mybusybox
mkdir -pv mybusybox/{bin,sbin,etc,proc,sys,usr/{bin,sbin}} # 创建文件夹树
cp -av busybox-1_33_0/_install/* mybusybox/
cd mybusybox
```
- cp copy
- `-a`参数表示使用归档模式进行复制，它会保留所有文件属性，包括所有者、权限、时间戳等。
- `-v`参数表示启用详细（verbose）输出模式，它会显示每个文件被复制的详细信息。
- `-p`参数表示自动创建父目录



initramfs需要一个init程序，可以写一个简单的shell脚本作为init。用gedit打开文件`init`，复制入如下内容，保存退出。
```shell
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
exec /bin/sh
```

加上执行权限。
change mode user + "execute" permission to "init"
```shell
chmod u+x init
```

将x86-busybox下面的内容打包归档成cpio文件，以供Linux内核做initramfs启动执行。

```shell
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ~/lab1/initramfs-busybox-x86.cpio.gz
```
- `-print0`让find输出文件名时用null字符作为间隔。这有利于将输出传递到后续命令
- `--null`对与cpio的功能与`-print0`相同。
- `-o`让cpio覆写目标文件
- `-9`压缩等级中的最高级


#### 完成Busybox的远程调试
```shell
cd ~/lab1
qemu-system-i386 -kernel linux-5.10.169/arch/x86/boot/bzImage -initrd initramfs-busybox-x86.cpio.gz -nographic -append "console=ttyS0"
```
重复之前用gdb调试运行qemu的方式，然后在qemu命令行窗口使用`ls`命令，若可看到当前文件夹则成功。
成功。

#### 完成Linux 0.11内核的编译、启动和调试
按指导对linux0.11内核进行编译后成功启动，打开了linus大师的`iceCityOS`（不得不说linus的注释风格挺幽默的，而且代码好整齐！）
随后将内核的磁盘镜像挂载到当前的ubuntu中，往内核磁盘映像的`/usr`文件夹里`touch`了一个`hello.txt`，虽然忘记往里面写东西了，但是再次启动linux0.11内核是可以看到`hello.txt`，而且能用！
![](./images/a909e67f98b0d281ab9a1fb07ec0a173)


### 总结
下面是我对上面实验所发生的事情的理解：
编译好linux的内核后，我们用qemu和linux的内核来搭载一些程序，例如上面的helloworld.c和busybox。在这个过程中，我们用命令行为qemu指定了linux镜像文件的位置、initramfs（初始化磁盘映像文件系统）的位置（作为系统启动期间的临时磁盘，存储一些初始化程序或脚本以启动系统）、rdinit（初始化程序）的位置。再使用gdb辅助执行qemu，我们得以看见内核执行的过程。而所指定的初始化程序，可以理解为“搭载”，是让一个操作系统执行特定程序的最简陋方法。
最后一个内容是实操，编译并用qemu和gdb运行linux内核，并且通过挂载实践“磁盘映像”的含义。
后面看了一下linux0.11的根目录，虽然main文件看不是很懂，但是大为震撼。文档中甚至教你如何在现代操作系统中用qemu等方式启动这个操作系统！