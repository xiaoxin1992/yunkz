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
### 环境准备
#### 增加hosts解析记录
分别3台机器配置hosts解析记录
```shell
192.168.50.31    master01.k8s.com
192.168.50.31    node01.k8s.com
192.168.50.32    node02.k8s.com
```
#### 配置主机名
- master01机器配置(192.168.50.31)
```shell
~$ hostnamectl set-hostname master01.k8s.com
```
- node01机器配置(192.168.50.31)
```shell
~$ hostnamectl set-hostname node01.k8s.com
```
- node02机器配置(192.168.50.32)
```shell
~$ hostnamectl set-hostname node02.k8s.com
```

#### 禁止所有机器的防火墙
```shell
~$ systemctl disable --now ufw.service
```

#### 所有机器启用IPV4内核转发模块
```shell
~$ modprobe br_netfilter
```
#### 所有机器配置ulimit参数
```shell
~$ ulimit -SHn 65535
```
##### 永久配置
```
~$ cat /etc/security/limits.conf
	soft nofile 65535
	hard nofile 65535
	soft nproc 65535
	hard nproc 65535
```

#### 所有机器内核参数配置
```shell
~$ cat /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max=655535
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
~$ sysctl --system
```
#### 所有机器IPVS模块安装
```shell
~$ modprobe -- ip_vs
~$ modprobe -- ip_vs_rr
~$ modprobe -- ip_vs_wrr
~$ modprobe -- ip_vs_sh
~$ modprobe -- nf_conntrack
~$ cat /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
```
#### 所有机器检查模块是否加载
```shell
~$ lsmod |grep -e ip_vs -e nf_conntrack
~$ apt install ipset # 安装ipset, ipvsadm 管理工具
```
#### 所有机器同步服务器时间
```shell
~$ ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime # 配置时区
~$ ntpdate time.windows.com
```
#### 所有机器关闭swap
```shell
~$ swapoff  -a
~$ cat /etc/fstab
# 注释swap分区
#/swap.img	none	swap	sw	0	0
```

### 容器运行时安装
#### 所有机器依赖包安装
```shell
~$ apt-get install -y libseccomp2
```

#### 所有机器containerd下载部署
```shell
~$ wget https://github.com/containerd/containerd/releases/download/v1.6.18/cri-containerd-cni-1.6.18-linux-amd64.tar.gz
~$ tar -tf cri-containerd-cni-1.6.18-linux-amd64.tar.gz # 查看压缩包包含哪些文件
~$ tar zxf cri-containerd-cni-1.6.18-linux-amd64.tar.gz -C / # 解压包到跟路径
~$ export PATH=$PATH:/opt/cin/bin # 增加环境变量
```
#### 所有机器配置文件生成
```shell
~$ mkdir -p /etc/containerd
~$ containerd config default > /etc/containerd/config.toml
```
#### 所有机器修改配置文件
> 配置文件路径`/etc/containerd/config.toml`
```yaml

root = "/var/lib/containerd" # 保存持久化数据目录
....
sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
....
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
....
	SystemdCgroup = true
....
[plugins."io.containerd.grpc.v1.cri".registry] # 镜像加速配置
....
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://bqr1dr1n.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://registry.aliyuncs.com/k8sxio"]
```
#### 所有机器启动containerd服务
```shell
~$ systemctl  enable --now containerd
```
### 开始集群安装
#### 所有机器kubeadm安装
```shell
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet=1.23.13-00 kubeadm=1.23.13-00 kubectl=1.23.13-00
sudo apt-mark hold kubelet=1.23.13-00 kubeadm=1.23.13-00 kubectl=1.23.13-00 # 锁定版本
```
#### 所有机启动kubelet服务
```shell
~$ systemctl enable --now kubelet
```
#### master节点初始化集群配置文件
```shell
~$ kubeadm config print init-defaults --component-configs KubeletConfiguration >/root/kubeadm.yaml
```
#### master节点修改配置文件
> 配置文件路径 `/root/kubeadm.yaml`
```yaml
.....
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.50.31 # 替换成master地址
	bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock   # 修改为containerd的sock文件
  imagePullPolicy: IfNotPresent
  name: master01.k8s.com                         # 注册的master名称
  taints:                                      # 污点防止调度到master节点
    - effect: "NoSchedule"
      key: "node-role.kubernetes.io/master"
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1     # 增加kube-proxy模式配置
kind: KubeProxyConfiguration
mode: ipvs   # kube-proxy 模式
.... 
imageRepository: registry.aliyuncs.com/k8sxio     # 修改镜像获取地址为国内地址
kind: ClusterConfiguration
kubernetesVersion: 1.23.13                        # 获取k8s版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16  # 指定pod子网
....
```
#### 初始化master节点
```shell
# 获取镜像
~$ kubeadm  config images pull --config /root/kubeadm.yaml
# 手动拉取coredns镜像，阿里云没有这个镜像
~$ ctr -n k8s.io image pull docker.io/coredns/coredns:1.8.6
~$ ctr -n k8s.io image tag docker.io/coredns/coredns:1.8.6 registry.aliyuncs.com/k8sxio/coredns:v1.8.6
# 初始化master节点, 根据提示可加入node节点
～$ kubeadm  init -v 5 --config /root/kubeadm.yaml
```
> 初始化master节点后，根据kubeadm join提示来完成node节点的加入
> 如果忘记join节点token可以通过命令重新获取新的token`kubeadm token create --print-join-command`

#### 网络插件安装
> calico文件下载[github地址](https://github.com/projectcalico/calico/blob/v3.25.1/manifests/calico.yaml)
```shell
~$ kubectl apply -f calico.yaml
# 移除containerd的网络配置，放置影响pod分配的IP地址
~$ mv /etc/cni/net.d/10-containerd-net.conflist  /etc/cni/net.d/10-containerd-net.conflist.old
```


#### 清理集群
>  执行`kubeadm rest`以后需要在对应的节点执行清理，次命令可以在所有节点执行
```shell
~$ kubeadm rest
~$ ifconfig cni0 down && ip link delete cni0
~$ rm -rf /var/lib/cni/
```
