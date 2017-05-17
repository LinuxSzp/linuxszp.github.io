---
title: Qt编译4.8.6-Mac
date: 2016-09-10 11:36:19
categories: [教程]
tags: [Qt]
---

环境：
  - OS X: 10.11.5

由于之前linux和windows下写的大多数的Qt项目用的版本都是4.8.x的，为了能在Mac下更好的调试和编译之前的代码，就打算在Mac下也编译一个4.8.x版本。

4.8.x版本的Qt是一个很老的版本了，不管在linux下，还是在Mac下编译都是一个很麻烦的事情，因为总不是一帆风顺的，沉淫在google，各大论坛许久，终于编译通过，我把我编译的过程和错误解决的方法记录下来，供自己日后查看，和帮助后来者脱坑。

**如果你的 OS X 是10.9以上的，那就必须选4.8.6以上版本的**

<!-- more -->

## **下载** ##
下载4.8.6源码包 

下载地址：[http://download.qt.io/archive/qt/4.8/4.8.6/qt-everywhere-opensource-src-4.8.6.tar.gz](http://download.qt.io/archive/qt/4.8/4.8.6/qt-everywhere-opensource-src-4.8.6.tar.gz)

## **桌面版** ##

### configure ###
```bash
./configure -prefix ~/Software/Qt4.8.6/ -system-zlib -qt-libtiff -confirm-license 
-qt-libpng -qt-libjpeg -opensource 
-nomake demos -nomake examples 
-cocoa -fast -release -platform unsupported/macx-clang-libc++ 
-no-qt3support -nomake docs -arch x86_64
```

### 编译和安装 ###
```bash
make -j3
make install
```

### 错误 ###
 -
```bash
clang++ -c -pipe -stdlib=libc++ -mmacosx-version-min=10.7 -O2 -arch x86_64 -fvisibility=hidden -fvisibility-inlines-hidden -Wall -W -fPIC -DQT_SHARED -DQT_BUILD_GUI_LIB -DQT_NO_USING_NAMESPACE -DQT_NO_CAST_TO_ASCII -DQT_ASCII_CAST_WARNINGS -DQT_MOC_COMPAT -DQT_USE_QSTRINGBUILDER -DQT_USE_BUNDLED_LIBPNG -DPNG_ARM_NEON_OPT=0 -DQT_NO_CUPS -DQT_NO_LPR -DQT_NO_OPENTYPE -DQT_NO_STYLE_WINDOWSVISTA -DQT_NO_STYLE_WINDOWSXP -DQT_NO_STYLE_GTK -DQT_NO_STYLE_WINDOWSCE -DQT_NO_STYLE_WINDOWSMOBILE -DQT_NO_STYLE_S60 -DQ_INTERNAL_QAPP_SRC -DQT_NO_DEBUG -DQT_CORE_LIB -DQT_HAVE_MMX -DQT_HAVE_3DNOW -DQT_HAVE_SSE -DQT_HAVE_MMXEXT -DQT_HAVE_SSE2 -DQT_HAVE_SSE3 -DQT_HAVE_SSSE3 -DQT_HAVE_SSE4_1 -DQT_HAVE_SSE4_2 -DQT_HAVE_AVX -D_LARGEFILE64_SOURCE -D_LARGEFILE_SOURCE -I../../mkspecs/unsupported/macx-clang-libc++ -I. -I../../include/QtCore -I../../include -I../../include/QtGui -I.rcc/release-shared -Iimage -I../3rdparty/libpng -I../3rdparty/harfbuzz/src -Idialogs -I.moc/release-shared -I.uic/release-shared -F/private/tmp/qt20150610-44719-1qwdhre/qt-everywhere-opensource-src-4.8.7/lib -o .obj/release-shared/qgraphicssystem_mac.o painting/qgraphicssystem_mac.cpp
painting/qpaintengine_mac.cpp:345:19: error: use of undeclared identifier 'CMGetProfileByAVID'
    CMError err = CMGetProfileByAVID((CMDisplayIDType)displayID, &displayProfile);
                  ^
painting/qpaintengine_mac.cpp:348:9: error: use of undeclared identifier 'CMCloseProfile'
        CMCloseProfile(displayProfile);

```
**解决办法：**
 需要打一个补丁，补丁地址：[https://github.com/Homebrew/formula-patches/blob/master/qt/el-capitan.patch](https://github.com/Homebrew/formula-patches/blob/master/qt/el-capitan.patch)
 
 你也可以自己修改，编辑src/gui/painting/qpaintengine_mac.cpp b/src/gui/painting/qpaintengine_mac.cpp文件
 ```bash
 CGColorSpaceRef QCoreGraphicsPaintEngine::macDisplayColorSpace(const QWidget *widget)
 {
 	....
 	
 	//屏蔽下面一段代码
 	// Get the color space from the display profile.
/*
    CGColorSpaceRef colorSpace = 0;
    CMProfileRef displayProfile = 0;
    CMError err = CMGetProfileByAVID((CMDisplayIDType)displayID, &displayProfile);
    if (err == noErr) {
        colorSpace = CGColorSpaceCreateWithPlatformColorSpace(displayProfile);
        CMCloseProfile(displayProfile);
    }
*/
    CGColorSpaceRef colorSpace = CGDisplayCopyColorSpace(displayID); //新加的一句
    
    ....
 }
 ```
### 安装完成后工作 ###
- 拷贝源码中的src文件夹到安装目录
- 拷贝源码中include中的文件夹到安装目录下的include，只拷贝安装目录中include没有的文件夹
- 如果编译Qt程序出现`error: symbol(s) not found for architecture x86_64`，将编译器选成gcc

最后提供一个Qt一些版本的补丁下载地址：[https://github.com/sandym/qt-patches](https://github.com/sandym/qt-patches) 
