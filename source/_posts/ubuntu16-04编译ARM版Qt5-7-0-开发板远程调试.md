---
title: ubuntu16.04编译ARM版Qt5.7.0+开发板远程调试
date: 2017-06-05 14:02:12
categories: [教程]
tags: [Qt]
---

环境：
- Ubuntu16.04 64bit

介绍在ubuntu16.04下编译安装Qt5.7.0ARM版本，以及通过ssh远程在ARM开发板上调试的过程。

<!-- more -->

## **安装依赖** ##
```bash
$ sudo apt-get install lsb-core
$ sudo apt-get install lib32stdc++6
$ sudo apt-get install libncurses5-dev
$ sudo apt-get install lib32ncurses5
```

## **编译Qt** ##
### 修改 ###
修改源码目录下qtbase/mkspecs/linux-arm-gnueabi-g++/qmake.conf
在其中添加和修改，
```bash
QT_QPA_DEFAULT_PLATFORM = linux #eglfs	添加
QMAKE_CFLAGS_RELEASE += -O2 -march=armv7-a	#添加
QMAKE_CXXFLAGS_RELEASE += -O2 -march=armv7-a	#添加

# modifications to g++.conf
QMAKE_CC                = arm-linux-gnueabihf-gcc		#修改成自己的交叉链
QMAKE_CXX               = arm-linux-gnueabihf-g++	#修改成自己的交叉链
QMAKE_LINK              = arm-linux-gnueabihf-g++	#修改成自己的交叉链
QMAKE_LINK_SHLIB        = arm-linux-gnueabihf-g++	#修改成自己的交叉链

# modifications to linux.conf
QMAKE_AR                = arm-linux-gnueabihf-ar cqs	#修改成自己的交叉链
QMAKE_OBJCOPY           = arm-linux-gnueabihf-objcopy	#修改成自己的交叉链
QMAKE_NM                = arm-linux-gnueabihf-nm -P	#修改成自己的交叉链
QMAKE_STRIP             = arm-linux-gnueabihf-strip	#修改成自己的交叉链
```
### 编译和安装 ###
configure的参数直接影响make和make install是否会出错，本配置不包括opengl，只满足基本需求，加入opengl需要做的工作有点多，我会整理一下再发表出来。

qtvirtualkeyboard是Qt5.7.0中自带的一个虚拟键盘，支持中文输入，这里需要将其跳过，因为编译qtvirtualkeyboard间接需要opengl库的支持，不跳过在make install这一步是会出错的。同时，如果没有opengl库，请自觉跳过tests，examples，tools的编译。
```bash
$ ./configure -v -prefix /home/linux/Software/Qt5.7.0-arm -release -make libs 
-xplatform linux-arm-gnueabi-g++ -optimized-qmake -no-accessibility 
-pch -qt-sql-sqlite -qt-zlib -no-sse2 -no-openssl -no-nis -no-cups -no-glib 
-no-separate-debug-info -qreal float -nomake tests -nomake examples 
-nomake tools -skip qtvirtualkeyboard
$ make -j4
$ make install
```
完成之后，就可以在Qt Creator中添加构建套件了。

### **问题总结** ###
1. 64位系统，已经将交叉编译链写入到PATH中，仍出现arm-linux-gnueabihf-gcc: 没有那个文件或目录
```bash
$ sudo apt-get install lsb-core
```

2. arm-linux-gnueabihf-g++: error while loading shared libraries: libstdc++.so.6: cannot open shared object file: No such file or directory
```bash
$ sudo apt-get install lib32stdc++6
```

3. 配置Qt Creator添加交叉链中的gdb出现警告：error while loading shared libraries: libncurses.so.5: cannot open shared object file: No such file or directory
```bash
$ sudo apt-get install libncurses5-dev
$ sudo apt-get install lib32ncurses5
```

## **Qt与ARM开发板实现远程调试** ##
在Qt4版本中，通常在将可执行文件移植到ARM板上之前，可以使用Embedded X86版本(qvfb)的Qt进行调试，没有问题下再移植到ARM板上运行。起码我没用Qt5之前都是这么干的。然而Qt5中已不再使用Qt4中的QWS机制，而是采用了新的机制QPA，所以使用之前qvfb调试Qt5已经行不通。
为了能在编写的过程中能更有效的显示出在ARM板上运行的效果，需要对Qt Creator和ARM板配置一下，使其通过Qt Creator运行，可以直接在ARM板上显示出来。
### 移植ssh到ARM板 ###
在PC端，先创建以下目录，主要是为了描述方便
```bash
$ mkdir ~/SSH
$ mkdir ~/SSH/install
$ mkdir ~/SSH/lib
$ mkdir ~/SSH/source
```
#### 下载源码包 ####
需要下载openssh，openssl，zlib，已放在了我的百度网盘中
下载地址：
将源码包都解压到 ~/SSH/source目录中

#### 交叉编译zlib ####
```bash
$ cd ~/SSH/source/zlib-1.2.3
$ ./configure --prefix=~/SSH/install/zlib-1.2.3
```
configure之后，修改Makefile
```bash
CC=arm-linux-gnueabihf-gcc		#修改成自己的交叉链
LDSHARED=arm-linux-gnueabihf-gcc	#修改成自己的交叉链
CPP =arm-linux-gnueabihf-gcc -E		#修改成自己的交叉链
AR=arm-linux-gnueabihf-ar rc		#修改成自己的交叉链
```
```bash
$ make
$ make install
```
#### 交叉编译openssl ####
```bash
$ cd ~/SSH/source/openssl-0.9.8e
$ ./Configure --prefix=~/SSH/install/openssl-0.9.8e 
os/compiler:arm-linux-gnueabihf-gcc
$ make
$ make install
```
#### 交叉编译openssh ####
```bash
$ cd ~/SSH/source/openssh-4.6p1
$ ./configure --host=arm-none-linux-gnueabi --with-libs 
--with-zlib=~/SSH/install/zlib-1.2.3 
--with-ssl-dir=~/SSH/install/openssl-0.9.8e --disable-etc-default-login 
CC= arm-linux-gnueabihf-gcc AR=arm-linux-gnueabihf-ar
$ make
```
#### 移植 ####
请确保ARM板中有以下目录，如果没有，则创建
```bash
/usr/local/bin/
/usr/local/sbin/
/usr/local/etc/
/usr/local/libexec/
/var/run/
/var/empty/
```
- 在PC端，先生成key文件
```bash
$ cd ~/SSH/source/openssh-4.6p1
$ ssh-keygen -t rsa -f ssh_host_rsa_key -N ""
$ ssh-keygen -t dsa -f ssh_host_dsa_key -N ""
$ ssh-keygen -t ecdsa -f ssh_host_ecdsa_key -N ""
```
- 将PC端 ~/SSH/source/openssh-4.6p1 目录下的 scp sftp ssh ssh-add ssh-agent ssh-keygen ssh-keyscan 拷贝到 ARM板中的/usr/local/bin目录

- 将PC端 ~/SSH/source/openssh-4.6p1 目录下的 moduli ssh_config sshd_config 拷贝到 ARM板中的/usr/local/etc目录

- 将PC端 ~/SSH/source/openssh-4.6p1 目录下的 sftp-server ssh-keysign 拷贝到 ARM板中的/usr/local/libexec目录

- 将PC端 ~/SSH/source/openssh-4.6p1 目录下的 sshd 拷贝到 ARM板中的/usr/local/sbin目录

- 将PC端 ~/SSH/source/openssh-4.6p1 目录下的 ssh_host_*_key文件 拷贝到 ARM板中的/usr/local/etc目录

- 修改ARM板passwd文件，在/etc/passwd文件最后添加
```bash
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
```
- 修改ARM板中/usr/local/etc目录下ssh_host_*_key的权限修改成700
```bash
$ chmod 700 ssh_host_*
```
- 修改ARM板/usr/local/etc/sshd_config文件，将
```bash
Subsystem sftp /usr/libexec/sftp-server	#原
修改成
Subsystem sftp /usr/local/libexec/sftp-server	#修改
不然ARM板开启sshd后，Qt Creator运行会提示
Failed to connect SFTP subsystem: Remote host may not have sftp-server
```
#### 运行 ####
```bash
$ /usr/local/sbin/sshd
```
可以在ARM板中将其加入到启动项中，开机自动开启，省得每次调试的时候都要在ARM板上手动开启。

### 配置ARM板Qt运行环境
以我的ARM板为例，我的里面是干净的，关于Qt的什么都没有。在ARM板上确保有以下目录，如果没有则创建
```bash
/opt
/opt/qt
```
#### 交叉编译libiconv-1.14 ####
交叉编译libiconv-1.14会得到preloadable_libiconv.so库
```bash
$ ./configure CC=arm-linux-gnueabihf-gcc
 --prefix=/home/linux/Software/libiconv-1.14-arm --host=arm-linux
 $ make
 $ make install
```
preloadable_libiconv.so在安装目录lib中(/home/linux/Software/libiconv-1.14-arm/lib)

#### 修改 ####
修改ARM板中/etc/profile文件，向其中添加以下内容
```bash
export TSLIB_ROOT=/usr
export QT_ROOT=/opt/qt
export TSLIB_TSDEVICE=/dev/touchscreen
export TSLIB_CALIBFILE=/etc/pointercal
export TSLIB_CONFFILE=/usr/etc/ts.conf
export TSLIB_PLUGINDIR=/usr/lib/ts
export TSLIB_CONSOLEDEVICE=none
export TSLIB_FBDEVICE=/dev/fb0
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$TSLIB_ROOT/lib
export QWS_MOUSE_PROTO=tslib:/dev/touchscreen
export LD_LIBRARY_PATH=/lib:/usr/lib:/usr/local/lib:/$QT_ROOT/lib:$LD_LIBRARY_PATH
export QT_QPA_PLATFORM_PLUGIN_PATH=$QT_ROOT/plugins
export QT_QPA_PLATFORM=linuxfb:tty=/dev/fb0
export QT_QPA_FONTDIR=$QT_ROOT/lib/fonts
export LD_PRELOAD=/usr/lib/preloadable_libiconv.so:$TSLIB_ROOT/lib/libts.so
export QT_QPA_GENERIC_PLUGINS=tslib
```
#### 拷贝 ####
假设将编译好的Qt5.7.0安装在了~/Software/Qt5.7.0-arm目录中

- 将PC端 ~/Software/Qt5.7.0-arm/目录下的lib 拷贝到 ARM板中的 /opt/qt目录

- 将PC端 ~/Software/Qt5.7.0-arm/目录下的plugins 拷贝到 ARM板中的 /opt/qt目录

- 创建Qt字库目录 `mkdir /opt/qt/lib/fonts`，将自己已有的带中文的字库拷贝到这个目录中，Qt程序就可以显示出汉字

- 将交叉编译好的preloadable_libiconv.so 拷贝到ARM板中的 /usr/lib目录

### 配置Qt Creator ###

#### 添加Qt构建套件 ####

- **设备**选择**通用Linux类型的设备**，如果没有需要创建

- 编译器选择交叉编译链中的g++

- 调试器选择成交叉编译链中的gdb 

![uaq1](/img/uaq1.png)

创建的通用Linux类型的设备：
![uaq2](/img/uaq2.png)

其中，超时时间需要设置的稍微长一点，第一次连接设备时会稍微久一点，设置好以后，可以点击Test测试一下是否能连到设备，测试时需要ARM板开启ssh服务(ARM板上运行 /usr/local/sbin/sshd)，使用交叉网线连接PC机和ARM板，并将IP设置在一个网段，使用ping命令能够ping通。

#### 创建项目 ####
所创建的项目，在选择为arm版Qt的同时，还需要在pro文件中添加以下内容
```bash
target.path=/home/root
INSTALLS += target
```

### 结束 ###
前面的都弄好以后，远程调试需要的条件：

- ARM板和PC机IP在同一个网段

- ARM板中已开启ssh服务(/usr/local/sbin/sshd) 

- 在Qt项目的pro文件中已添加
```bash
target.path=/home/root
INSTALLS += target
```