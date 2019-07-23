---
layout: post
title:  最新ruby源
author: AdminTest
date:   2019-03-06
catalog:    true
header-img: ""
header-mask: 0.6
tags:
    - Ruby
    - Other
---

#### 关于新的ruby源替换（最新）
15年之前使用ruby很大概率是使用的淘宝源`http://ruby.taobao.org/`，但在此之后，我们发现淘宝源没有再接着维护了，于是我们换成国内的ruby官方源`https://gems.ruby-china.org/`，但我们现在使用发现这个源的链接也出现问题，无法连接上这个源，

添加国内ruby源 

```
gem sources --add https://gems.ruby-china.org
```

报错：
```
Error fetching https://gems.ruby-china.org/:

bad response Not Found 404 (https://gems.ruby-china.org/specs.4.8.gz)
```

这其实是国内的ruby源的域名进行了更换，我们只需要将`".org"`改为`".com"`即可重新连接国内的ruby源

具体命令替换为 ```gem sources --add https://gems.ruby-china.com ```则解决问题



