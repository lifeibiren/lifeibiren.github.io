---
title: "关于Proxy的各种姿势"
date: 2025-06-28T02:20:38+08:00
---

# Preface
proxy server 的地址记为 PROXY

proxy server 的端口记为 PORT

*NOTE: 某些 proxy server 不同的协议的端口号不同*

# Linux & MinGW

## 环境变量

最常见的 proxy 指定方式，需要依赖 app 本身支持，通常仅针对 http/https 协议进行配置

* 若 proxy server 和 app 都支持 http/https
```
export HTTP_PROXY=http://PROSY:PORT
export HTTPS_PROXY=http://PROSY:PORT
```

* 若 proxy server 和 app 都支持 socks5（Recommend）
```
export HTTP_PROXY=socks5://PROSY:PORT
export HTTPS_PROXY=socks5://PROSY:PORT
```

上面的 export 命令可以放到 ~/.bashrc 中，对新启动的 bash 生效。

## [ProxyChains](!https://github.com/haad/proxychains)

proxychains 通过 hook libc 的网络函数来实现代理，但是这种方式要求被代理的程序动态链接到相同的 libc。显然，对静态链接或者不链接到 libc 的程序没有作用，例如 golang 编写的程序。

* 安装（以 Archlinux 为例子）

```
pacman -S proxychains
```

* 配置（以 socks5 为例）

```
nano /etc/proxychains.conf

...
# meanwile
# defaults set to "tor"
# socks4        127.0.0.1 9050
socks5 PROXY PORT
```

* 使用例子

```
proxychains git clone https://www.XXX.com/YYY
```

## 特定程序的配置方法

### SSH

ssh 支持对指定的 host 配置 proxy。

下面这个例子仅对 XXX.com 配置了 proxy，使用了 netcat（注意在 Archlinux 中是 openbsd-netcat），使用了 socks5 协议。

```
cat ~/.ssh/config
...
Host XXX.com
  ProxyCommand /bin/nc -X 5 -x PROXY:PORT %h %p
...
```

# Windows
TODO