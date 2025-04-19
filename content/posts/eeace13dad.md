---
date: '2024-12-29T09:39:37+08:00'
draft: false
title: 'Clash 配置文件简述'
tags: ["代理", "教程"]
noindex: false
enableKaTeX: false
cover:
  image: "/img/eeace13dad.md/cover.jpg"
  alt: "吃花的小猫咪"
  hidden: false
---

最近词元发现之前搭建的代理，访问部分国内网站的时候不光速度很慢，而且有时候会报 PR_END_OF_FILE_ERROR（无法确定内容完整性），就是远程，特别是 CSDN，全站都无法访问。

> 😮‍💨 唉：后来发现其实是服务端 WARP 的问题，跟 v2rayA 一毛钱关系都没有。

词元现在使用的 v2rayA 虽然安装起来很方便（Arch Linux CN 有打包），但是对于规则的支持并不算好，并且开启、关闭、切换节点的速度也不尽人意。在网上搜索一通，似乎眼下最完善的内核是 Clash 系列的后继者 Mihomo，因此词元就用它来学习一下**如何书写 Clash 配置文件**。

## 💻 客户端选择

Clash 系的客户端在 AUR 上基本全部都有打包。根据词元的测试，似乎还在开发且可用性比较高的，只有 Clash Verge Rev 和 FlClash，而后者的 UI 设计词元看着非常舒服，因此本文就以 FlClash 作为测试客户端。

首先对 FlClash 的软件设置进行调整。虽然诸多 Clash 系客户端使用同一个内核，但是其 UI 设计却千差万别。FlClash 的界面大概是这样的。

![image-20241229103214948](/img/eeace13dad.md/image-20241229103214948.png)

在 Tools 栏目中，主要是软件本身和覆写配置文件的选项，前者是主题、日志、自动启动等本地定义的选项，不出现在配置文件中，因此我们首先来完成对其的设置。覆写配置文件的设置主要针对使用远程配置文件（例如机场）的用户，方便修改配置文件。

<img src="/img/eeace13dad.md/image-20241229103541287.png" alt="image-20241229103541287" style="zoom:50%;" />

选项大部分您都应该能理解，说几个比较迷惑的：

- Logcat：就是日志功能，记录您使用代理访问的网站，关闭之后不会记录，但是似乎需要手动删除之前的记录（位于您的主目录下）。
- Auto lost connections：在切换节点之后切断当前存在的连接，开启之后换区域会更加彻底，但是您的下载会被终端。
- Only statistics proxy：只对通过的流量进行记录，这时候 FlClash 就变成了一个网络监测工具。

当然这个栏目还有其他功能，比如设置语言、主题、备份之类的，就请您自己探索了。

## ⚙️ 配置文件

这当然就是本文的重点了！通过 Clash 配置文件，您可以方便地修改 DNS、分流、（基础）去广告等，至于节点链接，倒成了配置文件中不太重要的一部分了。下面词元参考了一些链接和项目，为行文流畅就不一一列出角标了，一并在文末引用。

您书写的配置文件应当保存在任意名称的 `.yaml` 文件中，在 FlClash 中这样引用：

![image-20241229111508530](/img/eeace13dad.md/image-20241229111508530.png)

![image-20241229111702011](/img/eeace13dad.md/image-20241229111702011.png)

### 🏠 基础

Clash 配置文件使用 YAML 的语法，以 `key: value` 的形式书写，注意中间的空格不可以省略。如果一个键有子键，那么需要缩进并添加 `-`；注释以 `#` 开头。示例请见下文说明。

一个 Clash 配置文件类似于这样：

```yaml
mixed-port: 7890
allow-lan: false
# bind-address: "*"
ipv6: true
mode: rule
log-level: info
# external-controller: 127.0.0.1:9090
# secret:""
dns:
proxies:
proxy-groups:
rules:
```

每一个选项的含义：

- `mixed-port`：混合端口，即 Clash 内核所使用的端口，支持 HTTP、SOCKS 等代理协议混用。
- `allow-lan`：是否允许局域网连接，如果您希望连接其他设备可以连接代理，您应当将其设置为 `true`，公用网络下无论如何都应当设置为 `false` 来保证安全。
- `bind-address`：绑定地址，在 `allow-lan: true` 的情况下，端口允许的地址，例如 `0.0.0.0` 代表您设备在网络中的 IP 地址（所有 IPv4 设备均可连接），`127.0.0.1` 代表回环地址（只有本机可以连接），`::` 代表您设备的 IPv6 地址等等；设置为 `"*"` 即允许所有设备连接。
- `ipv6`：是否启用 IPv6，这要看您路由器是否支持。设为 `true` 没啥坏处。
- `mode`：模式，接下来我们将使用规则集，因此要设成 `rule` 即规则模式。其他还有 `direct`（不走代理）和 `global`（全走代理）。
- `log-level`：日志等级，`info` 比较适中，也可设成 `error`（仅错误）、`warning`（错误和警告）、`debug`（所有记录）、`silent`（不输出）。
- `external-controller` 和 `secret`：控制端口和密码，非常建议设置为 `127.0.0.1` 而非 `0.0.0.0`，后者将把控制界面向局域网开放。
- 其他的是接下来讨论的重点。

当然其实还有个 `authentication` 即代理认证功能，请您自行查阅文末引用，有详细介绍。本文主要讨论作为本地代理的功能，不过度介绍。

## 🗺️ DNS

在上述配置中继续添加。

DNS 是互联网上用于查找域名对应 IP 地址的工具，您在浏览器中输入的所有域名都会被发送到 DNS 服务器解析，然后再访问 IP。由于纯 DNS 使用的是未经加密的 UDP 包，非常容易被污染篡改，早年间的 GFW 主要就采用 DNS 污染的方式封禁域名。

说实话，词元建议您就算不使用代理，也按照下文方法（在 FlClash 或操作系统中）设置安全的 DNS 方案，详情请见文末引用的部分文章。

但是后来由于 DoH 即 DNS over HTTPS 技术的出现（其实还有 HTTPS 本身），GFW 原本准确的 DNS 污染和选择性屏蔽页面形同虚设，就开始使用 IP 黑洞这种最简单、最暴力的方法屏蔽一些网站。

> 当然对于 Cloudflare 这种 IP 大户还是没什么用，于是对于 `*.pages.dev` 和 `*.workers.dev` 又开始了域名封锁。

无论如何，您在使用代理时，最好还是要添加一下 DNS 配置。例如：

```yaml
dns:
  enable: true
  prefer-h3: true
  use-hosts: true
  use-system-hosts: true
  respect-rules: false
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
#  fake-ip-filter:
#    - '*.lan'
#    - "+.local"
#    - localhost.ptlogin2.qq.com
  default-nameserver:
    - 8.8.8.8
    - 1.1.1.1
    - 223.5.5.5
    - 119.29.29.29
# nameserver-policy:
#   'www.baidu.com': '114.114.114.114'
#   '+.internal.crop.com': '10.0.0.1'
#   'geosite:cn,private':
#   - https://223.5.5.5/dns-query
#   - https://223.6.6.6/dns-query
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
    - 8.8.8.8
    - 1.1.1.1
  fallback-filter:
    geoip: true
    geoip-code: CN
    geosite:
      - gfw
    ipcidr:
      - 240.0.0.0/4
      - 0.0.0.0/32
      - 127.0.0.1/32
#    domain:
#      - '+.google.com'
#      - '+.facebook.com'
#      - '+.youtube.com'
```

简单解释一下：

- `enable`：启用 DNS 设置。

- `prefer-h3`：加密的、使用 QUIC 实现的 DNS 协议，有助于加快查询速度。

- `use-hosts` 和 `use-system-hosts`：是否使用 `hosts` 优先查询。`hosts` 是独立于 DNS 系统的查询文件，一般在操作系统中定义，可以把某些地址强制解析到特定 IP 上，例如实际上 `localhost` 这个地址并不存在，是操作系统在 `hosts` 中定义了 `127.0.0.1 localhost`。建议开启。

- `respect-rules`：优先遵守下文规则的设置，如果您使用机场，不建议开启，因为如果机场定义了 DNS 设置会导致您的配置失效。（如果没有，就无所谓了，这玩意儿就成了“自尊”😄）

- `fake-ip-*`：这是个比较有意思的东西。正常来说，浏览器查询 DNS 服务器，得到 IP，然后访问；但是如果 Clash 开启了 `fake-ip`，浏览器得到 Clash 的解析结果都指向 `fake-ip-range` 中的一个随机地址，然后对其发送请求，再由 Clash 来请求服务端。

  > 由此也可以发现，Clash 运行时一定会劫持 DNS，所以如果您发现 DNS 设置不正常，不必惊慌。

  `fake-ip-filter` 还可以定义不使用 `fake-ip` 的地址，一般不需要，除了 QQ 对于本地的请求也使用域名……

- `default-nameserver`：如果定义了 `nameserver`，此配置将用于解析 DoH 域名，从上至下轮询直到得到第一个结果。由于我们使用代理，就优先选择国外服务；如果您想配置不开代理的 DoH，需要将国内的几个调到上面。

  > DoH 地址也是域名，也需要解析……所以有的时候称为“引导域名服务器”。

- `nameserver`：默认 DNS 服务器，但是由于下文设置了 `fallback-filter`，所有 GeoIP 非 CN 的域名实际上会采用 `fallback`，因此这里仅针对国内网站。

- `nameserver-policy`：域名解析规则，对于特定域名采用指定 DNS 服务器。

- `proxy-server-nameserver`：用于解析代理服务器域名的 DNS 服务器，建议设国内地址，因为这个流量不走代理。

- `fallback`：后备 DNS 服务器。若第一轮解析结果显示其 GeoIP 非 CN，则使用这些服务器验证。

- `fallback-filter`：过滤器，负责决定哪些域名需要使用 `fallback` 验证：

  - `geoip-code`：**除了**该国家代码的解析结果，全部视为已经污染，进入 `fallback`。

  - `geosite`：在列表中的全部视为污染，进入 `fallback`。

  - `ipcidr`：如果解析结果是这些网段，则认为是污染。

    > 这是因为 GFW 在 DNS 污染之后还是要返回结果的，通常就是回环地址或者不存在的 IP，因此可以根据这一点来判断是否被污染。文末的维基学院参考提供了一个污染返回的结果列表，但是词元未采用，因为该列表有极大概率滥杀无辜 😮‍💨

  - `domain`：对于这些域名，直接使用 `fallback`。

### 👷‍♂️ 代理

这里你应当填写你的订阅链接。如果您获得的链接是类似于 `vless://...` 的格式，您需要先进行转换。下面词元给出一个示范。

对于 `vless://[UUID]@[DOMAIN]:[PORT]?encryption=[ENCRYPTION]&security=[SECURITY]&type=[TYPE]&host=[DOMAIN]&path=[UUID2]#[TAB]` 这样的链接，如果我们使用 v2rayA 进行解析（自动分开各个部分），就会发现实际上是这个意思：

<img src="/img/eeace13dad.md/image-20241229135017078.png" alt="image-20241229135017078" style="zoom:50%;" />

那么根据 Clash 对应格式，就应该写为：

```yaml
proxies:
  - name: "any_name_is_okay"
    type: vless
    server: [DOMAIN]
    port: [PORT]
    uuid: [UUID]
    udp: true
    tls: true
    network: ws
    servername: [DOMAIN]
    ws-opts:
      path: "/[UUID]"
      headers:
        Host: [DOMAIN]
```

如果您使用的方案与词元不相同，请您查看参考中的 GitLab 仓库，其中有最常见的 13 种代理的配置模板。

这部分没什么好说的，词元也变不出来新的订阅 🧙‍♂️

### 🧭 策略组

如果您使用机场，机场应当已经给您配置好了，直接下载即可。如果您需要自定义，也可以在网上搜一搜好用的模板，直接下载即可。对于词元自建节点来说，就不需要了（只有一个节点）😄

只需要添加一个 PROXY 组，代表使用代理，Clash 内置了 REJECT 和 DIRECT 组，分别表示“拒绝连接”和“直接连接”。

```yaml
proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "any_name_is_okay"
```

### 🔌 分流规则

这应该是这篇文章的重头戏了！毕竟词元写这篇文章就是因为吃了没有做好分流的苦。

> ⚠️ 注意：部分站点（如 ChatGPT）会到处连接来确定您是否位于 OpenAI 不提供服务的区域。在这种情况下，建议启用全局代理。

由于网站的数量是无限的，无论是您还是词元还是网上的大神，这部分配置大概都不是一个字一个字打出来的，因此我们直接采用别人搞好的规则集。词元搜了一圈，好像比较好用的是 Loyalsoldier/clash-rules（链接请见参考），如下配置：

> 词元采用了全套配置，可能稍有多于，您可以自行删除一部分

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400

  icloud:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/icloud.txt"
    path: ./ruleset/icloud.yaml
    interval: 86400

  apple:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/apple.txt"
    path: ./ruleset/apple.yaml
    interval: 86400

  google:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/google.txt"
    path: ./ruleset/google.yaml
    interval: 86400

  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400

  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400

  private:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/private.txt"
    path: ./ruleset/private.yaml
    interval: 86400

  gfw:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/gfw.txt"
    path: ./ruleset/gfw.yaml
    interval: 86400

  tld-not-cn:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/tld-not-cn.txt"
    path: ./ruleset/tld-not-cn.yaml
    interval: 86400

  telegramcidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/telegramcidr.txt"
    path: ./ruleset/telegramcidr.yaml
    interval: 86400

  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400

  lancidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/lancidr.txt"
    path: ./ruleset/lancidr.yaml
    interval: 86400

  applications:
    type: http
    behavior: classical
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/applications.txt"
    path: ./ruleset/applications.yaml
    interval: 86400
```

以上是对规则集的引用。

- `type`：规则集的协议类别，本文均为针对 HTTP。
- `behavior`：规则集的内容类别。
- `url`：下载地址，本文选择使用 jsDelivr 的源。
- `path`：下载的规则集的保存位置。
- `interval`：更新时间间隔，这个规则集使用 GitHub Actions 每天更新一次，因此我们也每天重新下载一次。

下面需要继续添加规则，来使用这些规则集：

```yaml
rules:
  - RULE-SET,applications,DIRECT
  - RULE-SET,private,DIRECT
  - RULE-SET,reject,REJECT
  - RULE-SET,icloud,DIRECT
  - RULE-SET,apple,DIRECT
  - RULE-SET,google,PROXY
  - RULE-SET,proxy,PROXY
  - RULE-SET,direct,DIRECT
  - RULE-SET,lancidr,DIRECT
  - RULE-SET,cncidr,DIRECT
  - RULE-SET,telegramcidr,PROXY
  - GEOIP,LAN,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```

> 由于词元不适用 YACD 面板，以上规则集删除了两个 YACD 面板的域名。

这个部分应该很好理解：

- `RULE-SET`，`GEOIP`，`GEOSITE` 或 `DOMAIN` 表示匹配的类型。
- 后面是匹配的属性。
- `DIRECT`，`REJECT` 和 `PROXY` 分别代表“直接访问”“拒绝访问”和“使用代理访问”。

词元是 Arch Linux 用户，需要不时使用 BFSU、USTC 或者 TUNA 的源进行更新，并且词元不希望使用代理下载包（容易中断，而且浪费流量），因此额外添加了几条：

```yaml
rules:
  ...
  - DOMAIN,mirrors.bfsu.edu.cn,DIRECT
  - DOMAIN,mirrors.ustc.edu.cn,DIRECT
  - DOMAIN,mirrors.tuna.tsinghua.edu.cn,DIRECT
```

以上是白名单模式，因此不在列表中的域名会自动采用代理。

## 🎆 下课

经过这些配置，应该差不多就可以顺利地使用代理了！

这些只是最基本的配置，Clash Meta 内核还在不断发展，也不断有新的协议和工具出现。词元建议您看看结尾的参考文章，相信能给您更多启发！

> 💡 提示：本文不提供完整的样板。词元认为配置 Clash 是一个非常个人化的工作，没有所谓的“最佳实践”。请您认真阅读本文，相信您可以配置得比词元更好！

> ➕ 本文参考或引用了以下来源：
>
> [【保姆级教学】掰碎了给你讲！Clash配置文件详解（含实战演练）](https://linux.do/t/topic/163682)（原作者：[崔裕姝](https://linux.do/u/Yuju)）
>
> [Loyalsoldier/clash-rules: 🦄️ 🎃 👻 Clash Premium 规则集(RULE-SET)，兼容 ClashX Pro、Clash for Windows 等基于 Clash Premium 内核的客户端。](https://github.com/Loyalsoldier/clash-rules)（原作者：[Loyalsoldier](https://github.com/Loyalsoldier)）
>
> [Clash 知识库](https://clash.wiki/)（原作者：原文未标注，疑似 [Dreamacro](https://github.com/Dreamacro)）
>
> [DNS 污染和劫持原理](https://xiking.win/2019/03/27/dns-cache-pollution/)（原作者：[人身如逆旅，我亦是行人](https://xiking.win/)）
>
> [防火长城域名服务器缓存污染 IP 列表](https://zh.wikiversity.org/wiki/%E9%98%B2%E7%81%AB%E9%95%BF%E5%9F%8E%E5%9F%9F%E5%90%8D%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%BC%93%E5%AD%98%E6%B1%A1%E6%9F%93IP%E5%88%97%E8%A1%A8)（[维基学院](https://zh.wikiversity.org)）
>
> [Misaka-blog/clash-meta: 这是一个基于 Clash Meta 订阅的配置文件模板](https://gitlab.com/Misaka-blog/clash-meta)（原作者：[Misaka-blog](https://gitlab.com/Misaka-blog)，博客已删）

> 2025 年 1 月 26 日更新：在 Linux 发行版中，可能出现开启了 TUN 模式却无法接管流量的情况：可以通过手动设置代理来科学上网，但是不能直接使用。这是因为 TUN 模式要求 Root 权限来绑定小于 1024 的端口，但是运行一般是使用普通用户，Mihomo 内核无法绑定。可以通过以下方式解决：
>
> ```bash
> sudo setcap cap_net_admin=+ep /usr/share/FlClash/FlClashCore
> ```
>
> 也就是给内核赋予网络权限。
>
> Xray 内核的 v2rayN 采取了更加简单粗暴的方法：存储 `sudo` 密码；v2ray 内核的 v2rayA 则使用 Root 权限作为 systemd 服务运行。二者都绕过了这个权限问题。
