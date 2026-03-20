---
title: Inject-Helper分析
date: 2026-03-20 14:30:00 +0800
categories: [MalwareTech, 进程注入]
tags: [进程注入]
---

inject-helper其实就是一个很多程序都会带的一个程序，听名字就感觉是一个LOLBins，如下图是我在OBS-Studio安装目录中看到的。故进行一下分析以及How to use。

![image-20260320113608311](/assets/img/inject-helper-assets/image-20260320113608311.png)

##  0x01.分析

这个东西竟然在github上有[源码](https://github.com/obsproject/obs-studio/blob/master/plugins/win-capture/inject-helper/inject-helper.c)，用IDA对比看了一下，应该是完全相同的吧。首先是main函数中，将argv和argv[1]传到了`inject_helper`函数中。

![image-20260320113626860](/assets/img/inject-helper-assets/image-20260320113626860.png)

此函数根据我们传入的的第二个参数`use_safe_inject`决定采取哪一种注入方法，所以第一个参数就是`dllpath`，第三个参数是`thread_id`。

![image-20260320114053067](/assets/img/inject-helper-assets/image-20260320114053067.png)

首先看当`use_safe_inject == 1`时的注入方法，进入到另一个方法中。

![image-20260320114310529](/assets/img/inject-helper-assets/image-20260320114310529.png)

在此方法中利用SetWindowsHookEx设置`WH_GETMESSAGE`消息hook，最后调用PostThreadMessage触发执行回调。

![image-20260320115002403](/assets/img/inject-helper-assets/image-20260320115002403.png)

DLL需要导出的函数为`dummy_debug_proc`。

![image-20260320115433319](/assets/img/inject-helper-assets/image-20260320115433319.png)

当`use_safe_inject == 0`时，就是一个很经典的注入方式，这里的第三个参数即为`process_id`了。

![image-20260320143133880](/assets/img/inject-helper-assets/image-20260320143133880.png)

## 0x02.测试

先写一个测试DLL：

```
#include <Windows.h>

extern "C" __declspec(dllexport) void dummy_debug_proc() {
    MessageBoxA(NULL, "dummy_debug_proc", "Caption", NULL);
}

BOOL WINAPI DllMain(
    HINSTANCE hinstDLL,  // handle to DLL module
    DWORD fdwReason,     // reason for calling function
    LPVOID lpvReserved)  // reserved
{
    // Perform actions based on the reason for calling.
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        // Initialize once for each new process.
        // Return FALSE to fail DLL load.
        MessageBoxA(NULL, "DllMain", "Caption", NULL);
        break;

    case DLL_THREAD_ATTACH:
        // Do thread-specific initialization.
        break;

    case DLL_THREAD_DETACH:
        // Do thread-specific cleanup.
        break;

    case DLL_PROCESS_DETACH:

        if (lpvReserved != nullptr)
        {
            break; // do not do cleanup if process termination scenario
        }

        // Perform any necessary cleanup.
        break;
    }
    return TRUE;  // Successful DLL_PROCESS_ATTACH.
}
```

`use_safe_inject == 1`时，针对x64使用方法如下：

```
inject-helper64.exe <dllpath> 1 <tid>
```

这个导出函数（回调）会执行多次。

![image-20260320145504945](/assets/img/inject-helper-assets/image-20260320145504945.png)

`use_safe_inject == 0`时，针对x64使用方法如下：

```
inject-helper64.exe <dllpath> 0 <pid>
```

![image-20260320145731195](/assets/img/inject-helper-assets/image-20260320145731195.png)