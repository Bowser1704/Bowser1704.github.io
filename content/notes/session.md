---
title: "Session"
date: 2021-03-01T22:41:24+08:00
draft: false
comments: false
images:
---

## Session

分为两种：server-side-session 和 client-side-session，日常见的基本是 server-side-session，放在 cookies 里面的是 session-ID, 通过这个 ID 去后台 session 数据库里面找 session-data。

### JWT

[JSON Web Token](https://tools.ietf.org/html/rfc7519) 可以看做是 client-side-session 的一种。

JWT 只是一种处理数据的手法，它通过签名保证了信息的不可篡改。

1. JWT 默认是不加密的，分为三个部分

   header, payload, signature

   ![image-20210301224509717](https://i.loli.net/2021/03/01/3KgpvAdNL46sYOi.png)

   > base64URL 特殊的 base64
   >
   > Using standard Base64 in [URL](https://en.wikipedia.org/wiki/URL) requires encoding of '`+`', '`/`' and '`=`' characters into special [percent-encoded](https://en.wikipedia.org/wiki/Percent-encoding) hexadecimal sequences ('`+`' becomes '`%2B`', '`/`' becomes '`%2F`' and '`=`' becomes '`%3D`'), which makes the string unnecessarily longer.
   >
   > For this reason, **modified Base64 for URL** variants exist (such as **base64url** in [RFC](https://en.wikipedia.org/wiki/RFC_(identifier)) [4648](https://tools.ietf.org/html/rfc4648)), where the '`+`' and '`/`' characters of standard Base64 are respectively replaced by '`-`' and '`_`'

   signature 用来保证 jwt 没有被篡改，是 server 发出去的，这里用对称加密（HMAC）和非对称加密都行

   由于 jwt 中的 header 和 payload 默认都没有加密所以不要放敏感数据，当然我们可以把信息加密放进去。

2. jwt 放的位置一般是是 header：Authorization：Bearer

   不要放在 cookies 里面，前端存在 local storage，每次发请求带到 header 里面去。