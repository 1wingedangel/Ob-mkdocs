---
title: shadowsocks 安全打洞、SSL安全连接、自定义二级域名
class: article
status: working
created-date: 2024-03-28T12:08:26.528+08:00
tags:
  - date/2024/03/28
aliases: 
share: true
category: article
---

%%
> [!example]+ **Comments:**     
>  | Date___ | Comments |
> | ------- | -------- |
> 
>  **Comment**:: 
%%

# shadowsocks 安全打洞、SSL安全连接、自定义二级域名

之前为了满足我在自己的工作环境下访问家里的服务器用的 zerotier 服务被环境屏蔽了之后，我折腾出了通过在我的软路由（旁路由、iStoreOS）上安装 shadowsocks 服务器，然后结合 clash meta 配置规则让家里的局域网IP走 ss 的方式实现了安全打洞访问家里服务器，也顺道解决了绕开工作环境下有一些很奇怪的屏蔽规则实现的全局代理上网。从去年9月开始使用至今一直没有问题，我也很满意。

但是自从 iOS 升级之后，有很多应用已经不再接受不安全的 http 连接，并强制要求**有效的**HTTPS 连接。HTTPS 对于通过 shadowsocks 打洞访问的服务器意义其实不大，但是 iOS 的这番要求已经蔓延到了 Obsidian 甚至是音乐播放器 Navidrome 上，那我就得想办法搞定 HTTPS 的历史遗留问题了。经过在网上的一些探索，HTTPS 协议需要先申请一个域名、然后通过 ACME 进行认证发行证书，然后再用 nginx反代，不仅麻烦而且还需要把这些服务器暴露到公网上去。

所以我就在想，既然都用 shadowsocks 打洞实现了网络互通，那么直接用自签名的证书进行内网 SSL 连接行不行。查了一下发现这种需求在企业内部进行通信的案例中非常地多，经过一些折腾，也总算是实现了我需要的功能。这种方案相比申请域名后获取SSL的方案，有以下特点：

- 服务器不需要暴露在公网上。反正服务器也是自用，只需要能够使用软路由上的 shadowsocks 服务器的人使用就可以了。
- 不需要花钱申请域名，也不需要进行网上认证。
- 通过与自定义 hosts 结合使用，可以自己随便写一个域名访问家里的服务器 

不过也有一些需要注意的地方：

- 完全依赖软路由上的 shadowsocks。
- 需要给每个设备安装根证书。不过反正自用，需要访问的设备也有限。
- 证书很难自动续签。一般一年一换。
- 因为要改hosts，需要搭配 clash 使用。嘛本来 shadowsocks 也要搭配 clash 使用所以问题也不大。

## 准备工具

- openWRT（实际情况还是 `iStoreOS`）
  - 反向代理软件。本文用的是 Lucky。
- clash。建议使用 meta 内核
- openssl。在文中通过 Powershell 操作。注意不是 openssh……

## 生成 SSL 证书

我们通过开源工具 openssl 来生成 SSL 证书。openssl 各个OS的软件源里应该都有，Windows 的话需要找第三方编译的二进制程序。我用的是 [Sourceforge 里的 OpenSSL for Windows](https://sourceforge.net/projects/openssl-for-windows/)。解压到随便一个文件夹之后，打开该路径的 PowerShell 进行接下来的操作。

在进行 SSL 连接的过程中，我们需要准备两种证书，一个是认证机构的根证书又称 CA 证书，还有一个是服务器上的 SSL 证书。CA证书验证 SSL 证书是否有效，而SSL 证书验证 https 是否有效。所以我们需要先新建一个随机密钥，然后根据这个密钥建立 CA 证书：

```ps
# 通过rsa算法生成2048位长度的秘钥
.\openssl.exe genrsa -out myCA.key 2048
# 生成 CA 证书，有效时长是1年
.\openssl req -utf8 -config .\openssl.cnf -new -x509 -key .\myCA.key -out myCA.cer -days 365
```

在这里会让你输入机构信息，分别是：

- 国家代码，如 `CN`
- 省名称
- 城市名称 
- 机构名称
- 机构单位名称
- 授权人
- 邮件地址

因为是自己发行自己签名，所以其实可以随便填写，只要能和其他证书分清楚就可以了。

然后创建服务器证书。因为要创建的服务器证书要和服务器IP进行绑定，所以需要手动修改 openssl 的配置文件 `openssl.cnf`，在 `[v3_req]` 部分添加几行内容：

```conf
[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = example.com
DNS.2 = *.example.com
IP.1 = 192.168.1.1
IP.2 = 192.168.1.2
```

这里的 `[alt_names]` 里面填写的是你的服务器IP和可能会用到的域名。域名支持通配符，但是IP不行，只能写准确的IP地址。

同样先创建一个服务器用的密钥、然后创建签名申请文件（需要像刚才输入CA机构信息一样输入服务器信息）：

```ps
.\openssl.exe genrsa -out server.key 2048
.\openssl.exe req -utf8 -config .\openssl.cnf -new -out server.req -key .\server.key
```

最后，对这个签名申请文件进行签名，得到最终的服务器密钥文件 `server.cer` 和 `server.key` ：

```ps
.\openssl.exe x509 -req  -extfile .\openssl.cnf -extensions v3_req -in .\server.req -out server.cer -CAkey .\myCA.key -CA .\myCA.cer -days 365 -CAcreateserial -CAserial serial
```

需要注意的是，因为苹果和 Google 的设备要求 SSL 证书的有效时间在368天以内，所以我们只能创建365天左右的有效期的证书，太长的话系统可能不会认为这个证书是安全的。

## 导入根证书到设备 

每个系统导入根证书的方式不同，我这边就以 Windows 和 iOS 为例。

### [Windows](Windows.md) 

我们需要在每一台需要访问家里服务器的设备上获取刚才获得的 CA证书 （`myCA.cer`），然后双击即可。此时会出现一个安装选项，点击`安装证书`。

然后会进入安装证书的向导，先选择存储位置，因为是自用服务器的验证，所以我一般选择`当前用户`。然后他会问你保存在哪个位置，这里必须要手动选择 `受信任的根证书颁发机构`。然后一路允许就可以了。

P.S. 如果需要删除证书的话在开始菜单搜索 `管理用户证书` 即可。

### iOS 

- 通过任意方式把 `myCA.cer` 传输到手机中，然后通过`文件`应用打开。此时会提示你到设置app安装描述文件。
- 打开设置app，你就会发现第一条变成了`已下载描述文件`，点进去后就可以安装描述文件。
- 安装完之后会出现在 `VPN与设备管理` 里看到这个描述文件。
- 回到 `通用`，打开 `关于本机` , 滚到最下面的 `证书信任设置` 。
- 在 `针对根证书启用完全信任` 里在证书上切换成启用，一路允许过去。

删除证书的时候也是反过来操作，现在根证书上禁用，然后去VPN那里删除描述文件。

## 导入服务器密钥 

服务器证书和密钥分别是 `server.cer` 和 `server.key` ，只要把这两个文件放到各个服务器设置里就可以实现 HTTPS 连接。对于本来就支持 SSL 连接如 alist 等，就直接指定这两个文件的文件路径就可以。

```json
  "scheme": {
    "address": "0.0.0.0",
    "http_port": 5244,
    "https_port": 5245,
    "force_https": false,
    "cert_file": "data\\server.cer",
    "key_file": "data\\server.key",
    "unix_file": "",
    "unix_file_perm": ""
```

至少 alist 的话这样配置好就可以直接通过 `https://<alist 的 IP地址>:5245` 来安全访问了。但是对于本身没有SSL接口的服务器应用，就得通过反向代理来解决。

### 配置反向代理 

一般情况下好像还是用 nginx 比较多，但是本文用的是 Lucky 。也没有其他原因，只是因为 iStoreOS 的商店里有 Lucky 可供快捷安装，另外就是 Web UI 比较方便。

iStoreOS 装好 Lucky 之后，先在 luci 界面启用服务，然后通过 端口 `16601` 访问 Web UI。

- 到 Web 服务，添加一个 web 规则。监听端口为想要服务器监听的 https 端口。
- 监听类型选上 `tcp4`
- 启用防火墙自动放行和 TLS。
- 添加子规则。服务类型为反向代理，前端地址可以是 Lucky 服务器的IP，后端地址为你的目标服务器的IP和端口。
- 点击修改完成

此时你就可以通过 `https://<Lucky服务器IP:监听端口>` 来安全访问目标服务器了。需要注意的是有时候会牵涉到跨域的问题，此时需要点开定制模式，然后启用跨域支持。

## 自定义域名 

这样子问题是解决了，不过随着开启的服务器越来越多，占用的端口也就越来越多，自己访问的时候也越来越难记住了。那我们能不能利用反向代理的特点，用二级域名来访问这些服务器呢？既然都自签名证书了，而且也用 clash 来走 ss 了，何不在 clash 进行 hosts 配置，然后指定域名走lucky，然后lucky再转给各个服务器呢？

### 修改 CA 证书和服务器证书 

根据上面的步骤重新生成 SSL 证书。此时把自己想好的域名添加到 `[v3_req]` 中。根据这个配置再重新生成证书。当然，密钥不需要重新生成，只需要重新生成证书。

当然，更新好的证书要重新导入到各个设备中去。

### 添加反向代理 

- 添加 Web 服务规则。监听端口为一个共用端口，443 最好。但是因为 istoreos 已经占用了 443，所以我只能用别的。
- 添加子规则。不同于之前添加的反向代理需要一个服务器添加一个 web 规则，我们只需要给每个服务器分配一个子规则即可。前端地址为我们自己想好的二级域名（如：`alist.example.com`）,后端地址为目标服务器的IP和端口。

### 添加 clash 配置

到 clash 的配置文件中，添加hosts。

```yaml
hosts:
  '*.example.com': <Lucky服务器IP>
```

这样的话只要是这个域名的连接都会解析为lucky服务器IP，然后通过反向代理分配到指定的服务器上去。

# 参考

- [局域网内搭建浏览器可信任的SSL证书 – 唐玥璨 | 博客 (tangyuecan.com)](https://www.tangyuecan.com/2021/12/17/%e5%b1%80%e5%9f%9f%e7%bd%91%e5%86%85%e6%90%ad%e5%bb%ba%e6%b5%8f%e8%a7%88%e5%99%a8%e5%8f%af%e4%bf%a1%e4%bb%bb%e7%9a%84ssl%e8%af%81%e4%b9%a6/)
- [网络千万条，安全第一条——使用Lucky轻松实现反向代理+Https外网访问家庭NAS_NAS存储_什么值得买 (smzdm.com)](https://post.smzdm.com/p/awz4dnrk/)