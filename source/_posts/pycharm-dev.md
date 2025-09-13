---
title: Pycharm远程开发折腾日记
mathjax: true
date: 2025/9/13
category: 技术学习与分享
---

# Pycharm远程开发折腾日记

这里主要记录一下用Pycharm做远程开发时踩的坑

## 远程ssh解释器

### 使用小众shell导致的conda环境无法识别
组里的服务器默认用的`tcsh`，但是我平时都是ssh上去之后用zsh开发；没有root权限，不能用`chsh`改shell
怎么发现这个问题的呢……配置编译器时不管是选系统解释器还是Conda环境都无法自动识别，手动选Conda的binary也会报错
看到系统解释器里那一行`Unknown Option`之后，我意识到Pycharm配置远程编译器的时候大概率执行了类似`tcsh -l`这样的命令（`zsh`是有`-l`参数的），最终导致输出不符合预期而报错
![tcsh报错](attachments/pycharm-dev-1.png)

用23版的Pycharm可以看到连接时的日志，也确实如此：
![alt text](attachments/pycharm-dev-2.png)

目前还在找解决办法……