# 从源文件编译gcc

乔俊峰 Junfeng Qiao， 2017-10-16

[TOC]

最近用到pymatgen库，安装好Anaconda3-5.0.0.1-Linux-x86_64后import的时候提示需要`GLIBC_2.14`
```python
Python 3.6.2 |Anaconda, Inc.| (default, Sep 30 2017, 18:42:57) 
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import pymatgen
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/__init__.py", line 53, in <module>
    from pymatgen.core import *
  File "/home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/core/__init__.py", line 9, in <module>
    from .structure import Structure, IStructure, Molecule, IMolecule
  File "/home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/core/structure.py", line 30, in <module>
    from pymatgen.core.lattice import Lattice
  File "/home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/core/lattice.py", line 18, in <module>
    from pymatgen.util.coord_utils import pbc_shortest_vectors
  File "/home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/util/coord_utils.py", line 11, in <module>
    from . import coord_utils_cython as cuc
ImportError: /lib/libc.so.6: version `GLIBC_2.14' not found (required by /home/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen/util/coord_utils_cython.cpython-36m-x86_64-linux-gnu.so)
```

于是编译glic，发现服务器上的gcc及binutils版本太低，遂开始编译gcc，本文记录一下gcc的编译过程及遇到的问题和解决方案。

编译gcc需要gmp、mpfr、mpc、isl。顺便提前安装好texinfo。
注意build的objdir和源代码的srcdir最好不要在一个目录或其中一个在另一个目录里，互相独立最好。另外新建一个installdir。
基本都是编译三部曲，configure，make，make install。

## 1. 安装texinfo-6.5.tar.xz
  ```
  ../src/configure --prefix=/home/qiao/bin/texinfo-6.5
  make
  make install
  ```
  然后记得把texinfo-6.5/bin加入到`~/.bashrc`的`PATH`中
## 2. 安装gmp-6.1.2.tar.xz
  ```
  ../src/configure --prefix=/qiao/home/bin/gnu/gmp-6.1.2
  make
  make install
  ```
## 3. 安装mpfr-3.1.6.tar.xz
  mpfr依赖于gmp
  ```
  ../src/configure --prefix=/home/qiao/bin/gnu/mpfr-3.1.6 --with-gmp-include=/home/qiao/bin/gnu/gmp-6.1.2/include --with-gmp-lib=/home/qiao/bin/gnu/gmp-6.1.2/lib
  make
  make install
  ```
## 4. 安装mpc-1.0.3.tar.gz
  mpc依赖于gmp和mpfr
  ```
  ../configure --prefix=/home/qiao/bin/gnu/mpc-1.0.3 --with-gmp-include=/home/qiao/bin/gnu/gmp-6.1.2/include --with-gmp-lib=/home/qiao/bin/gnu/gmp-6.1.2/lib --with-mpfr-include=/home/qiao/bin/gnu/mpfr-3.1.6/include --with-mpfr-lib=/home/qiao/bin/gnu/mpfr-3.1.6/lib
  make
  make install
  ```
## 5. 安装isl-0.18.tar.bz2
  isl依赖于gmp
  ```
  ../configure --prefix=/home/qiao/bin/gnu/isl-0.18 --with-gmp-prefix=/home/qiao/bin/gnu/gmp-6.1.2
  make
  make install
  ```

## 6. 安装binutils-2.29.1.tar.xz
  ```
  ../configure --prefix=/home/qiao/bin/gnu/binutils-2.29.1
  ```
  同样记得把各个软件installdir下面的bin目录加入`PATH`，lib加入`LD_LIBRARY_PATH`，include <font color=red>不要</font>加入到`C_INCLUDE_PATH`之类的环境变量。后面有生动的案例告诉我们用gcc的`-I`要安全的多。
## 7. 安装gcc-7.2.0.tar.xz
  编译gcc的时候困难重重。
  遇到的错误有：

  1. libgfortran \_\_float128
    ```
    In file included from ./kinds.h:75:0,
    from /gcc-4.8.0/libgfortran/libgfortran.h:232,
    from /gcc-4.8.0/libgfortran/fmain.c:4:
    
    /gcc-4.8.2/libgfortran/kinds-override.h:40:5: error: #error "Where has   __float128 gone?"
    
    #   error "Where has __float128 gone?"
    ```
    参见
    [http://gcc.1065356.n8.nabble.com/Bug-bootstrap-53731-New-4-7-Bootstrap-fails-for-libgfortran-on-Solaris-10-x86-with-error-quot-Where--td435065.html]
    及
    [http://www.nlpers.pub/2015/06/23/centos4-install-theano/]
    原因是服务器上安装了intel parrallel studio，icc的header file和gcc的header混合在一起了。
    使用`cpp -v /dev/null`或``` `gcc -print-prog-name=cc1` -v```可得到
    ```
    #include "..." search starts here:
    #include <...> search starts here:
    /opt/intel/composer_xe_2013.0.079/mkl/include
    /opt/intel/composer_xe_2013.0.079/tbb/include
    /usr/local/include
    /usr/lib/gcc/x86_64-kylin-linux/4.4.7/include
    /usr/include
    End of search list.
    ```
    解决方法：
    清空环境变量`LD_LIBRARY_PATH`, `PATH`, `C_INCLUDE_PATH`, `CPATH`,每个环境变量按需设置，仔细核对，避免icc和gcc的header，lib相互污染。
  
  2. isl的stdint.h id.h 
    `unknown type name uint32_t`
    id.h包含stdint.h

    ``` c
    #ifndef ISL_ID_H
    #define ISL_ID_H
    
    #include <isl/ctx.h>
    #include <isl/list.h>
    #include <isl/printer_type.h>
    #include <isl/stdint.h>
    ```
  
    此stdin.h内容是

    ``` c
    #ifndef _ISL_INCLUDE_ISL_STDINT_H
    #define _ISL_INCLUDE_ISL_STDINT_H 1
    #ifndef _GENERATED_STDINT_H
    #define _GENERATED_STDINT_H "isl 0.18"
    /* generated using gnu compiler gcc (GCC) 4.6.1 */
    #define _STDINT_HAVE_STDINT_H 1
    #include <stdint.h>
    #endif
    #endif
    ```

    如果在`C_INCLUDE_PATH`中加入了`/home/qiao/bin/gnu/isl-0.18/include/isl`,那么preprocesser会先找到这个stdint.h，无法指向glibc的stdint.h，导致`uint32_t`没有定义。

    解决方法：

      - 可以把此处的`#include <stdint.h>`改为`#include "/usr/includestdint.h"`，使用绝对路径。

      - 更好的办法是不要使用`C_INCLUDE_PATH`，而是在Makefile里`-I`参数来把isl的头文件路径加进去。
      
  3. string.h error: '__locale_t' has not been declared
    原因参见[https://stackoverflow.com/questions/24738059/c-error-locale-t-has-not-been-declared]
    解决办法：修改Makefile，加上flag `-D__USE_XOPEN2K8`

  4. 关于优化问题
    不清楚为什么，有时候`-O2`默认优化选项会产生莫名其妙的问题，改成`-O0`问题消失。

## 8. 安装glibc-2.26.tar.xz
  继续出现错误。。。
  ```
  checking installed Linux kernel header files... missing or too old!
  configure: error: GNU libc requires kernel header files from
  Linux 3.2.0 or later to be installed before configuring.
  The kernel header files are found usually in /usr/include/asm and
  /usr/include/linux; make sure these directories use files from
  Linux 3.2.0 or later.  This check uses <linux/version.h>, so
  make sure that file was built correctly when installing the kernel header
  files.  To use kernel headers not from /usr/include/linux, use the
  configure option --with-headers.
  ```
  看来是linux kernel版本太低了。2.6.32版本太老了。。
  ```
  uname -a
  Linux ln3 2.6.32-358.11.1.2.ky3.1.x86_64 #1 SMP Mon Jul 8 13:05:58 CST 2013 x86_64 x86_64 x86_64 GNU/Linux
  ```
  准备弃坑了，再编译个内核太麻烦了。
  似乎编译了glibc后，虽然可以通过`LD_LIBRARY_PATH`来使链接到新版本的glibc，但是

  ```
  ldd ~/qiao/bin/anaconda/lib/python3.6/site-packages/pymatgen-2017.10.16-py3.6-linux-x86_64.egg/pymatgen/util/coord_cython.cpython-36m-x86_64-linux-gnu.so
        linux-vdso.so.1 =>  (0x00007fffcd5b9000)
        /usr/local/lib/libthbm.so (0x00002b1c6a73e000)
        libpython3.6m.so.1.0 => /home/qiao/bin/anaconda/lib/libpython3.6m.so.1.0 (0x00002b1c6a94b000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00002b1c6aea1000)
        libc.so.6 => /lib/libc.so.6 (0x00002b1c6b0bf000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00002b1c6b452000)
        libutil.so.1 => /lib64/libutil.so.1 (0x00002b1c6b656000)
        librt.so.1 => /lib64/librt.so.1 (0x00002b1c6b85a000)
        libm.so.6 => /lib/libm.so.6 (0x00002b1c6ba62000)
        /lib64/ld-linux-x86-64.so.2 (0x000000323ae00000)
  ```
  可见`ld-linux.so.2`是直接hard-coded到.so里面的。
  所以当有多个glibc的时候，需要用其他办法将`ld-linux.so.2`换成新的版本。
  方法有`chroot`，但是我没有root权限，所以不行。还有`fakechroot`，`proot`等似乎可行。
  除此之外，还有环境变量`LD_PRELOAD`
  参考[https://stackoverflow.com/questions/847179/multiple-glibc-libraries-on-a-single-host]
  个人觉得最好的办法就是`patchelf`了，不过都没有尝试。因为，我发现，达成我的目的，其实有更加简单的方法。。。。

## 9. 最终的解决办法
  与其费事费力编译gcc、glibc，为何不直接重新编译pymatgen？
  github上下载最新的release，卸载掉旧的pymatgen，之后编译安装
  ```
  conda remove pymatgen
  tar xvf pymatgen-2017.10.16.tar.gz
  cd pymatgen-2017.10.16
  python setup.py install
  ```
  再import pymatgen，没有报错，<font color=red>问题就这么解决了。。。。。。</font>

## 10. 附录
  最主要的是看各种manual的installation部分，总会有解决办法的。
  1. Installing GCC [https://gcc.gnu.org/install/]


