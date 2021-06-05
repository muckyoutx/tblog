---
title: 解决windowsupdate被组策略管理
img: false
cover: windowsupdate.JPG
top: false
tags: windows
categories: windows
abbrlink: 40587
date: 2021-06-04 21:22:18
summary:
password:
---

​		我的电脑windows更新被组策略限制，找到解决方法：

```bash
Microsoft Windows [版本 10.0.18362.1256]
(c) 2019 Microsoft Corporation。保留所有权利。

C:\Windows\system32>rd /s /q "%windir%\System32\GroupPolicyUsers"

C:\Windows\system32>rd /s /q "%windir%\System32\GroupPolicy"

C:\Windows\system32>gpupdate /force
正在更新策略...

计算机策略更新成功完成。
用户策略更新成功完成。


C:\Windows\system32>
```

{%asset_img win.jpg %}
