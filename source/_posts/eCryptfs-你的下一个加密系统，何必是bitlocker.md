---
title: eCryptfs-你的下一个加密系统，何必是bitlocker
mathjax: true
date: 2024/4/17
tags:
    - 理论相关
    - 实用工具
category: 技术学习与分享
---




# 什么是eCryptfs
eCryptfs 是 Linux 内核原⽣的⼀种堆叠式加密⽂件系统（stacked cryptographic filesystem）。堆叠式⽂件系统是指以现有挂载的⽂件系统（称为下层⽂件系统-lower filesystems）为基础，在其之上构建新的⽂件系统。eCryptfs 是⼀种堆叠式⽂件系统，它在 ⽂件被写⼊或从下层⽂件系统读取时对其进⾏加密和解密操作。

# eCryptfs的架构设计

![](attachments/Pasted%20image%2020231230151223.png)
eCryptfs是夹在底层⽂件系统和VFS(Virtual Filesystem Switch)之间的⼀层加密功能，并且 属于Kernel的⼀部分.当应⽤程序尝试读写加密的⽬录的时候，eCryptfs通过与Kernel的 Crypto API、Keyring以及Userspace⾥⾯的eCryptfs daemon的通信进⾏加密和解密操作。 其中Userspace的eCryptfs daemon和Kernel的Keyring共同负责秘钥的管理，⽽Crypto API 则负责使⽤密钥对数据进⾏加密和解密

# eCryptfs的加密解密流程
![](attachments/Pasted%20image%2020231230151309.png)

## 关于密钥

eCryptfs的加密解密涉及到使⽤“有层次结构”的密钥（ a hierarchy of keys ），在认证流程中涉及到三种密钥

*FEK(File Encryption Key)* 
由系统⽣成，⽤来加密下层⽂件系统的⽂件
*FEKEK(File Encryption Key Encryption Key)* 
由⽤户指定，⽤来加密FEK
*EFEK(Encrypted File Encryption Key)* 
FEK经过FEKEK加密之后⽣成的Key，保存在⽂件系统中

## 关于加密⽅式
eCryptfs默认采⽤AES加密，这是⼀种分块加密算法，在上图中加密之后的分块对应的是 Encrypted Data Extent；在加密⽂件的头部存储了加密⽂件的信息（包含EFEK），其对应的解密后⽂件也遵循相同的结构

## 加密/解密流程

*Step1*:⽤户提供FEKEK，系统⽤这个密钥解密储存在Header⾥⾯的EFEK，得到FEK 

*Step2*:eCryptfs使⽤这个密钥通过Crypto API解密在底层⽂件系统的加密⽂件，将其保存到 eCryptfs File⾥⾯⼀个叫做Page Cache的地⽅ 

*Step3*:⽤户可以直接像访问没有加密的⽬录⼀样访问eCryptfs 如果要加密的话，流程则正好相反

# 简单的eCryptfs使用


## eCryptfs Utils

Ubuntu官⽅使⽤的加密⽅法，⼀般⽤来加密~/home这个⽬录 
在安装ubuntu的时候有个加密主⽬录的选项，就是使⽤这个⼯具加密的，默认底层⽬录始 终是~/.Private/ 
挂载和解挂载命令如下： 
```bash
ecryptfs-mount-private
ecryptfs-umount-private
```

⽤户可以在shell⾥⾯通过`ecryptfs-setup-private` ⼯具设置开机的时候是否⾃动挂载加密的⽬录


## ecryptfs-simple

⽐eCryptfs Utils⾃由度更加⾼⼀点的eCryptfs管理⼯具，允许⽤户指定底层⽬录(需要另外安装) 

挂载：
```bash
⽐eCryptfs Utils⾃由度更加⾼⼀点的eCryptfs管理⼯具，允许⽤户指定底层⽬录(需要另外 安装) 挂载：
```

自动挂载
```bash
ecryptfs-simple -a /path/to/foo /path/to/bar
```

解挂载：既可以输⼊底层⽬录的路径，也可以输⼊挂载点的路径
```bash
ecryptfs-simple -u /path/to/foo #底层⽬录路径 
ecryptfs-simple -u /path/to/bar #挂载点路径
```

## 手动加密和解密

eCryptfs使⽤mount和umount命令来挂载和解挂载⽬录
```bash
sudo mount -t ecryptfs /secret /secret #挂载⽬录 
sudo umount -t ecryptfs /secret #解挂载⽬录
```

第⼀个secret是底层⽂件系统的地址，第⼆个secret则是eCryptfs的挂载点。这两个地址的 名称可以不同，但是建议采⽤相同的名称来实现⼀个叫Layover Mount的东⻄（⼤概的意思 就是挂载之后就看不到原来的底层⽂件夹了？）

在挂载期间，我们对/secret的访问实际上是对eCryptfs⾥⾯的/secret做访问，读写数据也是 在eCryptfs中进⾏

⽽解挂载之后，我们就会看到加密之后的⽂件，通过常规⼿段⾃然是⽆法读取的

和前两个管理⼯具的区别似乎就是可以指定⼀些⽐较详细的参数，⽐如加密⽅式之类的

设置⾃动挂载⽅案以及加密算法的⽅法⻅[Arch Wiki：eCryptfs](设置⾃动挂载⽅案以及加密算法的⽅法⻅Arch Wiki：eCryptfs)



# eCryptfs的优缺点

## 优点 
不需要单独开辟加密的分区，加密的单位不是驱动器⽽是⽬录 

底层⽂件系统的⽂件可以共享给其他⽤户，因为⽂件头部包含EFEK，只要有对⽅有合适的 FEKEK就能解密，灵活性⾼

## 缺点
性能问题：eCryptfs对读操作性能影响较⼩，对写操作性能影响较⼤。这是因为其在读取⽂件的时候可以反复访问存储在Page Cache⾥⾯的解密后⽂件，但是每次写⼊都必须进⾏⼀ 次加密操作

安全问题：eCryptfs的Page Cache⾥⾯存储的是明⽂，它是通过权限的设置来保证不被其他 应⽤程序读取的。如果权限设置不当，这些数据可能会暴露给未经授权的应⽤程序

# 参考

https://www.linuxjournal.com/article/9400 
https://wiki.archlinux.org/title/ECryptfs 
https://zhuanlan.zhihu.com/p/539350620
