+++
title = 'Ubuntu NAT 流量转发原理'
date = 2026-10-13T16:00:00+08:00
draft = false
tags = ['Ubuntu', 'Linux', 'NAT']
categories = ['网络技术']
+++

## 一、NAT 是什么？

NAT（Network Address Translation）是网络地址转换技术。

它可以让内网设备通过一个公网地址访问外部网络。

## 二、常见 NAT 类型

- SNAT：修改数据包的源 IP 地址
- DNAT：修改数据包的目标 IP 地址
- MASQUERADE：适用于出口地址动态变化的场景

## 三、排查命令

查看路由：

```bash
ip route
