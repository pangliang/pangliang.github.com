---
layout: post
categories : ios
tags : []
excerpt: 浅析如何在MacOS上编译同时支持32bit和64bit的universal库文件, 并记录编译libimobiledevice的过程
---
{% include JB/setup %}

原来总是依赖homebrew, 然后这个东西总是会吧-arch i386 -arch x86_64给你去掉, 无法编译出universal的包
还不如自己下载包编译, 主要就是注意有些包使用了汇编代码(加速), 去掉这个功能或者分开编译即可

###含有ABI的编译 

通过指定分别CFLAGS,CXXFLAGS,LDFLAGS 为-arch i386 或者 -arch x86_64, 并且将ABI对应设置为'32' 或 '64'
分别编译出32bit和64bit版本, 并把两个版本的库文件分别保存   
然后通过lipo工具将两个版本的库合成一个, 命令类似'lipo -create 32bit.dylib 64bit.dylib -output universal.dylib'

###编译工具的不同

普通编译时有遇到'llvm-gcc-4.2: -E, -S, -save-temps and -M options are not allowed with multiple -arch flags'错误,
也就是说这个gcc不支持通知编译多个arch  
看下`llvm-gcc-4.2 -v`:
 
    Using built-in specs.
    Target: i686-apple-darwin11
    Configured with: /private/var/tmp/llvmgcc42/llvmgcc42-2336.11~148/src/configure --disable-checking --enable-werror --prefix=/Applications/Xcode.app/Contents/Developer/usr/llvm-gcc-4.2 --mandir=/share/man --enable-languages=c,objc,c++,obj-c++ --program-prefix=llvm- --program-transform-name=/^[cg][^.-]*$/s/$/-4.2/ --with-slibdir=/usr/lib --build=i686-apple-darwin11 --enable-llvm=/private/var/tmp/llvmgcc42/llvmgcc42-2336.11~148/dst-llvmCore/Developer/usr/local --program-prefix=i686-apple-darwin11- --host=x86_64-apple-darwin11 --target=i686-apple-darwin11 --with-gxx-include-dir=/usr/include/c++/4.2.1
    Thread model: posix
    gcc version 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
    
看到这里有个'Target: i686-apple-darwin11',明显, 这个编译器是i686平台的

再看`clang -v`:

    Apple LLVM version 4.2 (clang-425.0.24) (based on LLVM 3.2svn)
    Target: x86_64-apple-darwin12.2.0
    Thread model: posix
    
看到`Target: x86_64-apple-darwin12.2.0`, 也就是是个64位的编译器, 能支持多个arch; 通过configure中加`CC=clang CXX=clang++`指定


###LIBS

如果某个库使用了第三方库, 且需要指定需要链接的库, 比如 

    ./configure libabcd_CFLAGS=/usr/local/ libabcd_LIBS=/usr/local/ 
    
那么就需要`LIBS="-labcd"` 告诉编译器去哪里找这个库


### libimobiledevie 编译示例

1. gmp

    需要分i386 和 x86_64 分别编译, 然后lipo合并

    1) i386
    
        make build-i386
        cd  build-i386
        ../configure CFLAGS='-arch i386' LDFLAGS='-arch i386' CXXFLAG='-arch i386' --prefix=/usr/local ABI=32
        make -j 4
        sudo make install
        cd ..
        
        #改名, 以备后面使用
        sudo mv /usr/local/lib/libgmp.10.dylib /usr/local/lib/libgmp.10.i386.dylib
        sudo mv /usr/local/lib/libgmp.a /usr/local/lib/libgmp.i386.a
     
    2) x86_64
    
        make build-x84_64
        cd build-x84_64
        ../configure CFLAGS='-arch x86_64' LDFLAGS='-arch x86_64' CXXFLAG='-arch x86_64' --prefix=/usr/local ABI=64
        make -j 4
        sudo make install
        cd ..
        
        #改名, 以备后面使用
        sudo mv /usr/local/lib/libgmp.10.dylib /usr/local/lib/libgmp.10.x84_64.dylib
        sudo mv /usr/local/lib/libgmp.a /usr/local/lib/libgmp.x84_64.a
        
    3) 生成universal的包
    
        cd /usr/local/lib
        sudo lipo -create libgmp.10.i386.dylib libgmp.10.x84_64.dylib -output libgmp.10.dylib
        sudo lipo -create libgmp.i386.a libgmp.x84_64.a -output libgmp.a
        
    4)验证下
    
        lipo -info libgmp.dylib libgmp.a
                Architectures in the fat file: libgmp.dylib are: i386 x86_64
                Architectures in the fat file: libgmp.a are: i386 x86_64
    
    5) 修改头文件 
    
        sudo vi /usr/local/include/gmp.h 
            #if defined(__i386__)
            #define __GMP_HAVE_HOST_CPU_FAMILY_power   0
            #define __GMP_HAVE_HOST_CPU_FAMILY_powerpc 0
            #define GMP_LIMB_BITS                      32
            #define GMP_NAIL_BITS                      0
            #elif defined(__x86_64__)
            #define __GMP_HAVE_HOST_CPU_FAMILY_power   0
            #define __GMP_HAVE_HOST_CPU_FAMILY_powerpc 0
            #define GMP_LIMB_BITS                      64
            #define GMP_NAIL_BITS                      0
            #elif defined(__ppc__)
            #define __GMP_HAVE_HOST_CPU_FAMILY_power   0
            #define __GMP_HAVE_HOST_CPU_FAMILY_powerpc 1
            #define GMP_LIMB_BITS                      32
            #define GMP_NAIL_BITS                      0
            #elif defined(__powerpc64__)
            #define __GMP_HAVE_HOST_CPU_FAMILY_power   0
            #define __GMP_HAVE_HOST_CPU_FAMILY_powerpc 1
            #define GMP_LIMB_BITS                      64
            #define GMP_NAIL_BITS                      0
            #else
            #error Unsupported architecture
            #endif
            
2.nettle, 同上分别编译后合并

3.pll-ket, configure时直接 CFLAGS='-arch i386 -arch x86_64' CXXFLAGS.... 编译universal

4.libimobiledevice 

    export PKG_CONFIG_PATH=/usr/lib/pkgconfig:/usr/local/lib/pkgconfig
    ./configure --disable-dependency-tracking --prefix=/usr/local/ --without-cython CFLAGS='-arch i386 -arch x86_64' LDFLAGS='-arch i386 -arch x86_64' CXXFLAG='-arch i386 -arch x86_64' CC=clang CXX=clang++ LIBS='-lplist -lusbmuxd -lcrypto -lssl'



    
                
        
        
        
        