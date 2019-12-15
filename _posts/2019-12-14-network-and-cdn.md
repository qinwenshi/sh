---
layout: post
title: 中国与欧洲直接网络延迟太大，怎么办？
description: 中国与欧洲直接网络延迟太大，怎么办？
categories: [development]
tags: [development, network]
redirect_from:
  - /2019/11/09/
---

# 背景

从去年开始，最让我头疼的问题之一就是中国到其他国家，特别是到欧洲的网络延迟。在[G2Rail](https://www.g2rail.com)，我们最早接通的铁路公司应该就是意大利国铁（Trenitalia），刚刚上线的时候，意铁的API通路可以说是相对来说比较稳定的。到了去年开始，频繁发现中国去意大利的网络丢包，或者连接超时，一时间客人和partner怨声载道。

意铁的这个API，真是一言难尽，首先，他们给我们的授权用了一个叫做NTLM的认证协议。有网友评论说，这个协议十分Microsoft，具体啥意思，我一会后面再说。估计是因为这个协议的原因，所以意铁所有的连他们的API的server都必须要经过他们的IT工程师，把固定的IP地址放进一个白名单里面。第三个问题，意铁的这个API，如果搜索多人的多段行程，我们最多需要从他们服务器上接受近5MB的数据。

# 第一把尝试

为了解决网络超时的问题，我简单粗暴把超时时间从默认的1分钟改成了1分30秒。这下倒是不超时了，就是网络不好的时候，点击了一次搜索，去上个厕所，接杯水回来，估计正好看到结果展示出来。

# 第二把尝试

经过上面的一步我寻思，光等也不是个办法啊，特别是在国家有重大会议、或者敏感的纪念日期的时候，网络慢得简直抓狂。这时候我开始琢磨着是不是通过什么方法来做一个加速（违法通过VPN或者Shadowsocks等方式首先被排除了，这样可能会导致整个服务器被查封，风险太高）。因此我想了个办法是在境外通过一台机器作为跳板。

初步想法是从日本中转，再到意铁本身在罗马的机房。所以我规划是用nginx在日本做一个到意铁的反向代理。前面说过，意铁的服务器是IP有地址白名单，所以我们必须得走一个十分漫长的流程，先在日本买一个VPS，然后把IP地址发给意铁，等了2个月之后终于给我开通了。反向代理刚上线就碰了一鼻子灰，意铁直接挂了，连不上。

因为NTLM！

  > Microsoft NTLM uses stateful HTTP, which is a violation of the HTTP/1.1 RFC. It relies on authentication (an affair which involves a handshake with a couple of initial 401 errors) and subsequent connections to be done through the exact same connection from client to server. This makes HTTP proxying nearly impossible, since each request would usually go through either a new or a random connection picked from a pool of open connections. It can be done though.

Google一番，有人提到了如下解决方案 [NTLM Nginx](https://stackoverflow.com/a/33096713)，重点是:
 1. 在upsteam里面加上keeplive，表示同时维护16个活跃的连接 
 2. 在server里面，指定http version是1.1
 3. 把Header里面的Connection清空

```json
upstream http_backend {
    server 2.3.4.5:80;
    keepalive 16;
}
server {
    ...
    location / {
       proxy_pass http://http_backend/;
       proxy_http_version 1.1;
       proxy_set_header Connection "";
    ...
    }
 }
 ```

 NTLM是一个有状态的协议，授权协商通过后，依赖于已经被授权过的TCP连接来完成后续的工作（发送请求，收到回复）。

 在通过NGINX的一番改造之后，意铁消停了一几天。到国庆节前夕，因为SEO的效果、同时也是旺季，G2Rail的并发量忽然就变得很高。而上面的NGINX的设置会时不时出现一种很奇怪的错误，说我们使用了【未授权认证的NTLM连接来访问意铁的资源】，所以大量的订单预定失败了。呼叫中心的同学们又疯了，看来还得折腾。。。

# 第三把尝试

 经过分析，我怀疑是因为上述配置的NGINX，没有意识到NTLM协议是对连接状态有要求的，而它自己会把对其他无状态的连接重用用于发送给意铁的报文，这种是非常随机的错误，并发量小的时候测不出来，一旦对各个铁路公司的并发量增多就会出现问题。

 这时候我内心已经把弄出这种协议的微软同行骂了好多遍了，当然腹诽还是解决不了问题的。我还是在琢磨，有没有什么办法，让中国访问意铁的服务器的请求只是经过日本那台机器中转一下，而不是通过Nginx来做反向代理呢？
 
 我把GitHub翻了个底朝天，最后还真找到一个神奇的代码库[Chisel](https://github.com/jpillora/chisel)，它现在也有3400+ Star了，也不算太小众。

Chisel仓库里面是是这么描述自己的。

![BunleNameChangeSet](/image/2019-12-14/chisel-overview.png){:width="450px"}

也就是说客户端通过一个加密通道连接到服务器，然后服务器建立一个到目标的TCP连接，转发所有客户端发送过来的信息。

如果你也想试试这个小工具，可以参考教程：https://0xdf.gitlab.io/2019/01/28/tunneling-with-chisel-and-ssf.html

实现步骤也比较简单，服务器端（日本主机）运行通道的服务端，中国主机运行客户端。服务器端通过Cloudflare解析为 http://trenitalia.g2rail.com，并且把对意铁API的443端口访问暴露在本地的 9001 端口。

服务端

```bash
./chisel server -p 8080
```

客户端

```bash
chisel client --auth user:pass http://trenitalia.g2rail.com:8080 9001:trenitalia.it:443
```

通道建立好之后，在widnows服务器里面修改对意铁的endpoint为 https://localhost:9001 就可以实现访问

这个通道上线之后，我很是得瑟了一段时间。因为不管怎么测试，都不会出现Nginx那个”未授权的连接“错误了。好景不长，过了大概2个星期，发现这个连接会非常随机失效，失效后把chisel client重新启动一下就又好了。究竟是啥原因，我有几个猜测，不过没有验证：

1. chisel client把一个HTTPS服务器放给了HTTPS的一个localhost，而localost实际上我没有给他生成很合证书，虽然我试了跳过证书验证的逻辑。但是似乎还是不行。
2. chisel本身不够稳定，或者对NTLM授权方式不是很友好，连接保持时间长了出现了问题。

另外发现还有一个问题，日本设置了这台主机之后，速度有所提升，但是还是会有300ms左右的延迟，所以搜索结果最快也需要8～10秒回来。

# 第四次尝试，哪里摔倒了就换个地方爬起来

此时如果有个微软的专家在我眼前，我一定会哭着想他求助的，为啥要搞NTLM这种不符合规范的协议出来。但是事实上，只有Google那条细长的搜索框，默默鼓励我不要放弃。。。

我把Github里面搜索出来的所有NTLM的项目看了个遍，在最后一页，看到了一个没有任何Star的项目[ntlm-reverse-proxy](https://github.com/xynova/ntlm-reverse-proxy)。让我喜出望外，灵光一闪 —— 既然NTLM协议是一个非标准的协议，为什么我要在所有的代码里面去适配它了？为什么不把它限制在一个十分小的范畴内里面呢？让我来捋一下迄今为止我的网络链路中发生的事情。

总体上来说，这个连接是这样的：
  中国机器 <---> 日本跳转机器 <----> 意铁罗马网关

当发起一次搜索请求的时候，会经过如下步骤：

### NTLM协商

1. 第一个来回，未登录 
  请求连接罗马这个机器发现这个连接没有被授权，于是就发回一个Challenge要客户端通过约定好的用户名算一个HASH请求登陆
  
  中国机器 --[请求连接]--> 日本机器 --[请求连接]--> 意铁罗马机器

  中国机器 <--[Challenge]-- 日本机器 <--[Challenge]-- 意铁罗马机器

2. 第二个来回发送认证hash
  发回认证信息，建立真正有效的连接
  
  中国机器 --[NTLM Hash]--> 日本机器 --[NTLM Hash]--> 意铁罗马机器

  中国机器 <--[hash认可，连接有效]-- 日本机器 <--[hash认可，连接有效]-- 意铁罗马机器

3. 发送搜索请求
  此时再发送搜索请求，比如出发站，到达站，哪天出发，几个人乘车

  中国机器 --[搜索请求]--> 日本机器 --[搜索请求]--> 意铁罗马机器

  中国机器 <--[搜索结果]-- 日本机器 <--[搜索结果]-- 意铁罗马机器

为了一个授权，通过中转服务器来来回回好几次。我想，其他的铁路公司，简简单单就是通过TLS传输数据，并且通过报文自身的一个字段来做签名验证，要不然就是通过Basic授权来实现，也没听说出了啥安全问题。而这个破NTLM基本上没有很好的加速方法，也就是说cloudflare或者Akamai这种CDN对他也束手无策。

绕回来说我为啥看到ntlm-reverse-proxy很激动。
1. 它的代码很简单，简单意味着出问题几率更少
2. 它把授权的过程封装起来，所有的铁路公司的challenge都由它来处理:
  如下图所示，也就是说，中国的机器只需要把搜索请求发给日本的机器，由日本这台机器来处理NTLM授权的问题，所有授权登录相关的问题不再回到中国的机器。
  ![BunleNameChangeSet](/image/2019-12-14/NTLM-reverse-proxy.jpg){:width="450px"}
3. NTLM的有状态的连接被我的NTLM reverse proxy改造成无状态的http协议，从而我也可以通过cloudflare来加速中国和日本之间的链路，也可以把这台反向代理的IP地址隐藏在CDN网络之后。

# 结论

这是一个持续了超过一年的持续尝试和优化的马拉松。到现在为止还没有打到最优状态。我有个假设，我可以把日本这台机器移到离意铁服务器更近的地方，比如法兰克福，获取到搜索结果之后，走Cloudflare的链路回到中国服务器。如你所知，IP地址白名单还在申请中。。。。

这个过程对我来说是一个十分有启发的过程，我们永远会遇到问题，而如何把影响全局的问题，通过各种手段限制在一个十分受控的小范围内？这也许是一个值得花更多时间思考的。