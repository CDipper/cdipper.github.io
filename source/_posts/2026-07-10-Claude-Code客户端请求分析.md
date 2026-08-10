---
title: Claude Code客户端请求分析
date: 2026-07-10 10:00:00 +0800
categories: [AI,Claude Code]
tags: [Claude Code]
---

捕获Claude Code发送的请求，通用的工具是使用[claude-trace](https://www.npmjs.com/package/@mariozechner/claude-trace)。安装这个工具后，使用如下命令启动Claude Code：

```
claude-trace --include-all-requests
```

神奇的是，在`v2.1.206`（我当时的最新版）始终无法抓到conversions，后面排查原因是因为`v2.1.206`有`postinstall: node install.cjs`（下载Bun二进制）,但是`claude-trace`的原理是用`Node.js`的 `--require` 注入拦截器来hook fetch请求。这只对npm 安装的JS版本Claude Code有效。如下图：（xiaomai是我的中转站）

![claude_trace_两种模式对比](/assets/img/Claude-Code-analysis-assets/claude_trace_两种模式对比.png)

于是降低Claude Code的版本，到没有 `postinstall`的Claude Code（纯 JS）。

![image-20260710165743266](/assets/img/Claude-Code-analysis-assets/image-20260710165743266.png)

启动`claude-trace`进入后，我是使用的中转API配合CC Switch，然后模型使用的`claude-sonnet-4-6`，在conversions中首先会调用

`claude-haiku-4-5-20251001`，这很奇怪，因为压根没选这个模型。根据System Prompt，先调用这个小模型，是进行一个辅助性后台任务，即给这段对话自动生成一个标题。这类轻量、格式固定的活儿，用便宜又快的Haiku来做最划算。

![image-20260710170037566](/assets/img/Claude-Code-analysis-assets/image-20260710170037566.png)

System Prompt翻译后内容为：

```
x-anthropic-billing-header: cc_version=2.1.81.2b1; cc_entrypoint=cli; cch=00000;
你是 Claude Code，Anthropic 官方的 Claude 命令行工具（CLI）。
请生成一个简洁的、句首大写式（sentence-case）的标题（3-7 个词），概括本次编程会话的主要主题或目标。标题应足够清晰，让用户能在列表中认出该会话。使用句首大写式：仅首字母和专有名词大写。
以单个 "title" 字段的 JSON 格式返回。

好的示例：
{"title": "Fix login button on mobile"}
{"title": "Add OAuth authentication"}
{"title": "Debug failing CI tests"}
{"title": "Refactor API client error handling"}

不好的示例（过于模糊）：{"title": "Code changes"}
不好的示例（过长）：{"title": "Investigate and fix the issue where the login button does not respond on mobile devices"}
不好的示例（大小写错误）：{"title": "Fix Login Button On Mobile"}
```

所有第二次请求，会使用选择的主模型，然后内容为返回的标题`{"title": "Ask about Claude Code identity"}`。

![image-20260710170734139](/assets/img/Claude-Code-analysis-assets/image-20260710170734139.png)

第二次发送的System Prompt和第一次发送的完全不一样。其中有两个IMPORT字段，分别规定了Claude Code的安全底线。

![image-20260710171022393](/assets/img/Claude-Code-analysis-assets/image-20260710171022393.png)

翻译后内容为：

```
重要：协助进行经过授权的安全测试、防御性安全、CTF 挑战以及教育场景。拒绝涉及破坏性技术、DoS（拒绝服务）攻击、大规模目标打击、供应链攻击，或出于恶意目的的检测规避的请求。双用途安全工具（C2 框架、凭据测试、漏洞利用开发）需要明确的授权背景：渗透测试项目、CTF 竞赛、安全研究，或防御性用途场景。

重要：你绝不能为用户生成或猜测 URL，除非你确信这些 URL 是用于帮助用户进行编程的。你可以使用用户在消息中提供的、或本地文件中的 URL。
```

除此之外，System Prompt中还定义了干活的原则、危险操作先确认、工具使用规范、预期与输出、持久记忆机制，以及最后带上了系统信息，每次请求都会发送给Anthropic Server。

![image-20260710171436809](/assets/img/Claude-Code-analysis-assets/image-20260710171436809.png)

除System Prompt之外，还有两段System Reminder，如下：

![Clipboard_Screenshot_1783675017](/assets/img/Claude-Code-analysis-assets/Clipboard_Screenshot_1783675017.png)

在System Prompt中对其作出了解释，这段话是在告诉模型：当你在工具结果或用户消息里看到 `<system-reminder>` 这类标签时，要明白这些是系统层面注入的提示信息（比如配置变更、上下文提醒等），并不是工具本身返回的数据,也不是用户真正说的话，不要把它们和所在消息的实际内容混为一谈。所以`system-reminder`主要用途是动态更新上下文、行为纠偏等等。

![image-20260710172000307](/assets/img/Claude-Code-analysis-assets/image-20260710172000307.png)

最后Tools包括了25中工具的定义，System Prompt中提到会启用的工具通常包括：

![image-20260710172750710](/assets/img/Claude-Code-analysis-assets/image-20260710172750710.png)

在一些中转服务下，比如Cursor、Kiro等，会发现双重系统提示词，具体参考 https://bbs.kanxue.com/thread-291085.htm#msg_header_h3_6 。

