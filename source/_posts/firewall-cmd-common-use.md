title: firewall-cmd 常用命令使用
tags:
  - firewall
  - firewall-cmd
categories:
  - Linux
author: Steven Hu
date: 2018-03-15 18:33:00
---
简要记录一下 firewall-cmd 常用命令的使用。

### 启用端口

```bash
$ firewall-cmd [--zone=<zone>] --add-port=<port>[-<port>]/<protocol> [--timeout=<seconds>]
```

如下示例为添加 tcp 18080 端口:

```bash
$ firewall-cmd --add-port=18080/tcp
```

### 禁用端口

```bash
$ firewall-cmd [--zone=<zone>] --remove-port=<port>[-<port>]/<protocol>
```

如下示例为禁用指定的 tcp 18080 端口:

```bash
$ firewall-cmd --remove-port=18080/tcp
```

### 启用服务

```bash
$ firewall-cmd [--zone=<zone>] --add-service=<service> [--timeout=<seconds>]
```

如下为启用 http 服务:

```bash
$ firewall-cmd --add-service=http
```

<!--more-->

### 禁用服务

```bash
$ firewall-cmd [--zone=<zone>] --remove-service=<service>
```

如下为禁用 http 服务:

```bash
$ firewall-cmd --zone=home --remove-service=http
```

> 注：上面的配置都为临时配置，在重启之后均会失效，如果需要永久配置，则添加选项参数 --permanent。

### 查看当前防火墙运行状态

```bash
$ firewall-cmd --state
running       == 正在运行
     
$ firewall-cmd --state
not running   == 防火墙已关闭
```

### 查看当前防火墙的配置

```bash
$ firewall-cmd --list-all
public (default, active)
  interfaces: eth0 eth1
  sources: 
  services: dhcpv6-client ssh
  ports: 5100/tcp 18080/tcp 8080/tcp
  masquerade: no
  forward-ports: 
  icmp-blocks: 
  rich rules:
```
