Easymake 中文使用手册

# Introduction #

EMake 是一个 Make工具，用来从源文件输出可执行文件。EMake直接从源文件的注释中提取EMake宏指令，来确定这个工程有哪些源文件，他们该如何编译（编译参数，或者头文件路径），并且输出成什么格式（可执行，动态库，静态库，Windows下的窗口模式可执行）。

GNU Make是一个强大的工具，但大部分时候我不会去写一个非常大的需要用Makefile的工程，我经常写一些中小项目，代码在2万行以内，甚至更少（比如几百行）。为每个小工程都编写一个新的Makefile（或者修改一下旧的）是一件非常累的事情（当然你也可以写个足够强大的Makefile来应付我说的情况，但并非大多数人的用法）。

因此我所需要的是像Visual Studio工程管理一样简单直观的编译工具，让我简单的：增加源文件，然后编译他们。这这么简单，所有源代码依赖的头文件都会自动被分析出来，一旦更改，就需要编译他们，否则就跳过。所有繁琐的编译参数都有默认值，在大多数情况下你可以不去管他们，而当你需要时，你可以去定制。

EMake就是一个让你从繁重的Makefile维护中解放出来的工具，它并不是一个GNU Make的 Replacement，只是一个合理的补充，让你用代码中的注释宏来精简的描述工程信息，当你源代码改变的时候，你也可以随手改变工程信息，而且你可以用它指定各种详细的编译细节，真正做到简单，并且真切有力。




# Details #

EMake 从你源代码的宏中得到工程信息。在C代码中使用 "//!" 或者 `"/*!"`作为注释的开头，Emake 就会提取并将该行注释看成 EMake的工程配置信息，比如你的main.c中一行这样的注释:
```
//! exe: file1.c, file2.c
```

当你使用 **"emake.py main.c"**时，Emake会知道该项目有三个源文件: file1.c, file2.c 和main.c 自己. 输出的模式是"exe" (还可以是**"exe/dll/clib/win"**). 接着 emake会分析所有源代码并指出，每个源文件总共使用过了哪些头文件（包括那些头文件又使用了哪些新的头文件）. 然后比较它们和obj文件的时间，如果他们任何一个新于OBJ文件的时间戳，那么，这个对应的源代码就需要重新编译，否则跳过。

Emake会调用 gcc来编译工程中的所有文件，然后连接他们输出成指定的文件格式（可执行文件、动态库、静态库、Windows下窗口模式程序。。。）。




# Installation #

EMake是用Python来开发的，只有一个单一的emake.py文件。你可以下载tar包，然后将emake.py解压到当前。注意，emake当前只支持gcc，如果你要在windows下编译项目，还需要下载mingw来协同工作。

### Install EMake in Unix ###
使用 “./emake.py -install” 来安装，emake.py会自己将自己拷贝到/usr/local/bin下面，同时它将会在 /usr/bin, /usr/local/bin下面搜索gcc可执行文件，如果不在这两个目录，需要用 emake.ini手动指定。

### Install EMake in Windows with mingw ###
将emake.py拷贝到mingw的根目录（就是包含bin, include, lib等的mingw根目录）。因为默认情况下 emake.py会倒 “.\bin\”路径下查找 gcc.exe。

下载地址：http://code.google.com/p/easymake/




# Tutorial #

### Build with multi-source-file project ###

**testmain.c:**
```
#include <stdio.h>
#include "testsrc1.h"

//! mode: exe
//! flag: -O3
//! link: m
//! out: testmain
//! src: testmain.c, testsrc1.c
int main(void)
{
	foo();
	return 0;
}
```

  * 宏 **"//!"** 告诉 emake该行使emake命令，后面是内容。
  * **mode** 表示输出格式是exe（可执行文件），还可以是dll, lib, win。
  * **out** 指明了输出文件的名字是 "testmain"
  * **src** 指明工程中有两个源文件 (testmain.c 和 testsrc1.c)。
  * **flag** 告诉emake在编译时，将 "-O3"参数传递给gcc。
  * **link** 告诉emake在连接时，将"-lm"参数传递给gcc。


**testsrc1.h:**
```
#ifndef __TESTSRC1_H__
#define __TESTSRC1_H__

void foo(void);

#endif
```

**testsrc1.c:**
```
#include <stdio.h>
#include "testsrc1.h"

void foo(void)
{
	printf("This is foo\n");
}
```

这时你就可以用emake来编译你的整个工程了:
```
% python emake.py testmain.c
compiling ...
testmain.c
testsrc1.c
linking ...
```

这时你可以运行你的程序:
```
% ./testmain
This is foo
```

实际上Emake根据testmain.c里面的信息找到所有工程文件，进行作了如下事情：
```
gcc -O3 -c testmain.c -o testmain.o
gcc -O3 -c testsrc1.c -o testsrc1.o
gcc testmain.o testsrc1.o -o testmain -lm
```

每次你调用 "emake.py testmain.c"时, 它都会检测testmain.c和testsrc1.h的文件时间戳并且对比testmain.o的时间戳，如果testmain.o的时间晚于前两个文件的时间戳，那么emake认为这个.o文件是最新的，可以跳过，不需要重新编译testmain.c。

Emake会分析源代码并用递归的方式扫描出该源代码所有的头文件依赖，并且根据这些依赖文件以及源代码本身的时间来判断是否需要重新编译该源代码。


### Intermediate Files Path ###
如果你在testmain.c里面增加了emake宏"//! int: PATH"指明一个临时目录，emake会将所有编译的.o文件放到该临时目录下面，比如：
```
//! mode: exe
//! int: obj
//! out: testmain
//! src: testmain.c
//! src: testsrc1.c
```

Emake会这样调用 gcc：
```
mkdir obj
gcc -c testmain.c -o obj/testmain.o
gcc -c testsrc1.c -o obj/testsrc1.o
gcc obj/testmain.o obj/testsrc1.o -o testmain
```

注意，**src\*关键字可以多次重复使用来定义一大堆项目文件。**

### Easier Instructions ###

如果在 main.c中包含了如下的 emake宏：

```
//! exe: source1.c, source2.c
```

则完全等价于下面的三行指令：
```
//! mode: exe
//! int: obj
//! src: source1.c, source2.c
```

如果你使用了关键字 **exe / dll / clib / win\*并且后面跟随了文件列表以后（这个文件列表可以为空，比如**'//!exe :\n'**，Emake会将其解释成下面的操作：**

  * 将临时目录设置到”./obj” 等同于：’//! int: obj’
  * 将输出模式设置成exe/dll/lib/win其中的一个
  * 将后面描述的文件加入项目文件列表

注意: 使用clib来代表编译成静态库模式是为了避免和设置链接库路径的关键字’lib’冲突，接着emake会做如下事情：

```
mkdir obj
gcc -c source1.c -o obj/source1.o
gcc -c source2.c -o obj/source2.o
gcc -c main.c -o obj/main.o
gcc obj/main.o obj/source1.o obj/source2.o -o main
```


### Build with special GCC parameters ###
```
//! flags: -O3, -g
//! link: stdc++, pthread
//! exe: source1.c, source2.c
```

EMake会为你执行如下指令:

```
mkdir obj
gcc -O3 -g -c source1.c -o obj/source1.o
gcc -O3 -g -c source2.c -o obj/source2.o
gcc -O3 -g -c main.c -o obj/main.o
gcc obj/main.o obj/source1.o obj/source2.o -o main -lstdc++ -lpthread
```

### Build a dynamic link file ###
```
//! dll: src1.c, src2.c
```

### Build a static link file ###
```
//! clib: src1.c, src2.c
```

### Add Including/Librarys Directories ###
```
//! inc: ../ffmpeg/include
//! inc: ../ogg/include
//! lib: ../ffmpeg/lib
//! lib: ../ogg/lib
//! exe: src1, src2
```

EMake会为你做如下事情:
```
mkdir obj
gcc -I../ffmpeg/include -I../ogg/include -c src1.c -o obj/src1.o
gcc -I../ffmpeg/include -I../ogg/include -c src2.c -o obj/src2.o
gcc -I../ffmpeg/include -I../ogg/include -c main.c -o obj/main.o
gcc -L../ffmpeg/lib -L../ogg/lib obj/src1.o obj/src2.o obj/main.o -o main
```

此外还有其他一些宏可以大大方便你的项目配置，详细见下面的说明。


# Documentation #

### src ###

  * 作用：声明项目里面的源文件（主文件可以不声明）。
  * 格式：//! src: file1.c, file2.c, file3.cpp, file4.cpp
  * 注意：如果不想再一行中写太多源文件，可以用多行 **//! src:**来表示

### inc ###

  * 作用：声明项目的include目录。
  * 格式：//! inc: ../include, d:/local/dxsdk/include, ../../include

### lib ###

  * 作用：声明项目的library目录。
  * 格式：//! lib: ../lib, ../../lib

### link ###

  * 作用：添加要连接的库
  * 格式：//! link: stdc++, m, pthread
  * 注意：如果连接内容是有扩展名的，如abc.o，那么将连接该文件；如果没有扩展名，比如 pthread，那么将视为一个libpthread.a库，使用 -lpthread来连接。

### mode ###

  * 作用：指定输出的模式。
  * 格式：//! mode: [exe|dll|lib|win]
  * **exe** 可执行模式
  * **dll** 动态链接库模式
  * **lib** 静态链接库模式
  * **win** windows程序（无console窗口）。

### out ###

  * 作用：指定连接后输出的文件名。
  * 格式：//! out: name

### int ###

  * 作用：指定临时文件路径，如.o文件以及.p文件。
  * 格式：//! int: path
  * 注意：默认临时文件放在当前文件夹，一般可以改为`//! int:obj`，让临时文件保存到 ./obj/目录下面去。

### flag ###

  * 作用：指明 gcc编译时候的其他参数。
  * 格式：//! flag: -Wall, -O3
  * 注意：像类似 -O, -pg, -g之类的编译参数可以如此传递过去。

### import ###

  * 作用：导入 emake.ini里面的某项配置
  * 格式：//! import: name
  * 注意：需要在emake.ini里面描述对应的编译配置，比如link, inc, lib, flag等。这样一次性import就可以省掉重复写很多inc, lib，比如 //! import: ffmpeg这样就一次性导入了 ffmpeg的编译环境了。

### export ###

  * 作用：导出模式（仅在输出动态链接库时使用）。
  * 格式：`//! export: msvc` 或者 `//! export:`
  * 注意：//! export: 以后将会导出动态库的 .a连接索引文件，而如果后面跟了msvc，则还可以同时导出vc的 .lib连接文件（前提是在emake.ini配置了vc环境）。

### 配置文件 ###

下面一个典型的配置文件：
```
[default]
home=/opt/mingw-x86/
flag=-Wall
link=stdc++, winmm, wsock32, opengl32, gdi32, glu32, ws2_32, user32

[local]
include=d:/dev/local/include
lib=d:/dev/local/lib

[ffmpeg]
include=d:/dev/local/opt/ffmpeg/include
lib=d:/dev/local/opt/ffmpeg/lib
link=avcodec, avdevice, avfilter, avformat, avutil, postproc, swscale
```

  * `[default]` 中的home指明了mingw的路径(可以通过 `$home/bin/gcc`定位到gcc的路径)，默认的home是 ./, /usr/bin, /usr/local/bin，如果是这三个之一，可以不用写 home。default的link, flag等其他参数同前面的说明。

  * `[local]` 定义了一个可以被 import的配置区，如果源程序内有 `//! import: local`那么 `local`下面的 include, lib两项配置就会被加入到项目中区。

  * `[ffmpeg]` 定义了另外一个可以被 import导入的配置区，如果源程序内有 `//! import: ffmpeg` 那么`ffmpeg`下面的 include, lib, link等定义就会被加到该项目中去。

该配置文件emake.ini可以放在 emake.py同一个目录，运行emake时将会被加载。而如果是在unix下面，如果存在 ~/.emake.ini，那它里面的内容也会同时被加载。

### MSVC环境配置 ###

emake.ini里面default和msvc两个区间名都是特殊区间，default用来定义全局公共环境，msvc用来定义vc的环境，如果打算导出带有.lib的动态库则需要定义下面内容，否则可以忽略：

```
[msvc]
VSINSTALLDIR=C:\Program Files (x86)\Microsoft Visual Studio 8
VCINSTALLDIR=%VSINSTALLDIR%\VC
FrameworkDir=C:\Windows\Microsoft.NET\Framework
INCLUDE=%VCINSTALLDIR%\ATLMFC\INCLUDE;%VCINSTALLDIR%\INCLUDE;%VCINSTALLDIR%\PlatformSDK\include;%VSINSTALLDIR%\SDK\v2.0\include;d:\dev\local\include;d:\dev\dxsdk\include
LIB=%VCINSTALLDIR%\ATLMFC\LIB;%VCINSTALLDIR%\LIB;%VCINSTALLDIR%\PlatformSDK\lib;%VSINSTALLDIR%\SDK\v2.0\lib;d:\dev\local\lib;d:\dev\dxsdk\lib\x86
LIBPATH=C:\Windows\Microsoft.NET\Framework\v2.0.50727;%VCINSTALLDIR%\ATLMFC\LIB;d:\dev\local\lib
PATH=%VSINSTALLDIR%\Common7\IDE;%VCINSTALLDIR%\BIN;%VSINSTALLDIR%\Common7\Tools;%VSINSTALLDIR%\Common7\Tools\bin;%VCINSTALLDIR%\PlatformSDK\bin;%VSINSTALLDIR%\SDK\v2.0\bin;C:\Windows\Microsoft.NET\Framework\v2.0.50727;%VCINSTALLDIR%\VCPackages;c:\Windows\system32
```

可以发现，里面都是环境变量的定义，这些环境定义可以从vcvars32.bat里面获得。也可以进入VC命令行打set命令看看该命令行定义了哪些和vc相关的环境变量，支持环境变量宏展开。