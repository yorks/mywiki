---
title: 简单理解 https
slug: simple-understand-https
date: 2012-12-01
tags:
  - https
  - ssl
  - key
  - security
---


个人简单的理解，当作记录一下.

1. client connet 到 server.
2. server 发 pubkey 给 client <!-- more -->
3. client 用 生成一个对称密钥 pass，并用`2`收到的 pubkey 进行加密, 加密后为s_pass.
4. client 将http内容 经过SSL层加密后和s_pass一起发送给server (请求)
5. server 收到后用 private_key 对 s_pass 进行解密，并用 pass 对http内容进行对称加密，发送给client.
6. client 解开，展现出来


* 由pubkey加密过的密钥，可以使用私钥(private_key) 进行解密.
