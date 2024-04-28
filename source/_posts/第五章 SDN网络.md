---
title: 计网-SDN网络
mathjax: true
date: 2024/4/17
tags:
	- 计网
category: 课堂笔记
---

# 控制平面
传统的路由器应该包含的两项功能：
- 路由功能
- 数据转发功能
计算生成转发表的过程是分布式的


SDN的解决方案：
路由控制功能从本地路由器分离，汇聚到远程控制器，与路由器的本地控制代理（CAs）进行交互，以计算转发表；转发表的计算对于由单一远程控制器控制的路由器来说是集中的

![](attachments/Pasted%20image%2020231110161342.png)


# 数据平面

都由路由器实现
- 传统：基于目标地址 + 转发表
- SDN：基于多个字段+流表 #TBS 


# SDN架构

![](attachments/Pasted%20image%2020231110161803.png)






# OpenFlow协议
相比于IP协议有较大的改动

![](attachments/Pasted%20image%2020231110162736.png)

