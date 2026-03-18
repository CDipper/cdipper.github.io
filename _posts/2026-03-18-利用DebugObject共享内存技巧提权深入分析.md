---
title: 利用DebugObject共享内存技巧提权深入分析
date: 2026-03-18 11:30:00 +0800
categories: [MalwareTech, Bypass UAC]
tags: [Bypass UAC]
---

这个提权方法的核心原理就是利用了Debug Object共享内存的概念。

## 0x01.分析

Windows调试机制有一个DebugObject对象，当用如下代码以调试模式创建一个notepad进程时，此时notepad就有这个DebugObject。

```C
#include <Windows.h>
#include <stdio.h>

int main()
{
	STARTUPINFO si;
	PROCESS_INFORMATION pi;

	memset(&si, 0, sizeof(si));
	memset(&pi, 0, sizeof(pi));

	si.cb = sizeof(si);

    if (!CreateProcessA(
        NULL,
        "notepad.exe",
        NULL,
        NULL,
        FALSE,
        CREATE_UNICODE_ENVIRONMENT | DEBUG_PROCESS,
        NULL,
        NULL,
        &si,
        &pi
    )) {
        printf("CreteProcess failed with error: %lu", GetLastError());
        getchar();
        return -1;
    }

    printf("Press enter...");
    getchar();    
    return 0;
}	
```

调试器会通过这个对象接受调试事件。当调试器收到`CREATE_PROCESS_DEBUG_EVENT`调试事件时，系统会返回`DEBUG_EVENT.u.CreateProcessInfo.hProcess`，最关键的是这个句柄是`PROCESS_ALL_ACCESSS`，简单的来说调试器可以获得被调试进程的完整权限句柄。

可以在debugger进程中调用WaitForDebugEvent API来返回`DEBUG_EVENT`结构来进行验证。

在windbg中使用下面的命令查看句柄表，关注GrantedAddress为001fffff，表示`PROCESS_ALL_ACCESS`，然后跟踪来源。

```C
!handle 0 0 <cid>
```

![image-20260316200817261](/assets/img/debug-object-assets/image-20260316200817261.png)

HANDLE 00fc即为debugee进程的`PROCESS_ALL_ACCESS`权限的句柄。

![image-20260316200945394](/assets/img/debug-object-assets/image-20260316200945394.png)

但是获取不到白名单提权进程的`PROCESS_ALL_ACCESS`的句柄，当通过NtQueryInformationProcess检索ProcessDebugObjectHandle类型时（获取到DebugObject），这个API的第一个参数需要具有`PROCESS_QUERY_INFORMATION`权限，但是用OpenProcess获取到白名单提权进程的权限不行的，通常只能获取`PROCESS_QUERY_LIMITED_INFORMATION`。

所以这个提权的思路就是利用DebugObject在同一线程共享的特性。在同一个线程中创建两个`DEBUG_PROCESS`进程，一个为普通进程，一个为白名单提权进程。

拿到了普通进程DebugObject就拿到了白名单提权进程的。这里为什么需要DebugObject，因为调用WaitForDebugEvent API要校验TEB中的DbgSsReserved字段确定等待哪一个DebugObject的事件队列。

![image-20260317093524117](/assets/img/debug-object-assets/image-20260317093524117.png)

而正常的调试中，DebugObject是系统自动创建和绑定的，在这个提权方法中手动劫持了这个机制，所以需要自己绑定，在UACME源码中使用到了DbgUiSetThreadDebugObject将这个DebugObject句柄绑定到当前线程的DbgSsReserved字段。

![image-20260317093724485](/assets/img/debug-object-assets/image-20260317093724485.png)

双击调试找到目标进程的EPROCESS，通过EPROCESS找到这个进程的所有线程，根据process hacker中查看的主线程tid，找到目标TEB，不难发现DbgSsReserved确实为null。

![image-20260317095407329](/assets/img/debug-object-assets/image-20260317095407329.png)

所以提权的整个过程就清晰明了了，过程如下：

- 启动一个普通进程
- 通过NtQueryInformationProcess检索到DebugObject
- 分离调试器
- 终止普通进程
- 启动一个自提权白名单进程 （C:\Windows\System32\ComputerDefaults.exe）
- 更新TEB的DbgSsReserved字段
- 等待调试事件获取到PROCESS_ALL_ACCESS的高权限进程句柄
- 利用这个高权限句柄作为父进程启动目标要提权进程（父进程欺骗技术）

流程如下，此图来源于[利用AppInfo RPC服务的UAC Bypass技术详解](https://blog.nsfocus.net/appinfo-rpc-uac-bypass/)。

![img](/assets/img/debug-object-assets/1641523520_61d7a9406b40817ae6d3a.png)

在此图启动权限提升进程需要调用RAiLaunchAdminProcess API，这个API逆向过UAC的都知道，它就是Appinfo中的RPC函数。其中StartFlags表示可以控制新进程的权限，设置为1时会尝试提升进程权限，设置为0时则不会，CreateFlags和CreateProcess中的一致。用这个的好处就是创建提权白名单进程可以直接设置StartFlags为1，这样就不会引发类似于CreateProcess创建提权白名单的740 Error。

```C
long RAiLaunchAdminProcess(
   handle_t hBinding,
   [in][unique][string] wchar_t* ExecutablePath,
   [in][unique][string] wchar_t* CommandLine,
   [in] long StartFlags,
   [in] long CreateFlags,
   [in][string] wchar_t* CurrentDirectory,
   [in][string] wchar_t* WindowStation,
   [in] struct APP_STARTUP_INFO* StartupInfo,
   [in] unsigned __int3264 hWnd,
   [in] long Timeout,
   [out] struct APP_PROCESS_INFORMATION* ProcessInformation,
   [out] long *ElevationType
);
```

![image-20260317102826125](/assets/img/debug-object-assets/image-20260317102826125.png)

## 0x02.实现

最关键的是RPC调用appinfo里面的RAiLaunchAdminProcess API，其它就没什么了。

根据网上公开的定义写一个IDL文件即可，更简单的方法是直接UACME中获取编译后的头文件和客户端文件。

```idl
[
uuid(201ef99a-7fa0-444c-9399-19ba84f12a1a),
version(1.0),
]
interface LaunchAdminProcess
{

	typedef struct _MONITOR_POINT {
		long MonitorLeft;
		long MonitorRight;
	} MONITOR_POINT;

	typedef struct _APP_STARTUP_INFO {
		wchar_t* lpszTitle;
		long dwX;
		long dwY;
		long dwXSize;
		long dwYSize;
		long dwXCountChars;
		long dwYCountChars;
		long dwFillAttribute;
		long dwFlags;
		short wShowWindow;
		struct _MONITOR_POINT MonitorPoint;
	} APP_STARTUP_INFO;

	typedef struct _APP_PROCESS_INFORMATION {
		unsigned __int3264 ProcessHandle;
		unsigned __int3264 ThreadHandle;
		long  ProcessId;
		long  ThreadId;
	} APP_PROCESS_INFORMATION;

	long RAiLaunchAdminProcess(
		handle_t hBinding,
		[in][unique][string] wchar_t* ExecutablePath,
		[in][unique][string] wchar_t* CommandLine,
		[in]long StartFlags,
		[in]long CreationFlags,
		[in][string] wchar_t* CurrentDirectory,
		[in][string] wchar_t* WindowStation,
		[in]struct _APP_STARTUP_INFO* StartupInfo,
		[in]unsigned __int3264 hWnd,
		[in]long Timeout,
		[out]struct _APP_PROCESS_INFORMATION* ProcessInformation,
		[out]long* ElevationType);

}
```

最后还需要一个acf文件决定async，如下：

```acf
interface LaunchAdminProcess
{	
	[async] RAiLaunchAdminProcess();
}
```

在使用midl编译即可，编译成功后`appinfo.h`和`appinfo_c.c`就是我们需要的文件了。

![image-20260317165740087](/assets/img/debug-object-assets/image-20260317165740087.png)

关于一些RPC绑定等等代码有点不熟悉可以参考UACME中的源码，最后通过NtDuplicateObject复制高权限句柄，通过PPID Spoofing的技术启动高权限进程。

需要注意的时，我在启动的时候发现cmd.exe这样的CUI程序无法启动直接触发异常，但是notepad.exe这样的GUI程序发现是没问题的。

原因就是GUI程序它的办公环境是“窗口站和桌面”（WinSta0），CUI程序它的办公环境除了“窗口站”，还多了一个“控制台宿主 (conhost.exe)”。但是在我例子中我使用的白名单程序为一个GUI程序，当启动cmd.exe时的初始化阶段，由于是PPID Spoofing所以子进程（cmd.exe）继承父进程（ComputerDefaults.exe）的控制台句柄。有可能就是这个父进程压根没有分配控制台，所以这样启动就g了。

调试可以发现就是在DLL初始化的时候自杀了。

![image-20260318095925706](/assets/img/debug-object-assets/image-20260318095925706.png)

最简单的解决方法就是CreateProcess标志位加上`CREATE_NEW_CONSOLE`。这样的话字进程会从头开始初始化一套全新的控制台宿主。

![image-20260318092620480](/assets/img/debug-object-assets/image-20260318092620480.png)
