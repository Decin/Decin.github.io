---
title:  "iOS交叉编译环境搭建(ubuntu16.04 server)"
categories: system
tags: [vlc, 播放器, 音视频]
---

# 介绍
在linux环境下搭建ios编译环境, 主要需要llvm, clang, cctools-port和ios_sdk, 以及一些编译必备工具
以下是在Ubuntu 16.04.6 server下实践llvm-4.0.1, cfe-4.0.1(clang), iPhoneOS10.0.sdk


# 所需工具

* Ubuntu 16.04.6服务器（64位）(http://ftp.sjtu.edu.cn/ubuntu-cd/16.04.6/ubuntu-16.04.6-desktop-amd64.iso)
* llvm和clang（4.0.1）(http://releases.llvm.org/download.html#5.0.0)
* openssl（https://github.com/openssl/openssl）
* automake（http://ftp.gnu.org/gnu/automake/）
* cmake（https://cmake.org/download/）
* autogen（http://ftp.gnu.org/gnu/autogen/）
* libtool（http://ftp.gnu.org/gnu/libtool/）
* autoconf（http://ftp.gnu.org/gnu/autoconf/）
* libssl-dev（https://pkgs.org/download/libssl-dev）
* cctools-port（https://github.com/tpoechtrager/cctools-port）
* iOS-SDK（http://resources.airnativeextensions.com/ios/）

尝试通过apt安装这些必备工具
$ sudo apt update
$ sudo apt install git gcc cmake libssl-dev libtool autoconf automake clang-4.0

安装失败则通过手动安装, wget下载源码后安装, 也可以通过下载.deb文件dpkg -i安装

注意:如果源无法访问, 请考虑翻墙, 或者更换apt安装源(类似于:https://www.cnblogs.com/gabin/p/6519352.html)

安装完clang看看位置如果不是usr/bin
执行下面命令制作软链接，只为保险不是必需的....
$ sudo ln -s /usr/bin/clang-4.0 /usr/bin/clang
$ sudo ln -s /usr/bin/clang++-4.0 /usr/bin/clang++

# 安装llvm和clang
```
#下载llvm源码 
$ wget http://llvm.org/releases/4.0.1/llvm-4.0.1.src.tar.xz
$ tar xf llvm-4.0.1.src.tar.xz 
$ mv llvm-4.0.1.src llvm 

#下载clang源码 
$ cd llvm/tools 
$ wget http://llvm.org/releases/4.0.1/cfe-4.0.1.src.tar.xz
$ tar xf cfe-4.0.1.src.tar.xz 
$ mv cfe-4.0.1.src clang 
$ cd ../.. 

#下载clang-tools-extra源码 可选 
$ cd llvm/tools/clang/tools 
$ wget http://llvm.org/releases/4.0.1/clang-tools-extra-4.0.1.src.tar.xz
$ tar xf clang-tools-extra-4.0.1.src.tar.xz 
$ mv clang-tools-extra-4.0.1.src extra 
$ cd ../../../.. 

#下载compiler-rt源码 可选 
$ cd llvm/projects 
$ wget http://llvm.org/releases/4.0.1/compiler-rt-4.0.1.src.tar.xz
$ tar xf compiler-rt-4.0.1.src.tar.xz 
$ mv compiler-rt-4.0.1.src compiler-rt 
$ cd ../.. 


$ mkdir llvmbuild 
$ cd llvmbuild 


#正常套路安装 
#设置配置 
#–prefix=directory — 设置llvm编译的安装路径(default/usr/local). 
#–enable-optimized — 是否选择优化(defaultis NO)，yes是指安装一个Release版本. 
#–enable-assertions — 是否断言检查(default is YES). 
$ ../llvm/configure --enable-optimized --enable-targets=host-only --prefix=/usr/bin 
#如果不能通过configure按照提示使用cmake $ cmake -G "Unix Makefiles" -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DCLANG_DEFAULT_CXX_STDLIB=libc++ -DCMAKE_BUILD_TYPE="Release" ../llvm

#构建 
$ cmake ../llvm make 

#安装 
$ sudo make install
```

# 打包ios sdk
在Mac中打包ios sdk, tmp为临时目录, 可以在任意位置, 此步骤参考https://github.com/tpoechtrager/cctools-port/tree/master/usage_examples/ios_toolchain

```
$ SDK=$(ls -l /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs | grep " -> iPhoneOS.sdk" | head -n1 | awk '{print $9}')
$ cp -r /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk /tmp/$SDK 1>/dev/null
$ cp -r /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1 /tmp/$SDK/usr/include/c++ 1>/dev/null

$ pushd /tmp
$ tar -cvzf $SDK.tar.gz $SDK
$ rm -rf $SDK
$ mv $SDK.tar.gz ~  #此时最好放在~目录下, 由于cctools-port中usage_examples/ios_toolchain/build.sh可能会出问题
$ popd
```

还可以在http://resources.airnativeextensions.com/ios/下载ios sdk, 但是和上面的略有不同, 不包括拷贝的/c++/v1头文件

# 制作工具链
参考(https://github.com/tpoechtrager/cctools-port/tree/master/usage_examples/ios_toolchain)

## iOS arm64工具链

```
$ cd cctools-port
$ IPHONEOS_DEPLOYMENT_TARGET=9.0 usage_examples/ios_toolchain/build.sh ~/iPhoneOS10.0.sdk.tar.gz arm64
```

制作工具链成功后会提示 ***all done***
将生成的工具链移到 /usr/local/ 目录并更名为ios-arm64

```
$ sudo mv usage_examples/ios_toolchain/target /usr/local/ios-arm64
```

使用rename命令重命名前缀以与armv7区分开来把arm-前缀改为aarch64-前缀(苹果的arm64叫aarch64)

```
$ rename 's/arm-/aarch64-/' /usr/local/ios-arm64/bin/*
```

将库文件拷贝一份，放进公共库/usr/lib

```
$ sudo cp /usr/local/ios-arm64/lib/libtapi.so /usr/lib
最后将工具链的bin目录加入PATH，方便调用
$ export PATH=$PATH:/usr/local/ios-arm64/bin
```

## iOS armv7工具链 
```
$ IPHONEOS_DEPLOYMENT_TARGET=9.0 usage_examples/ios_toolchain/build.sh ~/iPhoneOS10.0.sdk.tar.gz armv7
```

制作工具链成功后会提示 ***all done***
将生成的工具链移到 /usr/local/ 目录并更名为ios-armv7

```
$ sudo mv usage_examples/ios_toolchain/target /usr/local/ios-armv7
将库文件拷贝一份，放进公共库/usr/lib
$ sudo cp /usr/local/ios-armv7/lib/libtapi.so /usr/lib
最后将工具链的bin目录加入PATH，方便调用
$ export PATH=$PATH:/usr/local/ios-armv7/bin
```

也可以合并工具链, 读者可以自行尝试

到这里已经可以通过工具链打包库给ios端用了
类似通过clang和ar打包静态库在真机上测试, 下面是一个大概例子

```
$ arm-apple-darwin11-clang -c ts_scan_program.c ts_scanner.c ../queue/ns_variable_queue.c -I ../queue/ -I ../libdvbpsi-1.3.2/src/ -I ../libdvbpsi-1.3.2/src/tables/ -I ../libdvbpsi-1.3.2/src/descriptors
$ arm-apple-darwin11-ar -cr libtsscan_m.a ./*.o
```

还可以模仿苹果的xcodebuild, 直接在linux环境下编译xcode项目, https://github.com/facebook/xcbuild

参考:
https://blog.csdn.net/fyf786452470/article/details/79160670
https://www.jianshu.com/p/d99995927527


# 遇到的问题

- *** building apple-libtapi ***
...
Scanning dependencies of target install-libtapi
...
gcc: error trying to exec 'cc1obj': execvp: No such file or directory
Makefile:558: recipe for target 'libobjc_la-NSBlocks.lo' failed
make[1]: *** [libobjc_la-NSBlocks.lo] Error 1
make[1]: *** Waiting for unfinished jobs....
libtool: compile: gcc -DPACKAGE_NAME=\"cctools\" -DPACKAGE_TARNAME=\"cctools\" -DPACKAGE_VERSION=\"895\" "-DPACKAGE_STRING=\"cctools 895\"" -DPACKAGE_BUGREPORT=\"t.poechtrager@gmail.com\" -DPACKAGE_URL=\"\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DLFCN_H=1 -DLT_OBJDIR=\".libs/\" -DEMULATED_HOST_CPU_TYPE=12 -DEMULATED_HOST_CPU_SUBTYPE=0 -D__STDC_LIMIT_MACROS=1 -D__STDC_CONSTANT_MACROS=1 -DHAVE_EXECINFO_H=1 -I. -I../../../../../cctools/libobjc2 -DTYPE_DEPENDENT_DISPATCH -DGNUSTEP -D__OBJC_RUNTIME_INTERNAL__=1 -D_XOPEN_SOURCE=500 -D__BSD_VISIBLE=1 -D_DEFAULT_SOURCE=1 -DNO_SELECTOR_MISMATCH_WARNINGS -isystem /home/shifttime/ios/build_en/cctools-port/usage_examples/ios_toolchain/target/include -std=gnu99 -Wall -O3 -c ../../../../../cctools/libobjc2/Protocol2.m  -fPIC -DPIC -o .libs/libobjc_la-Protocol2.o
gcc: error trying to exec 'cc1obj': execvp: No such file or directory
Makefile:561: recipe for target 'libobjc_la-Protocol2.lo' failed
make[1]: *** [libobjc_la-Protocol2.lo] Error 1
make[1]: Leaving directory '/home/shifttime/ios/build_en/cctools-port/usage_examples/ios_toolchain/tmp/cctools/libobjc2'
Makefile:414: recipe for target 'all-recursive' failed
make: *** [all-recursive] Error 1
	*** checking toolchain ***
	cannot invoke compiler!

- linux terminal 输入命令有历史记录 $ history
- ~/.bash_history HISTSIZE=1000(default)
- Ubuntu缺省情况下，并没有提供C/C++的编译环境，因此还需要手动安装。但是如果单独安装gcc以及g++比较麻烦，幸运的是，Ubuntu提供了一个build-essential软件包。查看该软件包的依赖关系：
apt-cache depends build-essential
gcc: error trying to exec 'cc1obj': execvp: No such file or directory
apt Install gobjc (http://security.ubuntu.com/ubuntu/pool/universe/g/gcc-5/)
https://www.kubuntuforums.net/showthread.php/35193-gcc-4-2-error-trying-to-exec-cc1obj-execvp-No-such-file-or-directory

- $ apt install git 
出现如下错误
E: Unmet dependencies. Try 'apt-get -f install' with no packages…
原因:
在新版的Ubuntu下,例如Ubuntu 14.04或者16.04一般是不会出现broken dependencies,或者出现unmet dependencies, 但是如果我们使用dpkg强制安装了某些deb包,或者在build-dep的是否手动更改了某些Packages的文件和版本时, 那么在再次使用apt-get install或者build-dep来安装library和packages的时就很可能出现问题.
按照提示输入 $ apt-get -f install


- LLVM最新的4.0.1版本已经不能通过configure/make来编译安装了，它只支持CMake编译。
$ ../llvm/configure  --enable-optimized --enable-targets=host-only CC=gcc CXX=g++
需要通过cmake编译, 例如:
$ cmake -G "Unix Makefiles" -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DCLANG_DEFAULT_CXX_STDLIB=libc++ -DCMAKE_BUILD_TYPE="Release" ../llvm
参考:https://typecodes.com/linux/cmakellvmclang4.html
