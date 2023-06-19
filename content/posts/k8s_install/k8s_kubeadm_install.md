---
weight: 5
title: "k8s安装基于kubeadm安装版本"
draft: false
date: 2023-06-19T23:05:11+08:00
tags: ["k8s", "容器"]
categories: ["容器"]   # 文档类型

twemoji: false
lightgallery: true
---
本文使用kubeadm方式来快速部署k8s集群，操作系统基于ubuntu操作系统
<!--more-->

### 基础配置
#### IP网段规划
| IP | 描述 |
| ------ | ----------- |
| 192.168.50.31   | master节点 |
| 192.168.50.32   | node01节点 |
| 192.168.50.33   | node02节点 |
> 请注意IP地址规划

#### 操作系统版本
> Ubuntu 22.04 LTS amd64版本
### 架构设计


去露营啦! :tent: 很快就回来.

真开心! :joy:
```go
import (
  "fmt"
)
func main() {
  fmt.Println("test")
}
```