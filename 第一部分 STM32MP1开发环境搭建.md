# 开发环境搭建

如果学习过UCOS/FreeRTOS应该知道，UCOS/FreeRTOS移植就是在官方的SDK包里面找一个和自己所使用的芯片一样的工程编译一下，然后下载到开发板就可以了。Linux的移植要复杂的多，在移植Linux之前需要先移植一个bootloader代码，这个bootloader代码用于启动Linux内核，bootloader有很多，常用的就是U-Boot。移植好U-Boot以后再移植Linux内核，移植完Linux内核以后Linux还不能正常启动，还需要再移植一个根文件系统(rootfs)，根文件系统里面包含了一些最常用的命令和文件。所以U-Boot、LinuxKernel和rootfs这三者一起构成了一个完整的Linux系统，一个可以正常使用、功能完善的Linux系统。

在Ubuntu下进行Cortex-A(STM32MP157)开发需要安装一些软件，也就开发环境搭建，环境搭建好以后我们就可以进行开发了。环境搭建分为Ubuntu和Windows，因为我们最熟悉Windows，所以代码编写、查找资料等一般是在Windows下进行的。但是Linux开发又必须在Ubuntu下进行，所以需要搭建Ubuntu下的开发环境，主要是交叉编译器的安装，本章我们就分为Ubuntu和Windows，讲解这两种操作系统下的环境搭建。

## 1 Ubuntu和Windows文件互传

### 1.1 开启Ubuntu下的FTP服务

打开Ubuntu的终端窗口，然后执行如下命令来安装FTP服务：

~~~bash
sudo apt-get install vsftpd
~~~

等待软件自动安装，安装完成以后使用如下VI命令打开/etc/vsftpd.conf，命令如下：

~~~bash
sudo vi /etc/vsftpd.conf
~~~

打开以后在vsftpd.conf文件中找到如下两行：

~~~bash
local_enable=YES
write_enable=YES
~~~

确保上面两行前面没有“#”，有的话就取消掉。修改完vsftpd.conf以后保存退出，使用如下命令重启FTP服务：

~~~bash
sudo /etc/init.d/vsftpd restart
~~~

### 1.2 Windows下FTP客户端安装

Windows下FTP客户端我们使用FileZilla，这是个免费的FTP客户端软件，可以在FileZilla官网下载，下载地址如下： [https://www.filezilla.cn/download] 。如果是32位电脑就选择32位版本，64位电脑就选择64位版本。

### 1.3 FileZilla软件设置

Ubuntu作为FTP服务器，FileZilla作为FTP客户端，客户端肯定要连接到服务器上，打开站点管理器，点击：文件->站点管理器。点击“新站点(N)”按钮来创建站点，新建站点以后就会在“我的站点”下出现新建的这个站点，站点的名称可以自行修改，比如我将新的站点命名为“Ubuntu”。

选中新创建的“Ubuntu”站点，然后对站点的“常规”进行设置，设置项点如下：

(1)协议(T):FTP-文件传输协议

(2)主机(H):192.168.3.4，Ubuntu主机IP地址

(3)加密(E):只使用明文FTP(不安全)

(4)登录类型(L):正常

(5)用户(U):此处填写主机用户名称

(6)密码(W):此处填写主机密码

设置好后点击“连接”按钮，第一次连接可能会弹出提示是否保存密码的对话框，点击确定即可。

连接成功以后显示资源管理，其中左边是Windows文件目录，右边是Ubuntu文件目录，默认进入用户根目录下（比如我电脑的“/home/username”）。但是注意观察资源管理界面中Ubuntu文件目录下的中文目录可能是乱码，这是因为编码方式没有选对，先断开连接，点击：服务器(S)->断开连接，然后打开站点管理器，选中要设置的站点“Ubuntu”，选择“字符集”，设置为“强制UTF-8”。设置字符集以后重新连接FTP服务器，Ubuntu下的文件目录中文显示就会正常。

## 2 Ubuntu下NFS和SSH服务开启

### 2.1 NFS服务开启

后面进行Linux驱动开发的时候需要NFS启动，因此要先安装并开启Ubuntu中的NFS服务，使用如下命令安装NFS服务：

~~~bash
sudo apt-get install nfs-kernel-server rpcbind
~~~

等待安装完成，安装完成以后在用户根目录下创建一个名为“linux”的文件夹，以后所有的东西都放到这个“linux”文件夹里面，在“linux”文件夹里面新建一个名为“nfs”的文件夹。创建的nfs文件夹供nfs服务器使用，以后可以在开发板上通过网络文件系统来访问nfs文件夹。首先配置nfs，使用如下命令打开nfs配置文件/etc/exports：

~~~bash
sudo vi /etc/exports
~~~

打开/etc/exports以后在后面添加如下所示内容：

~~~bash
/home/usernmae/linux/nfs *(rw,sync,no_root_squash)
~~~

重启NFS服务，使用命令如下：

~~~bash
sudo /etc/init.d/nfs-kernel-server restart
~~~

### 2.2 SSH服务开启

开启Ubuntu的SSH服务以后我们就可以在Windwos下使用终端软件登陆到Ubuntu，比如使用SecureCRT，Ubuntu下使用如下命令开启SSH服务：

~~~bash
sudo apt-get install openssh-server
~~~

上述命令安装ssh服务，ssh的配置文件为/etc/ssh/sshd_config，使用默认配置即可。

## 3 Ubuntu交叉编译工具链

### 3.1 交叉编译器安装

ARM裸机、Uboot移植、Linux移植等都需要在Ubuntu下进行编译，编译就需要编译器，在Liux进行C语言开发，需要使用GCC编译器进行代码编译，但是Ubuntu自带的gcc编译器是针对X86架构的。要编译ARM架构的代码，需要一个在X86架构的PC上运行，可以编译ARM架构代码的GCC编译器，这个编译器就叫做交叉编译器，总结一下交叉编译器就是：

1、一个GCC编译器。

2、这个GCC编译器运行在X86架构的PC上的。

3、这个GCC编译器是编译ARM架构代码的，也就是编译出来的可执行文件是在ARM芯片上运行的。

交叉编译器中“交叉”的意思就是在一个架构上编译另外一个架构的代码，相当于两种架构“交叉”。交叉编译器有很多种，ST推荐了两款通用交叉编译器，一个是ARM官方出品的：gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf；一个是linaro出品的：gcc-linaro-7.2.1-2017.11-x86_64_arm-linux-gnueabi.tar.xz，本教程我们使用ARM官方出品的交叉编译器。

首先是下载ARM官方出品的交叉编译器，编译器下载地址如下： [https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads] 。其中的gcc-arm-x.x-xxxx.xx-x86_64-arm-none-linux-gnueabihf.tar.xz是在64位Ubuntu下使用的交叉编译器(其中，x.x代表版本，xxxx.xx代表版本发布时间，本教程以gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf.tar.xz为例)。

先将交叉编译工具拷贝到Ubuntu中，在Ubuntu中创建目录：/usr/local/arm，命令如下：

~~~bash
sudo mkdir /usr/local/arm
~~~

创建完成以后将刚刚拷贝的交叉编译器复制到/usr/local/arm这个目录中，在终端使用命令“cd”进入到存放有交叉编译器的目录，拷贝完成以后在/usr/local/arm目录中对交叉编译工具进行解压，解压命令如下：

~~~bash
sudo tar -vxf gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf.tar.xz
~~~

解压完成后会生成一个名为“gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf”的文件夹，文件夹里面就是交叉编译工具链。修改环境变量，打开/etc/profile文件，命令如下：

~~~bash
sudo vi /etc/profile
~~~

打开/etc/profile以后，在最后面输入如下所示内容：

~~~bash
export PATH=$PATH:/usr/local/arm/gcc-arm-9.2-2019.12-x86_64-arm-none-linux-gnueabihf/bin
~~~

修改好以后就保存退出，重启Ubuntu系统，交叉编译工具链(编译器)就安装成功了。

### 3.2 安装相关库

在使用交叉编译器之前还需要安装一下其它的库，命令如下：

~~~bash
sudo apt-get update//先更新，否则安装库可能会出错
sudo apt-get install lsb-core lib32stdc++6//安装库等待这些库安装完成。
~~~

### 3.3 交叉编译器验证

首先查看一下交叉编译工具的版本号，输入如下命令：

~~~bash
arm-none-linux-gnueabihf-gcc -v
~~~

如果交叉编译器安装正确的话就会显示版本号。

使用Ubuntu自带的GCC编译器，用的是命令“gcc”。使用本节安装的交叉编译器的时候使用的命令是“arm-none-linux-gnueabihf-gcc”，“arm-none-linux-gnueabihf-gcc”的含义如下：

1、arm表示这是编译arm架构代码的编译器。

2、none表示厂商，一般半导体厂商会修改通用的交叉编译器，此处就是半导体厂商的名字，ARM自己做的交叉编译这里为none，表示没有厂商。

3、linux表示运行在linux环境下。

4、gnueabihf表示嵌入式二进制接口，后面的hf是hardfloat的缩写，也就是硬件浮点，说明此交叉编译工具链支持硬件浮点。

5、gcc表示是gcc工具。

## 4 Visual Studio Code软件的安装和使用

### 4.1 Visual Studio Code的安装

VisualStuioCode是一个编辑器，可以用来编写代码，Visual Studio Code本教程简称为VSCode，VSCode是微软出的一款免费编辑器。VSCode有Windows、Linux和macOS三个版本的，是一个跨平台的编辑器。VSCode下载地址是： [https://code.visualstudio.com/] 。

1、Windows版本安装

Windows版本的安装和容易，和其他Windows应用程序一样，双击.exe安装包，然后一路“下一步”即可，安装完成以后在桌面上就会有VSCode的图标。

2、Linux版本安装

有时候也需要在Ubuntu下阅读代码，所以需要在Ubuntu下安装VSCode。将Linux下VSCode的.deb软件包拷贝到Ubuntu系统中，然后使用如下命令安装：

~~~bash
sudo dpkg -i code_1.50.1-1602600906_amd64.deb
~~~

安装完成以后搜索“Visual Studio Code”就可以找到。将图标添加到Ubuntu桌面上，安装的所有软件图标都在目录/usr/share/applications中，找到Visual Studio Code的图标，然后点击鼠标右键，选择复制到->桌面。第一次双击的时候会提示“未信任的应用程序启动器”，点击“Trust and Launch”，选择信任即可。以后就可以直接双击图标打开VSCode。

### 4.2 Visual Studio Code的安装

VSCode支持多种语言，比如C/C++、Python、C#等等，本教程主要用C/C++来编写程序，所以需要安装C/C++的扩展包。

需要按照的插件有下面几个：

1)、C/C++，这个肯定是必须的。

2)、C/C++ Snippets，即C/C++重用代码块。

3)、C/C++ Advanced Lint,即C/C++静态检测。

4)、CodeRunner，即代码运行。

5)、IncludeAutoComplete，即自动头文件包含。

6)、Rainbow Brackets，彩虹花括号，有助于阅读代码。

7)、OneDarkPro，VSCode的主题。

8)、GBKtoUTF8，将GBK转换为UTF8。

9)、ARM，即支持ARM汇编语法高亮显示。

10)、Chinese(Simplified)，即中文环境。

11)、vscode-icons，VSCode图标插件，主要是资源管理器下各个文件夹的图标。

12)、compareit，比较插件，可以用于比较两个文件的差异。

13)、DeviceTree，设备树语法插件。

14)、TabNine，一款AI自动补全插件，强烈推荐！

安装完成以后重新打开VSCode，如果要查看已经安装好的插件

### 4.3 Visual Studio Code新建工程

新建一个文件夹用于存放工程，比如新建文件夹目录为E:\VScode_Program\1_test，路径尽量不要有中文和空格。打开VSCode，然后在VSCode上点击文件->打开文件夹...，选刚刚创建的“1_test”文件夹，点击文件->将工作区另存为...，打开工作区命名对话框，输入要保存的工作区路径和工作区名字。

工作区保存成功以后，点击“新建文件”按钮创建main.c和main.h这两个文件。此时“实验1TEST”工程中有.vscode文件夹、main.c和main.h，这三个文件和文件夹同样会出现在“实验1 test”文件夹中。

在main.h中输入如下所示内容：

~~~cpp
#include <stdio.h>

int add(inta,intb);
~~~

在main.c中输入如下所示内容：

~~~cpp
#include <main.h>

intadd(inta,intb)
{
    return(a +b);
}

intmain(void)
{
    intvalue =0;

    value=add(5,6);
    printf("5 + 6 = %d",value);
    return0;
}
~~~

此时提示找不到“stdio.h”这个头文件，找不到“main.h”，同样的在main.h文件中会提示找不到“stdio.h”。这是因为我们没有添加头文件路径。按下“Ctrl+Shift+P”打开搜索框，然后输入“Editconfigurations”，选择“C/C++:Editconfigurations...”，C/C++的配置文件是个json文件，名为：c_cpp_properties.json。c_cpp_properties.json中的变量“includePath”用于指定工程中的头文件路径，但是“stdio.h”是C语言库文件，而VSCode只是个编辑器，没有编译器，所以是没有stdio.h的，除非安装一个编译器，比如CygWin，然后在includePath中添加编译器的头文件。

在VScode上打开一个新文件的话会覆盖掉以前的文件，这是因为VSCode默认开启了预览模式，预览模式下单击左侧的文件就会覆盖掉当前的打开的文件。如果不想覆盖的话采用双击打开即可，或者设置VSCode关闭预览模式，设置->Enable Preview(勾选掉)。

我们在编写代码的时候有时候会在右下角有警告提示，这是因为插件C/C++ Lint打开了几个功能，我们将其关闭就可以了。在C/C++ Lint配置界面上找到CLang:Enable、Cppcheck:Enable、Flexlint:Enable这个三个，然后取消掉勾选即可。

## 5 MobaXterm软件安装和使用

### 5.1 MobaXterm软件安装

MobaXterm是一款终端软件，支持很多种协议，比如SSH、Telnet、Rsh、Xdmcp、RDP、VNC、FTP、SFTP、Serial等等，功能强大而且免费(也有收费版)！在这里推荐大家使用此软件作为终端调试软件，MobaXterm软件在其官网下载即可，地址为： [https://mobaxterm.mobatek.net/] 。

### 5.2 MobaXterm软件使用

下面简单介绍一下使用MobaXterm建立Serial连接，也就是串口连接。
双击MobXterm图标，打开此软件，点击菜单栏中的“Sessions->New session”按钮，打开新建会话窗口，点击“Serial”按钮，打开串口设置界面，先选择要设置的串口号，然后设置波特率为115200(根据自己实际需要设置)。还要设置串口的其他功能，下方一共有三个设置选项卡。

点击AdvancedSerialsettings选项卡，设置串口的其他功能，比如串口引擎、数据位、停止位、奇偶校验和硬件流控等。

如果要设置终端相关的功能的话点击“Terminal settings”即可，比如终端字体以及字体大小等。设置完成以后点击下方的“OK”按钮即可。串口设置完成以后就会打开对应的终端窗口。

## 6 Windows下ST官方软件安装

### 6.1 ST官方软件简介

在开发STM32MP157的时候还需要用到一些ST官方提供的软件，一共有三种：STM32CubeMX、STM32CubeIDE、STM32CubeProgrammer，这三个软件的功能如下所示：

|软件|功能|
|:----:|:----:|
|STM32CubeMXSTM32|图形化配置软件|
|STM32CubeIDE|ST官方集成IDE，用于M4内核|
|STM32CubeProgrammer|STM32烧写软件|

#### 6.1.1 STM32CubeMX

STM32CubeMX是图形化配置工具，可以直观的选择MCU/MPU型号，可以动态地配置引脚、配置时钟树、配置中间件、配置内存，可以生成初始化代码、MPU设备树源码，可以进行DDR测试等。STM32CubeMX开源且跨平台，支持在Windows、Linux和macOS操作系统上运行（64位）。

#### 6.1.2 STM32CubeIDE

STM32CubeIDE是ST于2019年新推出的一款多功能的集成开发工具，它集成了TrueSTUDIO和STM32CubeMX插件，并基于GDB进行调试，它允许集成数百个现有插件，这些插件完成Eclipse的功能。

TrueSTUDIO插件是一款建立在Eclipse  CDT、GCC和GDB的C/C++集成开发工具，其具有项目创建和管理、代码编辑、代码编译以及代码调试等功能。STM32CubeMX插件具有图形化配置功能，可以直观地选择MCU/MPU型号、动态配置引脚和设置时钟树、动态设置外围设备和中间器件的模式，可以自动处理引脚冲突和生成初始化代码。

#### 6.1.3 STM32CubeProgrammer

STM32CubeProgrammer将ST Visual Programmer、DFUse Device Firmware Update、Flash Loader和ST-Link等软件功能整合到了一起，为用户提供STM32 微控制器代码烧写和固件安全安装、更新功能，帮助用户更快地开发特定外部存储器的加载程序。

STM32CubeProgrammer支持对外部存储器进行编程、擦除和验证，用户烧写STM32微控制器既可使用片上SWD  (单线调试)或JTAG调试端口，也可以用程序引导装入端口(例如UART和USB)。它支持Motorola  S19、Intel  HEX、ELF 和二进制格式（BIN文件），支持ST-LINK固件更新，支持图形化界面操作也支持命令行操作，在命令行界面，可通过脚本实现自动化编程。目前STM32CubeProgrammer支持Windows、Linux和macOS多个操作系统（64位版本）。

### 6.2 Java环境安装

在安装STM32CbeMX和STM32CubeIDE前我们要先安装Java的环境，Java运行环境版本必须是V1.7及以上，否则会导致这两个无法使用。大家可以到Java官网 [https://www.java.com/zh-CN/download/manual.jsp] 查找下载最新的64位Java软件。

安装完Java运行环境之后，为了检测是否正常安装，我们可以打开Windows的cmd命令输入框，输入如下命令：

~~~bash
java -version //命令查询Java版本
~~~

如果安装成功的话就会打印出Java的版本号。

## 7 Ubuntu下ST官方软件安装

### 7.1 Java环境安装

在Ubuntu下安装这三个软件也是需要JAVA运行环境的，Ubuntu可能会默认安装了OpenJDK环境，但是STM32CubeProgrammer是用Oracle的JDK编写的，所以需要先卸载掉默认的OpeJDK，首先输入“java -version”查看一下当前Ubuntu系统下有没有OpenJDK，有的话会输出OpenJDK版本信息。

~~~bash
java -version
~~~

如果没有输出JAVA版本信息的话就说明当前Ubuntu还没有安装JAVA环境。卸载OpenJDK的方法很简单，输入如下命令：

~~~bash
sudo apt-get remove openjdk*
~~~

卸载完成以后重新输入“java-version”，如果不能输出java版本信息就说明卸载成功了。

将下载的Linux版本Java(本文以jre-8u271-linux-x64.tar.gz为例)压缩包拷贝到Ubuntu下，然后解压到Ubuntu的/usr/lib/jvm目录下，输入如下命令：

~~~bash
sudo mkdir /usr/local/java  //创建目录
sudo tar vzxf jre-8u271-linux-x64.tar.gz -C /usr/local/java  //解压
~~~

修改/etc/profile，在文件最后面追加如下内容：

~~~bash
export CLASSPATH=.:/usr/local/java/jre1.8.0_271/lib
export PATH=$PATH:/usr/local/java/jre1.8.0_271/bin
~~~

重启电脑，输入“java-version”查看java安装是否成功，安装成功的话就会打印出java环境的版本号。

### 7.2 STM32CubeMX安装

在Ubuntu下新建一个名为“STM32CubeMX”的文件夹，然后将STM32CubeMX安装包en.stm32cubemx_v6-0-1.zip发送到此文件夹下，然后进行解压，命令如下：

~~~bash
unzip en.stm32cubemx_v6-0-1.zip
~~~

解压出来的文件一个都不要删除，防止安装的时候出错，在终端里面执行文件夹中的SetupSTM32CubeMX-6.0.1.linux，输入如下命令：

~~~bash
chmod 777 SetupSTM32CubeMX-6.0.1.linux  //给予可执行权限
chmod 777 SetupSTM32CubeMX-6.0.1.exe    //给予可执行权限
./SetupSTM32CubeMX-6.0.1.linux
~~~

根据提示一步步安装下去，安装默认路径安装，注意，STM32CubeMX安装完成以后可能在/usr/share/applications目录下找不到STM32CubeMX的图标。所以我们需要直接搜索，点击Ubuntu主界面左上角的“活动”按钮，打开搜索条，输入“STM32CubeMX”查找相应的文件。

### 7.3 STM32CubeIDE安装

在Ubuntu下新建一个名为“STM32CubeIDE”的文件夹，然后将STM32CubeMX安装包en.st-stm32cubeide_1.4.0_7511_20200720_0928_amd64_sh.zip发送到此文件夹下，然后进行解压，命令如下：

~~~bash
unzip en.st-stm32cubeide_1.4.0_7511_20200720_0928_amd64_sh.zip
~~~

在终端里面执行图4.8.3.1中的st-stm32cubeide_1.4.0_7511_20200720_0928_amd64.sh，输入如下命令：

~~~bash
chmod 777 st-stm32cubeide_1.4.0_7511_20200720_0928_amd64.sh //给予可执行权限
./st-stm32cubeide_1.4.0_7511_20200720_0928_amd64.sh
~~~

输入上述命令以后会先解压压缩包，然后打开安装许可，阅读完安装许可，然后输入‘y’接受许可。输入‘y’，表示接受许可，默认为‘n’，也就是不接受许可。接下来会出现多个询问，根据实际情况填写即可，最后等待安装完成。安装完成以后到/usr/share/applications目录下将STM32CubeIDE的图标复制到桌面上，这样就可以通过点击桌面图标打开STM32CubeIDE软件了。

### 7.4 STM32CubeProgrammer安装

在Ubuntu下新建一个名为“STM32CubeProgrammer”的文件夹，然后将STM32CubeMX安装包en.stm32cubeprog_v2-5-0.zip发送到此文件夹下，然后进行解压缩，命令如下：

~~~bash
unzip en.stm32cubeprog_v2-5-0.zip
~~~

解压出来的文件一个都不要删除，防止安装的时候出错，在终端里面执行图4.8.4.2中的SetupSTM32CubeProgrammer-2.5.0.linux即可，输入如下命令：

~~~bash
./SetupSTM32CubeProgrammer-2.5.0.linux
~~~

不要修改安装路径，采用默认路径即可。以后就可以直接在Ubuntu下通过STM32CubeProgrammer来烧写系统了。

最后，在Ubuntu中安装libusb1.0软件包，输入如下命令：

~~~bash
sudo apt-get install libusb-1.0.0-dev
~~~

## 8 USB DFU以及STLink驱动安装

### 8.1 Windows下USB DFU以及STLink驱动安装

在Windows下USB DFU驱动不需要安装，所以只需要安装STLink驱动。如果是64位的电脑，则双击：dpinst_amd64.exe进行安装。如果是32位的电脑，则双击：dpinst_x86.exe进行安装。

在ST LINK驱动安装完成以后，我们在电脑设备管理器里面可以看到会多出一个设备（此时ST LINK必须通过USB连接到电脑，ST LINK红灯常亮）。

注意：

①、不同Windows版本设备名称和所在设备管理器栏目可能不一样，例如WIN7电脑插上STLINK后显示的是：STMicroelectronics STLINK dongle。

②、如果设备名称旁边显示的是黄色的叹号，请直接点击设备名称，然后在弹出的界面点击更新设备驱动。

### 8.2 Ubuntu下USB DFU以及STLink驱动安装

如果想要在Ubuntu下通过USB烧写系统，那么必须要安装USB DFU驱动。必须先在Ubuntu下安装STM32CubeProgrammer，因为STM32CubeProgrammer安装过程中会解压出USB DFU以及STLink相关驱动文件。

找到Ubuntu里面的STM32CubeProgrammer安装路径，如果安装的时候选择的默认路径，那么就会在用户根目录下生成一个名为“STMicroelectronics”的文件夹，比如在电脑上的绝对路径“/home/username/STMicroelectronics”。

进入路径：/STMicroelectronics/STM32Cube/STM32CubeProgrammer/Drivers/rules，将.rules文件全部拷贝到Ubuntu的/etc/udev/rules.d目录下，命令如下：

~~~bash
cd /home/zuozhongkai/STMicroelectronics/STM32Cube/STM32CubeProgrammer/Drivers/rulessudo 
cp * /etc/udev/rules.d/
~~~

拷贝完成以后重启Ubuntu，用USB Type-C线将开发板的USB OTG接口与电脑连接起来，默认情况下，拨码开关设置为000，也就是USB启动，然后按下复位按键，此时电脑会发出USB枚举的叮咚声。默认情况下，USB会连接到Windwos下，需要将USB连接到Ubuntu，所以需要设置一下VMware，WMware右下角会有当前电脑所有连接的USB设备，鼠标放上去以后会显示每个USB设备的名字，找到含有“STMicroelectronics DFU”字样的USB设备，鼠标放上去以后点击鼠标右键“连接”按钮，此时USB DFU就会断开与主机(Windows)的连接，从而连接到虚拟机(Ubuntu)上。连接成功以后对应的USB图标颜色就会深。

连接成功以后就可以使用STM32CubeProgrammer软件进行测试，打开Ubuntu下的STM23CubeProgrammer软件，选择“USB”字样，查看一下端口是否有“USB1”字样。右上角的“Connect”按钮即可通过USB接口连接到开发板上，左下角的log区就会输出相应的信息。如果此时USB DFU已经成功工作，说明Ubuntu下STM32 USB DFU驱动成功。

最后，测试下STLink，这个测试比较简单，将STLink连接到Ubuntu下，如果STLink工作成功的话就会在/dev目录下生成相应的设备文件，命令如下：

~~~bash
ls /dev/stlink*
~~~
