---
title: 'xss学习笔记'
published: 2026-04-19
description: 'xss_labs Less1-20 wp'
tags: [Security]
category: note
draft: false
---

# XSS（跨站脚本攻击）

攻击者把恶意 JS 脚本注入到网页里，浏览器分不清脚本是合法的，就会执行这段恶意代码。

分三类：

1. **存储型 XSS**：恶意代码存到服务器数据库（评论、留言），别人访问页面就触发，危害最大。
2. **反射型 XSS**：恶意脚本放在 URL 参数里，诱导用户点开特制链接才触发，不会存服务器。
3. **DOM 型 XSS**：完全在浏览器前端执行，后端看不到恶意代码，靠前端 DOM 渲染触发。

XSS 主要危害：

1. 盗取 Cookie：拿到用户会话 cookie，直接冒充用户登录账号。
2. 窃取页面数据：读取页面上的隐私信息（手机号、密码、个人资料）。
3. 篡改网页内容：篡改页面，伪造钓鱼弹窗骗密码。
4. 劫持用户浏览器：跳转钓鱼网站、下载恶意程序。
5. 以用户身份执行后台操作：利用 JS 发送同源请求，偷偷执行改密码、转账等操作。
6. 窃取本地存储 localStorage/sessionStorage 里的敏感数据。

> 核心本质：**利用浏览器信任当前域名，执行非授权的 JS 代码**。



下面就开始打 xss 的靶场了。本人是纯萌新，wp会写的比较详细，方便自己复习。

![image-20260831195646058](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260831195646387.png)

# Level 1-无任何过滤

先记住 XSS 的核心：**让你输入的内容被浏览器当成 HTML/JavaScript 执行，而不只是普通文字显示。**

1. 尝试把靶场地址`http://127.0.0.1/xss-labs/level1.php?name=test`中的test换成stw，页面从欢迎用户test变成了欢迎用户stw，说明 `name` 参数的内容会被服务器输出到网页中。

2. F12打开浏览器开发者工具，在“元素/Elements”中找到`<h2 align="center">欢迎用户stw</h2>`，所以可以判断用户输入位于 HTML 标签之间。

3. 尝试插入普通 HTML 标签：为了确认输入是否会被当作 HTML 解析，可以先提交一个没有 JavaScript 的测试内容`<h1>stw</h1>`，页面中的 `stw` 变成大号粗体标题，说明浏览器把我们输入的 `<h1>` 当成了真正的 HTML 标签。

   ![image-20260831211448657](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260831211448805.png)

4. 构造并提交 XSS Payload：既然用户输入可以形成新的 HTML 标签，下一步可以尝试插入 `script` 标签：`<script>alert(1)</script>`。

   ![image-20260831212205831](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260831212205903.png)

成功了！整体执行过程如下：

```
浏览器发送 name 参数
        ↓
PHP 读取 name 参数
        ↓
PHP 把参数拼接到 HTML
        ↓
服务器把 HTML 返回给浏览器
        ↓
浏览器解析并执行 script
```

这属于典型的**反射型 XSS**：参数经服务器放回响应页面，浏览器随即执行。Payload 没有保存进数据库。关闭页面后，它也不会一直存在。只有再次访问包含 Payload 的 URL 时，才会再次触发。

漏洞产生的原因是：程序读取了用户可控制的 `name` 参数，却没有对参数进行 HTML 转义，直接把参数拼接进 HTML。浏览器把用户输入误认为网页代码，导致攻击者提供的 JavaScript 被执行。

关键问题代码：

```
echo "<h2 align=center>欢迎用户".$str."</h2>";
```

修复思路：输入最终被放在 HTML 标签之间，因此输出时应该进行 HTML 编码。

```
$str = $_GET["name"] ?? "";

echo "<h2 align=center>欢迎用户"
    . htmlspecialchars($str, ENT_QUOTES, "UTF-8")
    . "</h2>";
```

`htmlspecialchars()` 会转换具有 HTML 含义的特殊字符。例如攻击者输入：`<script>alert(1)</script>`，经过转义以后，会变成类似：`&lt;script&gt;alert(1)&lt;/script&gt;`浏览器只会把它显示为普通文字：`<script>alert(1)</script>`。不会再将其识别为真正的 `script` 标签。

总结：Level 1 将 `name` 参数未经 HTML 转义直接输出到标签之间，导致用户能够插入 `<script>` 标签并执行 JavaScript，形成反射型 XSS。

# Level 2-闭合标签

1. 依旧尝试把靶场地址`http://127.0.0.1/xss-labs/level2.php?keyword=test`中的test换成stw，页面从没有找到和test相关的结果变成了没有找到和stw相关的结果，搜索框中的内容也从test变成了stw，说明 `keyword` 参数的内容会被服务器输出到网页的这两处。

2. 依旧 F12 打开浏览器开发者工具，在“元素/Elements”中找到两处和stw相关的代码：

   ![943e20430d14185f0fde1f01f8e5d04b](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901000902487.png)

   第一处的 stw 位于 HTML 标签之间，第二处的 stw 位于 HTML 标签的 `value` 属性中。这就是 Level 2 与 Level 1 的关键区别。

3. 尝试用 Less-1 的payload对这一关进行注入，发现不行，后端源码里应该是用`htmlspecialchars()`对具有 HTML 含义的字符进行转换了。

   ![image-20260901001752812](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901001752908.png)

去看一下源码，HTML 标签之间的输入确实是被`htmlspecialchars()`转换了，但是`value` 属性里的没有。这意味着第一处相对安全，但第二处存在 XSS 漏洞。

![0ebabb58b06f390c9045236aad020822](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901002756818.png)

4. 现在只能想办法从第二处注入，虽然前端的搜索框里也是`<script>alert(1)</script>`，代码也没有被执行，但这是因为js代码被放在了value的双引号中，浏览器认为整段内容只是输入框的属性值，而不是一个一个独立的 `script` 标签，所以不会执行。这里就可以想到我们当时学sqli的时候常用的闭合手法给这个双引号闭合了！所以可以构造 payload 为`"><script>alert(1)</script>`。

   ![image-20260901011754068](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901011754153.png)

   

拿下了！整体执行过程如下：

```
浏览器发送 keyword 参数
        ↓
PHP 读取 keyword 参数
        ↓
第一处输出经过 htmlspecialchars() 转义
        ↓
第二处输出未经转义直接放入 value 属性
        ↓
Payload 使用"结束 value 属性
        ↓
Payload 使用>结束 input 标签
        ↓
构造出独立的 script 标签
        ↓
浏览器执行 alert(1)
```

这关依旧属于典型的**反射型 XSS**。Payload 通过 URL 中的 `keyword` 参数发送给服务器，服务器将其直接放入当前页面，浏览器随后解析并执行。Payload 没有被保存到数据库中，只有访问包含该 Payload 的特制 URL 时才会触发。

漏洞产生的原因：程序读取了用户可控制的 `keyword` 参数，该参数被输出到页面中的两个位置：第一处使用了 `htmlspecialchars()` 进行转义；第二处却未经转义直接拼接到了 `value` 属性中。攻击者可以使用 `"` 结束属性，再使用 `>` 结束标签，随后插入独立的 `script` 标签并执行 JavaScript。

关键问题代码：

```
<input name=keyword value="'.$str.'">
```

这也说明：**对同一个参数进行过一次转义，并不代表整个页面就一定安全。必须根据参数的每一个输出位置分别进行安全处理。**

修复思路：因为 `$str` 被输出到了 HTML 属性中，所以应该对属性值进行 HTML 转义，并使用引号将属性值完整包裹起来。

可以修改为：

```
$str = $_GET["keyword"] ?? "";

$safeStr = htmlspecialchars(
    $str,
    ENT_QUOTES,  
    "UTF-8"
);

echo '<input name="keyword" value="'.$safeStr.'">';
```

ENT_QUOTES 表示同时转换双引号和单引号。例如输入：`"><script>alert(1)</script>`，经过转义后会变成类似：`&quot;&gt;&lt;script&gt;alert(1)&lt;/script&gt;` 浏览器只会把它作为输入框中的普通文字，不会把双引号识别为属性的结束位置，也不会把 `<script>` 识别成真正的脚本标签。

总结：Level 2 虽然对 `keyword` 参数的一处输出进行了 HTML 转义，但在 `input` 的 `value` 属性中仍然直接输出原始输入，导致攻击者可以闭合属性和标签，再插入 `script` 标签形成反射型 XSS。

# Level 3-绕过htmlspecials()函数

1. 这关跳转到的的靶场地址是`http://127.0.0.1/xss-labs/level3.php?writing=wait`，但是这里是有一个小bug：输入传入所用的参数使用的是`writing`，但后端源码实际读取的却是`keyword`。但是没关系，我们可以手动输入`keyword=`。并且可以看到这次两个地方都进行了HTML转义。

   ![d56714e6-5261-45c5-9f13-aa55a6f20fa4](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901031421034.png)

2. 输入stw后，页面中和上一关一样依然有两处输出：

   ![8872d34c-0da5-44ff-a735-6de3090f7ba6](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901033117890.png)

   但是这里还有个注意的地方，虽然 Elements 中看到的是value="stw"，是双引号，但刚刚看源码里却是单引号包裹的，`ctrl+u`看一下原始代码确认一下。还真是，查了一下，这是因为：Elements 显示的是浏览器解析并整理后的 DOM。浏览器可能统一使用双引号展示属性，所以 Elements 不一定保留服务器返回时的原始引号。总之，本关原始的属性结构是：`<input name=keyword value='用户输入'>`，用户输入被包裹在一对单引号中。

   ![0dc75f40-856d-49e0-b780-b8415ac95ed3](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901034250321.png)

3. 还记得源码中两个输出位置都使用了`htmlspecialchars()`函数转义，所以level2中的那个payload不行了。但是还记得level2中的我们当时的修复思路是下面这样的吗？

   ```
   $str = $_GET["keyword"] ?? "";
   
   $safeStr = htmlspecialchars(
       $str,
       ENT_QUOTES,  
       "UTF-8"
   );
   
   echo '<input name="keyword" value="'.$safeStr.'">';
   ```

   但这关它却没有传入`ENT_QUOTES`，所以这里的`htmlspecialchars()`只能转换双引号，却不会转换单引号，而这关的属性值恰好是使用单引号包裹的！这就是突破口！

   但是虽然单引号能够结束属性，但是>却还是会被转义，不能结束标签，因此还是无法创建`script` 标签，所以

    `'><script>alert(1)</script>`还是用不了。这时候只能换个思路：不创建新的标签，而是给页面中已经存在的 `input` 标签添加一个能够执行 JavaScript 的事件属性。

   现在可以开始构造payload了：`' onmouseover='alert(1)`。服务器拼接后就得到：`<input name=keyword value='' onmouseover='alert(1)'>`。

   ![image-20260901042555301](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901042555431.png)

拿下了！所以这关真正想让我们理解的是：**当 `<` 和 `>` 被转换、无法创建新标签时，不代表一定无法产生 XSS。如果属性使用未被转义的引号包裹，仍然可以结束原属性，并给已有标签添加 JavaScript 事件。**

整体执行过程如下：

```
浏览器发送 keyword 参数
        ↓
PHP 读取 keyword 参数
        ↓
htmlspecialchars() 转义 <、> 和双引号，单引号没有被转义
        ↓
使用单引号结束 value 属性
        ↓
给已有的 input 标签添加 onmouseover 属性
        ↓
鼠标移动到输入框上
        ↓
浏览器执行 alert(1)
```

依旧是典型的**反射型 XSS**。

漏洞产生的原因是：程序读取了用户可以控制的 `keyword` 参数，并输出到了单引号包裹的 HTML 属性中。虽然使用了 `htmlspecialchars()`，但没有使用 `ENT_QUOTES`，所以用户输入中的单引号不会被转义。导致攻击者可以利用单引号提前结束 `value` 属性。`<` 和 `>` 虽然被过滤，但攻击者可以给已有标签添加事件属性。用户触发事件后，浏览器就会执行其中的 JavaScript！

关键问题代码：

```
<input name=keyword value='".htmlspecialchars($str)."'>
```

修复思路：和上关一样的加上`ENT_QUOTES`。

```
$str = $_GET["keyword"] ?? "";

$safeStr = htmlspecialchars(
    $str,
    ENT_QUOTES,
    "UTF-8"
);

echo '<input name="keyword" value="'.$safeStr.'">';
```

总结：Level 3 虽然使用 `htmlspecialchars()` 过滤了 `<`、`>` 和双引号，但输入位于单引号包裹的属性中，且单引号没有被转义，导致攻击者可以闭合 `value` 属性并添加 `onmouseover` 事件，形成反射型 XSS！

# Level 4-去掉尖括号

1. 靶场地址是`http://127.0.0.1/xss-labs/level4.php?keyword=try harder!`，这次跳转参数和 PHP 实际读取的参数是一致的，没什么毛病。页面依旧是有两个位置输出这个输入：一个HTML 标签之间；一个`input` 标签的 `value` 属性中。

   ![c70af7b6-ad23-43da-8571-34171bfaf17a](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901164039614.png)

   但和 level 3不同的是，这次 `value` 属性值确实是由双引号包裹的。

![3643975e-4652-49fc-8e8b-39c84e6e5b4a](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901171424136.png)

2. 看一下源码，发现属性中的用户输入并没有用`htmlspecials()`函数转义，而是创建了三个变量：`$str`保存用户的原始输入，`$str2`删除原始输入中的`>`，`$str3`继续删除用户输入中`<`，所以，最后的`$str3` 是同时删除了 `<` 和 `>` 的结果。

   ![image-20260921045804446](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921045804569.png)

3. 但是我们可以发现属性值虽然是用双引号包裹的，但是`str3`却没有处理双引号嗷，所以我们直接输入一个双引号就可以离开`value`属性，再和 level3 一样给 `input` 标签添加一个能够执行 JavaScript 的事件属性就可以进行xss了！

   所以构造 payload:`" onmouseover="alert(1)`。

   ![image-20260901174555794](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260901174555923.png)

成功了！整体执行过程如下：

```markdown
浏览器发送 keyword 参数
        ↓
PHP 读取 keyword 参数
        ↓
str_replace() 删除所有的 > 再删除所有的 <
        ↓
第一处输出用 htmlspecialchars() 转义（安全），第二处输出未转义，放入双引号包裹的 value 属性
        ↓
Payload 使用"提前结束 value 属性
        ↓
尖括号已被删除，无法创建新标签，但是可以给已有的 input 标签添加 onclick 事件属性
        ↓
    点击搜索框
        ↓
浏览器执行 alert(1)
```

依旧是典型的**反射型 XSS**。

漏洞产生的原因：程序读取了用户可控制的 `keyword` 参数，没有进行转义，而是用"黑名单删除"的方式过滤——先删掉所有的 `>`，再删掉所有的 `<`，想让攻击者无法创建新标签。但这个黑名单里没有引号，而输出位置又恰好在双引号包裹的 `value` 属性中。攻击者可以用 `"` 提前结束属性；虽然创建不了新标签，但可以给已有的 `input` 标签注入 `onclick` 这类事件属性，用户鼠标移动到搜索框时，其中的 JavaScript 就会被执行。

关键问题代码：

```markdown
$str2=str_replace(">","",$str);
$str3=str_replace("<","",$str2);
...
<input name=keyword  value="'.$str3.'">
```

把 level 3 和 level 4 放在一起看挺有意思：level 3 想转义，但没加 `ENT_QUOTES`，漏了单引号；level 4 干脆不转义了，直接删尖括号，但忘了双引号。**输出位置在 HTML 属性里时，真正的命门是引号，尖括号只是表象。**这也暴露了黑名单式过滤的天生缺陷: 永远不知道自己漏了哪个字符。

修复思路：和 level2、level3一样的。

总结：Level 4 用 `str_replace()` 删除了所有的 `<` 和 `>`，让攻击者无法创建新标签，但没有过滤引号，攻击者可以用 `"` 闭合 `value` 属性并给 `input` 标签注入 `onclick` 事件，点击搜索框即触发，形成反射型 XSS。

# Level 5-绕过检测<script和on事件

1. 输入依旧会在两处输出：

   ![image-20260902025904635](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260902025904894.png)

   并且看下源代码，这次的过滤有三条：

   | `strtolower()`         | 把所有字母转成小写，大小写混写绕过失效                     |
   | ---------------------- | ---------------------------------------------------------- |
   | `<script` → `<scr_ipt` | script 标签一写出来就被改成一个浏览器不认识的陌生标签      |
   | `on` → `o_n`           | 所有 `on` 开头的事件属性（onclick、onmouseover……）全部失效 |

   这就很难受了。script 标签首先用不了，on事件也全部用不了，大小写啥的花活也都失效。

   ![805278b6-1497-47c3-ad57-24e8009d5efa](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260902030445805.png)

2. 盘点一下这关唯一没过滤的一些东西：`<` 、`>`、`"`、`‘`，还有除了 `script` 以外的所有标签名。于是问题就变成了：有没有一种执行 JavaScript 的方式，既不需要 `<script>` 标签，也不需要 `on` 开头的事件？有的，`javascript:` 伪协议。

   > `javascript:` 伪协议：
   >
   > <a> 标签的 href 属性通常放网址：`<a href="http://www.baidu.com">百度</a>`，但 `href` 的值还可以写成：`<a href="javascript:alert(1)">点我</a>`
   >
   > 以 `javascript:` 开头表示：**点击这个链接时，不跳转页面，而是把冒号后面的内容当作 JavaScript，在当前页面里执行。**

   `javascript:` 伪协议不含`<script`，也不含 `on`，完美躲开三条过滤。

3. 构造payload：原始结构`<input name=keyword value="用户输入">`，先使用双引号结束value属性，再使用 `>` 结束 input 标签。

   然后开始新建一个标签`<a href="javascript:alert(1)">x</a>`，所以最后服务器拼接后得到`<input name=keyword  value=""><a href=javascript:alert(1)>x</a>">`。最终 Payload：`"><a href=javascript:alert(1)>x</a>`。

   ![image-20260902040257292](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260902040257380.png)

拿下！整体执行过程如下：

```markdown
浏览器发送 keyword 参数
        ↓
PHP 读取 keyword 参数
        ↓
strtolower() 全部转成小写，str_replace() 把 <script 替换成 <scr_ipt，str_replace() 把 on 替换成 o_n
        ↓
第一处输出用 htmlspecialchars() 转义（安全），第二处输出未转义，放入双引号包裹的 value 属性
        ↓
Payload 使用 "> 闭合 value 属性并结束 input 标签，插入 <a> 标签，href 使用 javascript: 伪协议
        ↓
点击页面上的 x 链接
        ↓
浏览器执行 alert(1)
```

还是**反射型 XSS**。

漏洞产生的原因是：程序读取了用户可控制的 `keyword` 参数，没有对输出位置做转义，而是采用三条黑名单替换过滤：小写化、点杀 `<script`、点杀 `on`。但 `<`、`>`、引号全部原样放行，攻击者可以闭合 `value` 属性、结束 `input` 标签，插入任意非 script 标签；`javascript:` 伪协议既不含 `<script` 也不含 `on`，躲过全部点杀，用户点击链接时，其中的 JavaScript 就会被执行。

关键问题代码：

```markdown
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
...
<input name=keyword  value="'.$str3.'">
```

把 1-5 关连起来看，攻击方的思路是一条清晰的递降链：

| 关卡    | 防护                                        | 我们的打法                      |
| ------- | ------------------------------------------- | ------------------------------- |
| Level 1 | 无任何过滤                                  | 直接插 `<script>`               |
| Level 2 | 一处转义，value 裸奔                        | `">` 逃出后插 `<script>`        |
| Level 3 | 转义漏了单引号，而 value 值恰好被单引号包裹 | `'` 逃出属性加事件              |
| Level 4 | 删除所有尖括号，但 value 值被双引号包裹     | `"` 逃出属性加事件              |
| Level 5 | 小写化 + 点杀 script 和 on                  | 逃出标签 + `javascript:` 伪协议 |

这就保留了黑名单过滤的一些弊端：**黑名单是点杀，永远杀不干净——只要还有一种能执行 JS 的载体没进名单，XSS 就依然成立。**

修复思路：依然是一样的答案——别追着 payload 打补丁，输出到属性之前统一转义：

```markdown
$safeStr = htmlspecialchars(
    $str,
    ENT_QUOTES,
    "UTF-8"
);

echo '<input name="keyword" value="'.$safeStr.'">';
```

总结：Level 5 在小写化的基础上点杀了 `<script` 和 `on`，堵死了 script 标签和事件注入两条路，但没有过滤 `<`、`>` 和引号，攻击者可以闭合属性插入 `<a>` 标签，用 `javascript:` 伪协议在点击时执行 JS，形成反射型 XSS。

# Level 6-大小写绕过

1. 这关输入的输出位置还是前面一样，没区别。看下源码，这次有五条过滤！

   | 过滤                   | 点杀目标                                     |
   | ---------------------- | -------------------------------------------- |
   | `<script` → `<scr_ipt` | script 标签                                  |
   | `on` → `o_n`           | 所有 on 系事件                               |
   | `src` → `sr_c`         | `img`、`iframe` 等的 `src` 资源属性          |
   | `data` → `da_ta`       | `data:` 协议（另一种加载代码的方式）         |
   | `href` → `hr_ef`       | `a` 链接的 `href`，也就是 Level 5 的绕过方式 |

   script、on、src、data、href。靶场把前五关出现过的所有载体挨个杀了。但是上一关的 `strtolower()`被删了！

   ![c5de48f7-5ada-4fbe-8d67-b6131d148d21](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260902042314271.png)

2. 现在盘点局面。`<`、`>`、引号放行；`script、on、src、data、href` 五个被点杀；没有`strtolower()`，大写字母活下来了。

   这里要引入两个知识点：**第一，`str_replace()` 是大小写敏感的。**PHP 的字符串函数默认按字节逐个比较，它只认得全小写的，只要大小写不完全一致，就可以蒙混过关。**第二，浏览器解析 HTML 时，标签名和属性名是大小写不敏感的。**在浏览器眼里，这些大小写没区别，是同一个东西！

   所以，这里存在一条完美的缝隙：过滤器只认小写，浏览器不挑大小写！攻击者可以直接大小写绕过，非常简单！

3. 这里就可以直接构造 payload：`"><scRipt>alert(1)</scRipt>`

   ![image-20260902043510838](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260902043510941.png)

完成的不错！整体执行过程如下：

```markdown
浏览器发送 keyword 参数
        ↓
PHP 读取 keyword 参数
        ↓
str_replace() 把 <script 替换成 <scr_ipt，str_replace() 把 on 替换成 o_n
str_replace() 把 src 替换成 sr_c，str_replace() 把 data 替换成 da_ta，str_replace() 把 href 替换成 hr_ef
        ↓
第一处输出用 htmlspecialchars() 转义（安全），第二处输出未转义，放入双引号包裹的 value 属性
        ↓
Payload 使用 "> 闭合 value 属性并结束 input 标签，插入大小写混写的 <scRipt> 标签，躲过全小写点杀
        ↓
浏览器把 <scRipt> 当作 script 标签解析
        ↓
页面加载，脚本自动执行 alert(1)
```

还是**反射型 XSS**。

漏洞产生的原因是：程序读取了用户可控制的 `keyword` 参数，没有对输出位置做转义，而是采用五条**大小写敏感**的黑名单替换：点杀 `<script`、`on`、`src`、`data`、`href`。但它删掉了 level 5 的 `strtolower()`，且 `<`、`>`、引号全部原样放行。攻击者可以用 `">` 闭合属性、结束标签，插入大小写混写的 `<scRipt>`——过滤器认不出它，浏览器却照常执行。

修复思路还是和之前一样无区别。

总结：Level 6 用五条替换点杀了 script、on、src、data、href，但删掉了小写化过滤，且 `<`、`>`、引号未过滤，攻击者可用大小写混写的 `<scRipt>` 绕过全部点杀并自动执行，形成反射型 XSS。

# Level 7-script等标签绕过一次移除操作，双写绕过

1. 看源码。点杀名单和 Level 6 一模一样：`script`、`on`、`src`、`data`、`href`。但有两个关键变化：第一，点杀的不再是 `<script`，而是裸的 `script`。第二，替换值是空字符串。过滤策略从“点杀改写”变成了“彻底删除”！

   ![eae5899b-cd60-4c9a-8de2-d2ff02ef28ce](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921044527195.png)

2. 盘点一下局面：`<`、`>`、引号 放行；大小写绕过死了；script 标签、on 事件、href 链接、伪协议这些全死。但是想想我们当时在 sqli 里学的双写的两个成立条件：关键词是多个字符组成的词、过滤是删除式的。Level 7 恰好两条全满足。所以这关的思路就是双写绕过！开始构造payload：原始结构`<input name=keyword value="用户输入">`， 使用双引号结束 value 属性；然后使用 `>` 结束 input 标签；再插入双写版 script 开标签`<scrscriptipt>`；然后加入 `JavaScript`；最后插入双写版闭合标签`</scrscriptipt>`。所以最终payload为：`"><scrscriptipt>alert(1)</scrscriptipt>`。

   ![image-20260921050615706](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921050615822.png)

完成的不错！

# Level 8-关键字 ASCII 编码

1. 看了下参数的输出位置，发现这关结构大变。虽然依旧是两个输出位置，但这次我们的输入被放进了 `<a>` 标签的 `href` 属性里，成了这条链接的地址。（前七关我们一直在攻击输入框的 `value`）。

   ![937d97fc-1bed-4362-a8a5-24aa5290bf43](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921054927392.png)

2. 看下源码。过滤共六条替换 再加 一条小写化：

   | 过滤                 | 点杀目标                                           |
   | -------------------- | -------------------------------------------------- |
   | `strtolower()`       | 大小写混写                                         |
   | `script` → `scr_ipt` | script 标签及一切含该词的内容（包括 `javascript`） |
   | `on` → `o_n`         | on 系事件                                          |
   | `src` → `sr_c`       | 资源属性                                           |
   | `data` → `da_ta`     | data: 协议                                         |
   | `href` → `hr_ef`     | href 属性                                          |
   | `"` → `&quot`        | **引号，首次被杀**                                 |

   盘一下局面：引号被杀 → 逃不出 href 属性；`value` 处被转义 → 旧路走不通；唯一的思路是：既然出不去，就让 href 的值本身变成可执行的 JS，也就是让这条友情链接的地址变成 `javascript:alert(1)`。但 `script` 这个词被点杀，所以有没有一种写法，让过滤器看不见`script` 这个词，但浏览器看得见？有，那就是**HTML 实体编码**！这里有一个很有意思的对称：`htmlspecialchars()` 防御时，是把 `<` 编码成 `&lt;`，让浏览器认不出标签；我们攻击时，是把 `s` 编码成 `&#115;`，让过滤器认不出单词。攻守双方用的是同一个思路。

   ![](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921055209926.png)

3. 构造payload：过滤器点杀`script`，所以我们要在`javascript` 的后六个字母里选一个进行替换。这里替换`s`。最终payload：`java&#115;cript:alert(1)`。

![image-20260921062310691](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921062310816.png)

# Level 9

1. 这关我们的参数好像只在一个地方输出了：

   ![image-20260921063744178](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921063744338.png)

2. 看下源码，发现还是两处输出，只不过第二处我们的输入要先过一道资格审查，合格才能在前面输出。所以，这一关payload 里必须包含字面的 `http://`。所以思路很简单：加上一个`http://` 再注释掉就行了。

   ![2de46610728a6e7248a33b658e766bde](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921063938268.png)

3. 构造payload：`java&#115;cript:alert(1)//http://`

   ![image-20260921064302447](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921064302594.png)

拿下！

# Level 10-利用hidden参数传递数据

1. 观察页面，本来以为这个页面没有表单，打开开发工具才发现里面其实藏着三个 `<input>`。`type="hidden"`代表是隐藏字段。这种输入框不会显示在页面上，通常被用来在表单里偷偷携带数据——用户看不见，但它在 HTML 里真实存在。三个隐藏字段的名字：`t_link`、`t_history`、`t_sort`，我们的输入哪个都没进，`keyword` 在这关只去了 `h2`（转义过的），露个脸迷惑人。

   ![914b8aaa07d19aadc3a48a371f4954b0](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921065945025.png)

2. 看一下源码。服务端其实是会读取第三个参数 `t_sort`——页面上没有任何输入框，但可以从 URL 传！三个隐藏字段里，只有 `t_sort` 的 `value` 用了用户输入 `$str33`；`t_link` 和 `t_history` 的 value 是写死的空串，纯属摆设；过滤也很简单，只是删除了`<` 和`>`。所以这关的注入点就在 `t_sort`上。

   ![image-20260921070224265](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921070224378.png)

3.  因为`<`、`>`都被删了，所以这关的绕过方法其实和 Level4一样，把 payload 改成`" onclick="alert(1)`，但又有一个新的问题：这个输入框是隐藏的，鼠标根本扫不到。所以要考虑怎么让隐藏框现身。

   这里需要用到一条新的 HTML 知识：浏览器解析一个标签时，如果遇到**同名属性出现多次**，只有**第一次出现的生效**，后面的同名属性全部忽略。比如`<input type="text" type="hidden">`，浏览器只认第一个 `type="text"`，第二个 `type="hidden"` 被无视——这个输入框是**可见**的。

   所以我们这关 payload 构建为：`" type="text" onclick="alert(1)` 

   然后注意在 url 中把这关 payload 赋给`t_sort`。

   ![image-20260921071839428](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921071839591.png)

# Level 11-利用Referer请求头

1. 按 `F12` 展开 form，这次隐藏字段变成了四个，而且很骚的是，这个新的 `t_ref`的 value 里居然是上一关通过时的完整URL。

![782da25cf9bf6ec76192bb59bf0484b4](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921170459042.png)

2. 看下源码。`t_sort` 这次套了 `htmlspecialchars()`走不通了。然后还有了新的输入源：`$_SERVER['HTTP_REFERER']`——PHP 从HTTP 请求头里取 Referer。过滤还是和上关一模一样：删除所有 `<` 和 `>`，引号放行；输出进 `t_ref` 的 `value`。

   Referer 是每次浏览器向服务器发请求时带着的“浏览器→服务器”的介绍信息，服务器常用它做来源统计、图片防盗链。我们在 `t_ref` 里看到 Level 10 的 URL，就是浏览器自动带上的。但是问题来了：Referer 是浏览器默认填，不是只有浏览器能填。任何能自己构造 HTTP 请求的工具（Burp、curl、浏览器插件）都可以伪造它。对服务器来说，请求头和 URL 参数一样，都是**用户可控的输入**。

   ![f930cb97-5644-4a44-b762-56818d38960c](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921171543402.png)

   3. 因为Referer 里的过滤和 Level 10 一模一样，所以照搬 Level 10 的 payload `" type="text" onclick="alert(1)`就可以。这一关的新知识在运载工具。打开Burp，开启拦截，访问`http://127.0.0.1/xss-labs/level11.php?keyword=stw`，在拦截到的请求头里添加一行：`Referer: " type="text" onclick="alert(1)`。

      ![image-20260921184327885](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921184328007.png)

      完成的不错！

      ![image-20260921183945196](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921183945560.png)

# Level 12-利用User-Agent字段

1. 这关其实和 Level 11 没啥区别，利用点从 Referer 变成了 User-Agent。

   ![f44f4d039782814453368baa753073da](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921201404251.png)

2. 源码的处理也都基本一样。UA是纯浏览器的介绍，服务器常拿它做访问统计、兼容性适配、反爬虫识别。和 Referer 一样，它也只是浏览器默认会填，任何工具都能伪造。对服务器来说，它就是又一处用户可控输入。

   ![image-20260921201558669](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921201558794.png)

3. payload 依旧是：`" type="text" onclick="alert(1)`。burp开启拦截，请求`http://127.0.0.1/xss-labs/level12.php?keyword=good%20job!`，将拦截到的请求中的`User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...`换成`User-Agent: " type="text" onclick="alert(1)`。

   ![image-20260921202809604](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921202809739.png)

拿下！

![bc4f9a332136cb8477c615b14ee7137d](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921202850135.png)

# Level 13-利用Cookie值

1. 这次的新字段是 `t_cook`，value 是 "call me maybe?"。

   ![dd0221cb-54e9-481d-897c-d435779c35f3](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921205235087.png)

​	结合字段名，我们去看下浏览器中现在的cookies值恰恰就是`call+me+maybe%3F`。

![image-20260921205527764](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921205527905.png)

2. 源码中也能看到服务器每次访问都给浏览器发一张 Cookie——`user=call me maybe?`，有效期一小时（`time()+3600`）；PHP 从请求的 **Cookie 头** 里读 `user` 的值；过滤还是老样子：删 `>` 删 `<`，引号放行，输出进 `t_cook` 的 value。

   Cookie 是服务器让浏览器在本地保存的键值对：服务器通过响应头 `Set-Cookie: user=` 下发；浏览器存起来，之后**每次请求都自动**通过 `Cookie: user=...` 带回给服务器。

   但是**Cookie本身对用户是完全透明可改的**，F12 里就能看、能编辑。所以我之前做后端开发的时候常听到的铁律就是“永远不信任客户端数据”。Cookie 里常存着登录凭证，这正是 XSS 的头号目标！

   ![image-20260921205725017](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921205725160.png)

3. payload还是之前的payload：`" type="text" onclick="alert(1)`。和 Referer、UA 不同（浏览器禁止 JS 设置那两个头），**Cookie 本来就存在浏览器里，随手就能改，不用通过burp**。

   直接F12修改：

   ![image-20260921210534037](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921210534165.png)

成功拿下！

![7da0f9b8e13e19b3e1b69bb5e056235e](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921210550337.png)

# Level 14-坏关

坏关，直接 Pass。

# Level 15-利用文件解析xss

1. 这关很怪啊，没有表单、没有隐藏字段、输入不回显。页面就一句话：“欢迎来到第15关，自己想个办法走出去吧！”

   ![6d862e21-3d8c-4135-b6ea-bb3e8601538d](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921222632609.png)

2. 源码也很简洁，直接一个`htmlspecialchars` 。看似无懈可击。

   总结一下可利用的点：1. `htmlspecialchars`默认不转义单引号，所以单引号存活。2.就算全部转义了，这一关也不需要逃出属性，注入面是在 class 里的框架指令。3.使用的是`AngularJS`的js框架，使用了`ng-include`这个表达式的意思是当HTML代码过于复杂时，可以将部分代码打包成独立文件，再使用`ng-include`来引用这个独立的HTML文件。**简单来说就是我们可以引入另一个文件然后在另一个文件中插入 JS 恶意代码执行。**

   于是攻击模型成立：payload 自己在 Level 15 绕过不了过滤 → 让 ng-include 去包含另一个php → 把 payload 塞进 这个新 php 的参数中 →  新 php 零过滤输出 → 响应被插进本页 → 触发！

   ![image-20260921230149719](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921230149941.png)

3. 构造payload：这里我们采用Level1.php这个零过滤文件。一步步来：一对单引号分别开始字符串变量和结束字符串变量；然后插入level1.php和它的参数：`level1.php?name=`；然后输入XSS脚本：`<img src=x onerror=alert(1)>`。注意这里为什么用 `img onerror` 而不用 `<script>`：**ng-include 插入内容走的是 DOM 编译路径**，这样插入的 script 标签不会执行，而`img` 的错误事件是最可靠的触发器。所以最终payload为：`'level1.php?name=<img src=x onerror=alert(1)>'`

   ![image-20260921234642605](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260921234642750.png)

# Level 16-替换空格

1. 可以看到，这是 Level 1 以来第一次，输入直接落在标签之间——不在任何 `value` 属性里，没有引号要闭合，没有属性要逃逸。我们可以光明正大创建新标签。

   ![468a48aa-bfbf-476c-951a-e88ac0bdefa3](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922001148594.png)

2. 看源码，四条替换 + 小写化：

   | 过滤                 | 点杀目标         |
   | -------------------- | ---------------- |
   | `script` → ` &nbsp;` | script 标签      |
   | `空格` → ` &nbsp;`   | 属性分隔符       |
   | `/` → ` &nbsp;`      | 斜杠             |
   | `TAB` → ` &nbsp;`    | 另一个属性分隔符 |

   没有 on 过滤、没有引号过滤、没有尖括号过滤——新标签随便建，`onerror` 随便挂。但问题是：标签里多个属性之间必须用空白分隔，而空白全被杀了。

   然而在HTML规范里，属性分隔符有五种，过滤器只杀了两种：

   | 字符               | 写法  | 本关是否被杀 |
   | ------------------ | ----- | ------------ |
   | 空格 (0x20)        | `%20` | ✗ 被杀       |
   | TAB (0x09)         | `%09` | ✗ 被杀       |
   | **换行 LF (0x0A)** | `%0A` | ✓ 存活       |
   | **回车 CR (0x0D)** | `%0D` | ✓ 存活       |
   | **换页 FF (0x0C)** | `%0C` | ✓ 存活       |

   又来了，这个贯穿整个靶场的母题：**过滤器按自己的字符清单杀字节，解析器按语法规范认字符，两个集合对不齐，缝隙就永远在。**（Level 8 是实体解码层，Level 15 是框架表达式层，这一关是空白字符层。）

   ![image-20260922001443654](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922001443780.png)

3. 构建 payload。尖括号、引号都没过滤，先按理想结构写：`<img src=x onerror=alert(1)>`。然后把两个空格都换成 `%0D`（回车符）：`<img%0Dsrc=x%0Donerror=alert(1)>`。

   ![image-20260922002923211](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922002923329.png)

# Level 17- 空格造属性

1. 页面跳转到`http://127.0.0.1/xss-labs/level17.php?arg01=a&arg02=b`，URL 里第一次出现两个参数（`arg01` 和 `arg02`），而且都进了页面。F12看，网页里嵌了一个 **Flash 动画**，两个参数被拼在了它的 `src` 属性值里面。

   ![35e630b9-1dc7-4053-a223-bfcf4158b20f](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922003636796.png)

2. 源码很短。直接把两个参数都套上`htmlspecialchars()`。但是`src` 的属性值是**无引号**包裹的：`src=xsf01.swf?a=b`——这是本关的命门。

   **HTML 知识：无引号属性值，遇空格即终止**

   回忆 HTML 属性的两种写法：

   ```html
   <input value="a b">    ← 有引号：空格是值的一部分
   <input value=a b>      ← 无引号：值在第一个空格处结束！
   ```

   无引号的属性值，遇到第一个空格就**终止**，空格后面的内容被解析器当作**下一个属性**。

   那么既然`src` 的值是无引号的，那我们在 `arg02` 里塞一个空格，就能让 `src` 提前结束，**凭空造出一个新属性！**

   ![image-20260922003840611](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922003840748.png)

3. 构造 payload：`?arg01=a&arg02=b onmouseover=alert(1)`。服务器拼接后：

   ```html
   <embed src=xsf01.swf?a=b onmouseover=alert(1) width=100% heigth=100%>
   ```

​	`src` 值提前结束，`onmouseover` 成为 embed 的合法属性！

![image-20260922004739526](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922004739662.png)

# Level 18

1. 这关的页面和 Level 17是一模一样的。

2. 看下源码。除了自动跳转到下一关其他也一模一样。

   ![de5f0839-de0b-4cc6-942c-9b68f7d5c430](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922005253385.png)

3. 什么都没区别，没搞懂这个是18关来干啥的。payload 还是`?arg01=a&arg02=b onmouseover=alert(1)`。

   ![image-20260922005502081](https://fastly.jsdelivr.net/gh/Stjorn/image_bed@main/images/20260922005502210.png)

# Less 19-坏关

坏关，直接 Pass。

# Less 20-坏关

收官之战！！

呃还是坏关。

# 总结

| 关卡     | 防护                              | 我们的打法                                       |
| -------- | --------------------------------- | ------------------------------------------------ |
| Level 1  | 无任何过滤                        | `<script>alert(1)</script>`                      |
| Level 2  | 一处转义，value 裸奔              | `"><script>alert(1)</script>`                    |
| Level 3  | 转义漏单引号                      | `' onmouseover='alert(1)`                        |
| Level 4  | 删除尖括号                        | `" onclick="alert(1)`                            |
| Level 5  | 小写化+点杀script/on              | `"><a href=javascript:alert(1)>x</a>`            |
| Level 6  | 五连点杀，删了小写化              | `"><scRipt>alert(1)</scRipt>`                    |
| Level 7  | 小写化+五连删除                   | `"><scrscriptipt>alert(1)</scrscriptipt>`        |
| Level 8  | 转义+六连替换+杀引号，href 注入点 | `javascript:alert(1)`                            |
| Level 9  | + `http://` 校验                  | `javascript:alert(1)//http://`                   |
| Level 10 | 注入点藏进隐藏字段 t_sort         | `" type="text" onclick="alert(1)`                |
| Level 11 | t_sort 补丁，注入点换 Referer 头  | `Referer: " type="text" onclick="alert(1)`       |
| Level 12 | 注入点换 User-Agent 头            | 同款                                             |
| Level 13 | 注入点换 Cookie                   | `Cookie: user=" type="text" onclick="alert(1)`   |
| Level 14 | 坏关                              |                                                  |
| Level 15 | Angular 指令上下文                | `'level1.php?name=<img src=x onerror=alert(1)>'` |
| Level 16 | 标签之间回显+杀空白               | `<img%0Dsrc=x%0Donerror=alert(1)>`               |
| Level 17 | embed 无引号 src                  | `?arg01=a&arg02=b onmouseover=alert(1)`          |
| Level 18 | 同 17                             | 同款                                             |
| Level 19 | 坏关                              |                                                  |
| Level 20 | 坏关                              |                                                  |

**三条主线：**

1. **1-9 关：payload 进化论**——从裸 `<script>` 开始，过滤器每堵一条路（属性转义、删尖括号、点杀关键词、小写化、内容校验），攻击者就退一步换载体（事件注入 → 伪协议 → 大小写 → 双写 → 实体编码 → JS 注释）；
2. **10-13 关：输入面四部曲**——URL 参数、Referer、User-Agent、Cookie，同一套洞换四个马甲，教会“凡是回显皆是注入点”；
3. **14-20 关：现实世界的样子**——依赖第三方服务会死（14）、框架有框架的缝（15）、语法字符集对不齐永远有缝（16）、插件型 XSS 随插件一起被时代埋葬（17-20）、以及**正确的防御是真的有效的**（19/20 就是修复示范）。
