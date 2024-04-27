
---
title: HUST CSE 机器人与人工智能安全课程  Q&A
date: 2024/4/27
category: ROS相关
---
# No.1
## Q
```bash
RLException: [turtlebot3_remote.launch] is neither a launch file in package [turtlebot3_bringup] nor is [turtlebot3_bringup] a launch file name
The traceback for the exception was written to the log file
```

## A
修改`~/.bashrc`，确保两个`setup.bash`是如下顺序并且没有重复source：
```bash
source /opt/ros/noetic/setup.bash
source ~/catkin_ws/devel/setup.bash
```

# No.2
## Q
ubuntu的网络设置中没有有线连接这个选项
## A
用`nmcli`重启网络服务
```bash
sudo nmcli networking off 
sudo nmcli networking on
```

# No.3
## Q
网络激活失败
## A
1. 首先确认自己连接的不是校园网而是实验室的路由器
2. 然后检查虚拟机设置->桥接模式中没有勾选“复制物理网络连接状态”
3. 检查编辑->虚拟网络编辑器->更改设置（需要管理员权限）中是否有VMNet0，以及桥接模式连接的是不是自己的无线网卡
![](http://chev.n2ptr.space/images/2024/04/26/b0c0096d57dc82c1e4d069a1829e9fd8.png)