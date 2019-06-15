---
title: DNS解析
description: DNS解析
date: 2018-09-06 16:46:42
keywords: java
categories : [计算机网络]
tags : [计算机网络, DNS]
comments: true
---

nslookup
<img src="/images/nslook-baidu.png">

第一行Server是：DNS服务器的主机名--10.4.1.14
第二行Address是：它的IP地址--10.4.1.14#53

百度有一个cname = www.a.shifen.com的别名

下面的Name是：解析的URL--www.a.shifen.com
Address是：解析出来的IP--220.181.112.244和220.181.111.188

dig www.baidu.com +trace
<img src="/images/dig-baidu.jpg">	

Dig工具会在本地计算机做迭代，然后记录查询的过程
- 第一步，向我这台机器的ISPDNS获取到根域服务区的13个IP和主机名[b-j].root-servers.net.
- 第二步，向其中的一台根域服务器（m.root-servers.net）发送www.baidu.com的查询请求，他返回了com.顶级域的服务器名称
- 第三步，向com.域的一台服务器i.gtld-servers.net请求,www.baidu.com，他返回了baidu.com域的服务器IP（未显示）和名称，百度有5台顶级域的服务器
- 第四步，向百度的顶级域服务器（202.108.22.220）请求www.baidu.com，他发现这个www有个别名，而不是一台主机，别名是www.a.shifen.com

当dns请求到别名的时候，查询不会终止，而是重新发起查询别名的请求，此处返回的是www.a.shifen.com，然后继续请求

使用dig www.a.shifen.com +trace查看
<img src="/images/dig-shifen.jpg">	

再一次去请求com域，重复上面的步骤，最终从ns X.a.shifen.com中一台拿到了一条A记录，便是www.baidu.com的IP地址了