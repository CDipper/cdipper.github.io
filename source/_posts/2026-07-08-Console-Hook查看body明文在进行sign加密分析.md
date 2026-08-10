---
title: Console Hook查看body明文在进行sign加密分析
date: 2026-07-09 10:00:00 +0800
categories: [Web Security,javascript逆向]
tags: [javascript逆向]
---

最近我在github上看到一个针对javascript逆向的MCP工具：  https://github.com/zhizhuodemao/js-reverse-mcp 。

然后我随便找了个网站进行简单逆向，首先我是确定在此站点的register接口会有sign，然后丢给Claude Code分析（配合上面MCP工具），发现其http body是使用非对称加密的，我之前遇到的情况都是http body部分（可能是一个json）使用对称加密的，这种情况下的渗透测试显得比较轻松。我通常的做法是，采取Yakit输入明文，下游代理到MITM脚本读取明文进行加密，然后MITM在配置上游代理，再次发送到Yakit进行Web Fuzzer。对于今天到这种情况并不太适用，因为压根不知道body部分到数据有什么？sign生成也需要依赖body的明文数据，是拿不到私钥的。

![image-20260709165400082](/assets/img/js-reverse-assets/image-20260709165400082.png)

![image-20260709203846050](/assets/img/js-reverse-assets/image-20260709203846050.png)

遇到这种情况最好的解决方法是，在DevTools Console中Hook明文。核心就是用一个自己的函数，把原来的函数"包起来"，在它真正执行前后插进去偷看参数。 这在JS里叫猴子补丁（Monkey Patching），也就是函数劫持。

如下是针对我测试站点的hook脚本：

```
(function(){
  var oEnc = JSEncrypt.prototype.encryptLong;
  JSEncrypt.prototype.encryptLong = function(str){
    console.log("%c[body]", "color:#0a0", str);
    return oEnc.apply(this, arguments);
  };
  console.log("hook injected!");
})();
```

`JSEncrypt.prototype.encryptLon`是明文传输路径上的一个函数（这个需要具体分析，可以叫Claude Code配合上面那个MCP分析一下），这里选择这个作为hook点。整个hook代码的意思是先保存原函数引用，然后将原函数覆盖为偷看body明文的新函数，console打印完成后，再调用原函数，保证原功能不变。

![image-20260709165454127](/assets/img/js-reverse-assets/image-20260709165454127.png)

再次测试此接口时，可以按照此json格式进行测试。现在就可以叫Claude Code写一个MITM脚本，读取body明文数据在进行加密，然后重新生成sign，在配置上游代理到Yakit，完成Web Fuzzer。这里不太好操作（后面会说）。

如下先在Yakit中抓到包，修改body为明文（这个明文结构是console中hook输出修改后的）。

![image-20260709203916813](/assets/img/js-reverse-assets/image-20260709203916813.png)

然后启动MITM脚本并配置上游代理（配置这个上游代理的目的是为了修改后的请求再次导入Yakit，在Yakit操作可以更好的看到响应）：

```
mitmdump -p 9080 --ssl-insecure --mode upstream:http://127.0.0.1:9999 -s mitm_gateway.py
```

![image-20260709194329426](/assets/img/js-reverse-assets/image-20260709194329426.png)

Yakit其实一点也不好配合mitmproxy，mitmproxy不配上游代理还好，如果配置了并且Yakit开启了下游代理（即代理到mitmproxy），是抓不到包的，会一直循环，卡爆。

![Clipboard_Screenshot_1783597636](/assets/img/js-reverse-assets/Clipboard_Screenshot_1783597636.png)

我抓包的方法是先在Yakit关闭下游代理，然后将请求包抓到，再开启下游代理，修改数据包后放行。这样在渗透测试过程中，非常麻烦。

那么能不能再启一个Yakit，监听另一个端口呢？这样就不会循环了，但是Yakit的Electron层有单实例锁（single-instance lock），双击第二次只会聚焦到已开的窗口，不会真开第二个。要简单的解决的话，可以在开一个BurpSuite抓MTIM脚本发的包即可解决（doge）。

但在MacOS上，`open -n`会强制新建一个进程实例，绕过"聚焦已有窗口"的行为：

```
open -n /Applications/Yakit.app
```

![image-20260709201506008](/assets/img/js-reverse-assets/image-20260709201506008.png)

这虽然能弹出第二个GUI窗口，但是会去连同一个yak引擎（grpc 同端口）+ 同一个`yakit-projects`项目库，于是：

- 要么提示”引擎已被占用/连接冲突“

- 要么两个GUI抢同一个项目SQLite库 ，导致数据错乱

但是Yakit启动时可以换引擎端口，第二个启动换一个即可，并且一个Yakit启动可以使用临时项目模式，另一个启动使用default就不用担心SQLite数据库撞的问题，而且还可以新建一个项目嘛。所以启第二个Yakit程序是完全行得通的。

需要注意的是监听端口不能重复，

![image-20260709202340127](/assets/img/js-reverse-assets/image-20260709202340127.png)

然后MITM脚本可以上游代理到第二个新启动的Yakit，并且第二个Yakit千万不要在配置下游代理了。整个链路就是一个单线了，修改请求包在第一个Yakit，看响应在第二个Yakit。

![image-20260709203640599](/assets/img/js-reverse-assets/image-20260709203640599.png)



