---
title: Fedora21搭建Qt4.8.6的X11版本+EmbeddedX86版本+ARM版本
date: 2017-05-19 11:48:31
categories: [教程]
tags: [Qt]
---

**环境:** 

 - Feodra21 32位

这次安装还是比较顺利的，比起3年前第一次装这家伙要快的多了。

需要说明的是，独立编译qvfb，是在configure配置为x11版本的前提下编译的，因为qvfb属于应用程序，是在x11平台下运行的。

如果是新装Fedora系统，请自行使用*yum update*命令更新后再安装Qt。

<!-- more -->

## **安装依赖** ##

``` bash
$ sudo yum install libX11-devel.i686
$ sudo yum install libXext-devel.i686
$ sudo yum install libXtst-devel.i686
$ sudo yum install libtool.i686
$ sudo yum install gcc-c++
```

## **错误** ##

### 编译生成qvfb时出现错误 ###
 
``` bash
 
 .obj/release-shared/qanimationwriter.o: In function `QAnimationWriter::QAnimationWriter(QString const&, char const*)':
 
qanimationwriter.cpp:(.text+0xbf): undefined reference to `png_create_write_struct'

qanimationwriter.cpp:(.text+0xca): undefined reference to `png_create_info_struct'

qanimationwriter.cpp:(.text+0xe0): undefined reference to `png_set_compression_level'

qanimationwriter.cpp:(.text+0xff): undefined reference to `png_set_write_fn'

```
 
  - 解决方法
  
1.找到本机上的libpng库

``` bash
$ locate libpng
```
 
 我的电脑上是在
``` bash
 ...
/usr/lib/libpng16.so.16
 
/usr/lib/libpng16.so.16.10.0
...
```
 如果没找到需要安装
 
2.建立软连接
 
``` bash 
$ sudo ln -s /usr/lib/libpng16.so.16 /usr/lib/libpng.so
```
3.修改qvfb目录下的Makefile
在LIBS后面添加png库，**-L/usr/lib -lpng**(根据自己的电脑上libpng.so的位置添加)
  ![qvfb](/img/qvfb.png)
 
### qvfb显示不出来汉字 ###

 - 解决方法

从网上下载一种带汉字的字体，我用的是simfang.ttf，将安装好的的Qt EmbeddedX86版本中的**lib/fonts**，fonts文件夹中的字体**全部删掉**(你也可以先备份一下)，将simfang.ttf拷到fonts文件夹中，再运行，汉字可以显示。

## **安装** ##

完成一个版本的安装，安装下一个版本的时候，先清除干净，或删除重新解压。

``` bash 
$ gmake clean
$ gmake confclean
```

### **X11版本** ###
``` bash
$ ./configure -prefix /home/linux/Software/Qt4.8.6 -qvfb
$ gmake -j4
$ gmake install
```

### **编译qvfb** ###
编译qvfb必须是在编译过X11版本之后

进入源码目录下 **tools/qvfb**
```bash
$ qmake
```
将源码目录bin下的qvfb文件拷贝到系统的PATH环境目录下，我的拷贝到了/usr/local/bin

使用方法
``` bash
$ qvfb -width 800 -heiget 600 &
```

### **Embedded X86版** ###
``` bash
$ ./configure -prefix /home/linux/Software/Qt4.8.6-qvfb -embedded x86 
-qt-gfx-qvfb -qt-kbd-qvfb -qt-mouse-qvfb
$ gmake -j4
$ gmake install
```

### **ARM版本** ###

 - 准备工作
 
首先系统中需要有交叉编译工具，如果没有，自己网上下载一个，或者用ARM开发版光盘里面的。

把交叉工具配置到PATH环境中，使用**arm-arago-linux-gnueabi-gcc -v**可以看到版本信息就算成功(以我的为例)

修改源码目录下 **mkspecs/qws/linux-arm-gnueabi-g++/qmake.conf**

将文本中的内容
``` bash
# modifications to g++.conf
QMAKE_CC                = arm-linux-gnueabi-gcc
QMAKE_CXX               = arm-linux-gnueabi-g++
QMAKE_LINK              = arm-linux-gnueabi-g++
QMAKE_LINK_SHLIB        = arm-linux-gnueabi-g++

# modifications to linux.conf
QMAKE_AR                = arm-linux-gnueabi-ar cqs
QMAKE_OBJCOPY           = arm-linux-gnueabi-objcopy
QMAKE_STRIP             = arm-linux-gnueabi-strip
```
修改成你电脑上的交叉工具(以我的为例)
``` bash
# modifications to g++.conf
QMAKE_CC                = arm-arago-linux-gnueabi-gcc
QMAKE_CXX               = arm-arago-linux-gnueabi-g++
QMAKE_LINK              = arm-arago-linux-gnueabi-g++
QMAKE_LINK_SHLIB        = arm-arago-linux-gnueabi-g++

# modifications to linux.conf
QMAKE_AR                = arm-arago-linux-gnueabi-ar cqs
QMAKE_OBJCOPY           = arm-arago-linux-gnueabi-objcopy
QMAKE_STRIP             = arm-arago-linux-gnueabi-strip
```
 - 安装
 
``` bash
$ ./configure -prefix /home/linux/Software/Qt4.8.6-arm -opensource 
-confirm-license -release -shared -embedded arm 
-xplatform qws/linux-arm-gnueabi-g++ -depths 16,18,24 
-fast -optimized-qmake -pch -qt-sql-sqlite -qt-libjpeg -qt-zlib 
-qt-libpng -qt-freetype -little-endian -host-little-endian 
-no-qt3support -no-libtiff -no-libmng -no-opengl -no-mmx -no-sse 
-no-sse2 -no-3dnow -no-webkit -no-qvfb -no-phonon -no-nis 
-no-opengl -no-cups -no-glib -no-xcursor -no-xfixes -no-xrandr 
-no-xrender -no-separate-debug-info -nomake examples 
-nomake tools -nomake docs
$ gmake -j4
$ gmake install
```