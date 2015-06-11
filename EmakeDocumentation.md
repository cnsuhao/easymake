# Description #

EMake is a easy tool which controls the generation of executables and other non-source files of a program from the program's source files.

Traditional GNU Make tool requires a Makefile to describe how to compile the whole project, which is too powerful to everyday usage. In my experience, there are not many large projects which need to write a Makefiles, but many small demos or test projects to prove my thoughts or experiment something.

Getting tired of writing (or editing) a Makefile for each project whose LOC is below 1k, I am looking for something as easy as Visual Studio: add source files, and build it.

# Features #
**EMake gets its knowledge of how to build your program from the comment macro of the main source file**. The comment macro describes how to compile the whole project. You can use "//!" as the Emake macro in your source file, such like you have this line in main.c:
```
//! exe: file1.c, file2.c
```

when you call **"emake.py main.c"**, Emake will know this project has three files: file1.c, file2.c and main.c itselfs. and the output mode is "exe" (it can be **"exe/dll/clib/win"**). Then emake will parse all the source files in the project.

In the second step, Emake will indicate what header files are used in each source file, and check if the source need to be compiled. (the timestamps of the source file and its header files is newer than the its obj file).

At last, Emake will call gcc to compile the source files in the project, and then link them to generate the executable file (or dynamic/static library).

Emake gets its knowledge directly from the source file comments, which avoid writing another Makefile. You can describe project source files, compiler flags, include/library directories, linking mode, ... etc directly in your C files with a few comment macro.

Providing a completely **new way to build projects**, emake is very suitable for small or medium projects, You can describe your project much simply than traditional GNU make, then detect dependences, at last build it. It is easy to make MAKE easy.

# Installation #

EMake is written with python in a single .py file. You can download the tarball and extract emake.py into current folder. Note, emake only supports gcc right now, so if you are under windows, you must download mingw to work with.

### Install EMake in Unix ###
use **"./emake.py -install"** to install. The installation will copy itself into /usr/local/bin. It will search gcc in /usr/bin or /usr/local/bin.

### Install EMake in Windows with mingw ###
copy emake.py into the root directory of mingw, in order to find gcc.exe in ".\bin\" directory. then you can execute as "c:\mingw\emake.py"



# Documentation #

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

  * The comment macro **"//!"** tells emake this is an EMake Macro,
  * **mode** tells emake the output format is an executable file (It can be 'exe', 'dll', 'lib').
  * **out** tells emake the output file's name
  * **src** tells emake there are two source files(testmain.c and testsrc1.c)
  * **flag** tells emake to send "-O3" to gcc parameters in compiling
  * **link** tells emake to send "-lm" to gcc parameters in linking


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

Then you can use emake to build this project:
```
% python emake.py testmain.c
compiling ...
testmain.c
testsrc1.c
linking ...
```

Then you can run your executable file:
```
% ./testmain
This is foo
```

Actually EMake has done these things below for you:
```
gcc -O3 -c testmain.c -o testmain.o
gcc -O3 -c testsrc1.c -o testsrc1.o
gcc testmain.o testsrc1.o -o testmain -lm
```

Every time you call "emake.py testmain.c", it will check the timestamp of testmain.c and testsrc1.h (included by testmain.c) and timestamp of testmain.o, If they are earlier than testmain.o emake will skip compiling testmain.c.

It is that emake will parse the source file and find what header files it included and what header files the header files included, and so on, at last emake will find all the header files involved by the source file. Emake can check whether to compile the source by the timestamp of these files and the timestamp of the obj file.


### Intermediate Files Path ###
If you add a emake macro with **"//! int: PATH"** in testmain.c, emake will output obj files
and other intermediate files into PATH，eg:
```
//! mode: exe
//! int: obj
//! out: testmain
//! src: testmain.c
//! src: testsrc1.c
```

emake will do these thing below for you:
```
mkdir obj
gcc -c testmain.c -o obj/testmain.o
gcc -c testsrc1.c -o obj/testsrc1.o
gcc obj/testmain.o obj/testsrc1.o -o testmain
```

Note: **"src"** keywords can be used many times, all the files followed by each **"src"** will be involved in the project.

### Easier Instructions ###

There is a main.c which contains the following comment lines as emake macro:

```
//! exe: source1.c, source2.c
```

It is same as these three instructions:
```
//! mode: exe
//! int: obj
//! src: source1.c, source2.c
```

If you used the keywords **exe/dll/clib/win** and followed by a file list (the list can be nothing '//!exe :\n'), emake will do the three things:

  * set intermediate file folder to "obj"
  * set output mode to exe/dll/clib/win (**note: use clib to avoid conflicting with another keyword lib**)
  * set the source files to the project file list.

and then do the following jobs for you:

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

EMake will do these jobs below for you:

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

EMake will do these jobs below for you:
```
mkdir obj
gcc -I../ffmpeg/include -I../ogg/include -c src1.c -o obj/src1.o
gcc -I../ffmpeg/include -I../ogg/include -c src2.c -o obj/src2.o
gcc -I../ffmpeg/include -I../ogg/include -c main.c -o obj/main.o
gcc -L../ffmpeg/lib -L../ogg/lib obj/src1.o obj/src2.o obj/main.o -o main
```