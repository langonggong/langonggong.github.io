---
title: https
description: https
date: 2018-10-26 17:44:29
keywords: https
categories : [计算机网络,https]
tags : [https]
comments: true
---

<img src="/images/https-connect.png">

- 客户使用https的URL访问Web服务器，要求与Web服务器建立SSL连接。
- Web服务器收到客户端请求后，会将网站的证书信息（证书中包含公钥）传送一份给客户端。
- 客户端的浏览器与Web服务器开始协商SSL连接的安全等级，也就是信息加密的等级。
- 客户端的浏览器根据双方同意的安全等级，在本地生成一对对称密钥作为会话秘钥，然后利用网站的公钥将会话密钥加密，并传送给网站。
- Web服务器利用自己的私钥解密出会话密钥。
- Web服务器利用会话密钥加密与客户端之间的通信

为什么要同时用到对称秘钥和非对称秘钥?

- 对称秘钥加密、解密效率高，涉及web页面和没提内容输出时更高效
- 对称秘钥不能以明文直接发给服务器，必须用非对称秘钥加密
- 对称秘钥是长度较短的随机字符，适合用非对称秘钥加密