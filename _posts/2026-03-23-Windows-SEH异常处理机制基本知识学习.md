---
title: Windows SEH异常处理机制基本知识学习
date: 2026-03-23 12:30:00 +0800
categories: [Windows机制, SEH]
tags: [SEH]
---

SEH (Structured Exception Handling, 结构化异常处理）是 Windows 操作系统用千自身除错的一种机制。SEH是一种错误保护和修复机制，它告诉系统当程序运行出现异常或错误时由谁来处理，给了应用程序一个改正错误的机会。从程序设计的角度来说，就是系统在终结程序之前给程序提供的一个执行其预先设定的回调函数的机会。

tips：以下讨论的点仅针对x86，在x64上的异常处理机制很不一样，了解堆栈欺骗技术的话，知道`.pdata`中`UNWIND_INFO`，handler就是编译时写入的，而不是运行时注册的。

## 0x01.SEH的相关数据结构

TIB (Thread Information Block, 线程信息块）是保存线程基本信息的数据结构。在用户模式下， 它位于TEB的头部，而 TEB 是操作系统为了保存每个线程的私有数据创建的，每个线程都有自己的TEB。TIB结构如下：

```
//0x1c bytes (sizeof)
struct _NT_TIB
{
    struct _EXCEPTION_REGISTRATION_RECORD* ExceptionList;                   //0x0
    VOID* StackBase;                                                        //0x4
    VOID* StackLimit;                                                       //0x8
    VOID* SubSystemTib;                                                     //0xc
    union
    {
        VOID* FiberData;                                                    //0x10
        ULONG Version;                                                      //0x10
    };
    VOID* ArbitraryUserPointer;                                             //0x14
    struct _NT_TIB* Self;                                                   //0x18
}; 
```

第一个字段其实是指向`_EXCEPTION_REGISTRATION_RECORD`的指针，显然也位于TEB偏移0处，结构如下：

```
//0x8 bytes (sizeof)
struct _EXCEPTION_REGISTRATION_RECORD
{
    struct _EXCEPTION_REGISTRATION_RECORD* Next;                            //0x0
    enum _EXCEPTION_DISPOSITION (*Handler)(struct _EXCEPTION_RECORD* arg1, VOID* arg2, struct _CONTEXT* arg3, VOID* arg4); //0x4
}; 
```

第一个字段指向了一下个`_EXCEPTION_REGISTRATION_RECORD`结构（简称ERR指针），形成一个单链表，链头就放在`fs:[0]`处，第二个字段Handler指向一个回调函数。当程序执行发生异常时，系统的异常分发器就会从`fs:[0]`拿到异常处理的链头，查找这个单链表，依次调用每个节点的回调函数。显然每个线程都有自己的这个单链表，即SEH是作用于线程的。

从数据结构的角度来说，SEH链就是一个只允许在链头进行进行增加和删除节点的单向链表，且链头始终在`fs:[0]`处。

当发生异常时，CPU就会通过IDT线性表处理该异常，调用的异常处理函数除了针对本异常的特定处理之外，通常会将异常信息进行封装，以便进行后续处理。

![image-20260323195605588](/assets/img/seh-basis-assets/image-20260323195605588.png)

封装的内容主要有两部分。 一部分是异常记录， 包含本次异常的信息，该结构定义如下：

```
//0x50 bytes (sizeof)
struct _EXCEPTION_RECORD
{
    LONG ExceptionCode;                                                     //0x0
    ULONG ExceptionFlags;                                                   //0x4
    struct _EXCEPTION_RECORD* ExceptionRecord;                              //0x8
    VOID* ExceptionAddress;                                                 //0xc
    ULONG NumberParameters;                                                 //0x10
    ULONG ExceptionInformation[15];                                         //0x14
}; 
```

ExceptionCode为异常产生的原因，对应参考 [EXCEPTION_RECORD (winnt.h) - Win32 apps | Microsoft Learn ](https://learn.microsoft.com/zh-cn/windows/win32/api/winnt/ns-winnt-exception_record)。

![image-20260323195922669](/assets/img/seh-basis-assets/image-20260323195922669.png)

另一部分被封装的内容称为陷阱帧（`_KTRAP_FRAME`），它精确描述了发生异常时线程的状态 (Windows的任务调度是基千线程的）。该结构与处理器高度相关，因此在不同的平台上 (Intel x86/x64、 MIPS、 Alpha和PowerPC处理器等）有不同的定义。在x64下有如下定义：

```
//0x8c bytes (sizeof)
struct _KTRAP_FRAME
{
    ULONG DbgEbp;                                                           //0x0
    ULONG DbgEip;                                                           //0x4
    ULONG DbgArgMark;                                                       //0x8
    USHORT TempSegCs;                                                       //0xc
    UCHAR Logging;                                                          //0xe
    UCHAR FrameType;                                                        //0xf
    ULONG TempEsp;                                                          //0x10
    ULONG Dr0;                                                              //0x14
    ULONG Dr1;                                                              //0x18
    ULONG Dr2;                                                              //0x1c
    ULONG Dr3;                                                              //0x20
    ULONG Dr6;                                                              //0x24
    ULONG Dr7;                                                              //0x28
    ULONG SegGs;                                                            //0x2c
    ULONG SegEs;                                                            //0x30
    ULONG SegDs;                                                            //0x34
    ULONG Edx;                                                              //0x38
    ULONG Ecx;                                                              //0x3c
    ULONG Eax;                                                              //0x40
    UCHAR PreviousPreviousMode;                                             //0x44
    UCHAR EntropyQueueDpc;                                                  //0x45
    union
    {
        UCHAR NmiMsrIbrs;                                                   //0x46
        UCHAR Reserved1;                                                    //0x46
    };
    UCHAR PreviousIrql;                                                     //0x47
    ULONG MxCsr;                                                            //0x48
    struct _EXCEPTION_REGISTRATION_RECORD* ExceptionList;                   //0x4c
    ULONG SegFs;                                                            //0x50
    ULONG Edi;                                                              //0x54
    ULONG Esi;                                                              //0x58
    ULONG Ebx;                                                              //0x5c
    ULONG Ebp;                                                              //0x60
    ULONG ErrCode;                                                          //0x64
    ULONG Eip;                                                              //0x68
    ULONG SegCs;                                                            //0x6c
    ULONG EFlags;                                                           //0x70
    ULONG HardwareEsp;                                                      //0x74
    ULONG HardwareSegSs;                                                    //0x78
    ULONG V86Es;                                                            //0x7c
    ULONG V86Ds;                                                            //0x80
    ULONG V86Fs;                                                            //0x84
    ULONG V86Gs;                                                            //0x88
}; 
```

上述结构中包含每个寄存器的状态，但该结构一般仅供系统内核自身或者调试系统使用。当需要把控制权交给用户注册的异常处理程序时，会将上述结构，换成一个名为`CONTEXT`的结构，定义如下：

```
//0x2cc bytes (sizeof)
struct _CONTEXT
{
    ULONG ContextFlags;                                                     //0x0
    ULONG Dr0;                                                              //0x4
    ULONG Dr1;                                                              //0x8
    ULONG Dr2;                                                              //0xc
    ULONG Dr3;                                                              //0x10
    ULONG Dr6;                                                              //0x14
    ULONG Dr7;                                                              //0x18
    struct _FLOATING_SAVE_AREA FloatSave;                                   //0x1c
    ULONG SegGs;                                                            //0x8c
    ULONG SegFs;                                                            //0x90
    ULONG SegEs;                                                            //0x94
    ULONG SegDs;                                                            //0x98
    ULONG Edi;                                                              //0x9c
    ULONG Esi;                                                              //0xa0
    ULONG Ebx;                                                              //0xa4
    ULONG Edx;                                                              //0xa8
    ULONG Ecx;                                                              //0xac
    ULONG Eax;                                                              //0xb0
    ULONG Ebp;                                                              //0xb4
    ULONG Eip;                                                              //0xb8
    ULONG SegCs;                                                            //0xbc
    ULONG EFlags;                                                           //0xc0
    ULONG Esp;                                                              //0xc4
    ULONG SegSs;                                                            //0xc8
    UCHAR ExtendedRegisters[512];                                           //0xcc
}; 
```

其余字段没啥注意的，ContextFlag表示该结构中的哪些域有效，当需要用`CONTEXT`结构保存的信息恢复执行时可对应更新，这为有选择地更新部分域而非全部域提供了有效的手段。

![image-20260323200719959](/assets/img/seh-basis-assets/image-20260323200719959.png)

封装完毕后，异常处理函数会进一步调用系统内核的`nt!KiDispatchException`函数来处理异常，该函数定义如下（参考ReactOS）：

```
VOID NTAPI KiDispatchException (
IN PEXCEPTION_RECORD ExceptionRecord,         // 异常记录
IN PKEXCEPTION_FRAME ExceptionFrame,          //  对 NT386 系统总是为 NULL 未使用           
IN PKTRAP_FRAME TrapFrame,                    // 陷阱帧
IN KPROCESSOR_MODE PreviousMode,              // 表示发生异常时是什么模式 kernelmode or usermode
IN BOOLEAN FirstChance                        // 是否第一次处理该异常
)
```

总结一下就是，程序运行发生异常时，通过IDT表处理，首先对两部分内容（陷阱帧、异常记录）进行封装，然后进一步调用KiDispatchException进行处理。

当一个异常发生时处理顺序首位通常时调试器，当不存在调试器时首位便是SEH。由于应用层和内核层使用的是两个不同的栈，所以为了让用户态的异常处理程序能够处理访问异常相关的结构，所以内核会将这次异常相关联的ExceptionRecord和CONTEXT放到用户态栈中，同时在栈放置一个`_EXCEPTION_POINTERS`结构，它包含两个结构，一个是指向`_EXCEPTION_RECORD`的指针，另一个则是指向CONTEXT的指针。这样能确保用户态的异常处理程序就能够取得异常的具体信息和发生异常时线程的状态信息，并根据具体情况进行处理了。

```
//0x10 bytes (sizeof)
struct _EXCEPTION_POINTERS
{
    struct _EXCEPTION_RECORD* ExceptionRecord;                              //0x0
    struct _CONTEXT* ContextRecord;                                         //0x8
}; 
```

## 0x02.SEH 处理程序的安装和卸载

要注册SEH，只需要填写一个新的`_EXCEPTION_REGISTRATION_RECORD`结构，并将其放到链表头即可。根据SEH的设计要求，它的作用范围与安装它的函数相同，所以通常在函数头部安装SEH异常处理程序，在函数返回前卸载。可以说， SEH是基于栈帧的异常处理机制。

在安装SEH处理程序之前，需要准备一个符合SEH标准的回调函数，然后使用如下代码进行SEH异常处理程序的安装。

```
push offset SEHandler  
push fs:[0]
mov fs:[0],esp
```

x86的话首先向栈里面压了两个参数，即`_EXCEPTION_REGISTRATION_RECORD`的第一个和第二个参数，此时esp刚好指向这个结构开头，所以后面mov直接放到链表头。这就是最简单的安装过程（直观的感受），用C代码则是如下：

```
__try
{
    crash();
}
__except(EXCEPTION_EXECUTE_HANDLER)
{
    printf("error");
}
```

本来想反编译看一下是不是和上面的汇编代码一致的，但是Visual Studio 2022默认采用SAFESEH了，注册的结构都不一样了。先将这个选项关闭。

![image-20260323205756953](/assets/img/seh-basis-assets/image-20260323205756953.png)

此时在用IDA看就一致了。

![image-20260323205931054](/assets/img/seh-basis-assets/image-20260323205931054.png)

卸载也很简单了，从链表头部删除一个节点即可，如下：

```
mov esp,dword ptr fs:[O] 
pop dword ptr fs:[O] 
```

![image-20260324092525425](/assets/img/seh-basis-assets/image-20260324092525425.png)

## 0x03.SEH实例跟踪

调试如下代码：

```
#include <windows.h>
#include <stdio.h>

int main()
{
    __try
    {
        int* p = NULL;
        *p = 1;
    }
    __except (EXCEPTION_EXECUTE_HANDLER)
    {
        printf("error");
    }
}
```

在Windbg显然能够看到`0xC0000005`的错误，代表内存访问异常。异常首次会交给调试器（first chance）。

![image-20260324093333299](/assets/img/seh-basis-assets/image-20260324093333299.png)

然后跟踪系统对异常的处理过程，将断点下到异常处理函数`KiUserExceptionDispatcher`（用户态） ，在Windbg中输入`gn`命令，表示不处理该异常继续往下执行。异常会被交给用户态的`KiUserExceptionDispatcher`函数进行进一步分发，所以继续执行会断在这个函数，如下：

![image-20260324093954504](/assets/img/seh-basis-assets/image-20260324093954504.png)

此时栈顶是`_EXCEPTION_POINTERS`结构，`_EXCEPTION_RECORD`以及`CONTEXT`。0x0116f35c表示ExceptionRecord，0x0116f3ac表示ContextRecord。可以发现esp此时指向的时`_EXCEPTION_POINTERS`，且大小位8字节，`ExceptionRecord == esp + 8`，所以`_EXCEPTION_POINTERS` 结构之后紧跟`_EXCEPTION_ RECORD`。

![image-20260324101426530](/assets/img/seh-basis-assets/image-20260324101426530.png)

可以很明显的看到异常代码以及异常发生位置。`EceptionInformation`大小不确定，是由前一个字段`NumberParameters`决定的，在当前例子中， `_EXCEPTION_RECORD`的实际大小如下：

- 0x14 + 2 * sizeof(ULONG) = 0x1c
- ExceptionRecord + 0x1c = 0x0116f35c + 0x1c = 0x0116f378 (ContextRecord的位置）

但是不知道是不是什么编译的问题，调试发现我的ExceptionInformation就是一个有15个元素的数组，重新计算大小才可以准确定位ContextRecord：

- 0x14 + 15 * sizeof(ULONG) = 0x50
- ExceptionRecord + 0x50 = 0x0116f35c + 0x50 = 0x0116f3ac (ContextRecord的位置）

CONTEXT结构如下：

![image-20260324103046191](/assets/img/seh-basis-assets/image-20260324103046191.png)

可以清楚的看到寄存器对应的值，和发生异常时的情况完全一致。

