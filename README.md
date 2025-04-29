# LinuxGuide
## CentOS 初始化

``` 
###  ping -c 4 www.baidu.com
```

作用：检查网络连通性（ICMP 协议是否畅通）。

结果分析：

如果有类似 64 bytes from 220.181.38.150 的回复，说明网络通畅。

如果显示 Unknown host 或超时，可能是 DNS 解析失败或网络阻断。

### 如何设置静态IP
#### 步骤 1：确定网卡名称
ip addr 或 nmcli device status
找到正在使用的网卡（如 ens33、eth0 等）。

#### 步骤 2：修改网络配置文件
``` 
cd /etc/sysconfig/network-scripts/
``` 

找到对应网卡的配置文件（如 ifcfg-ens33）
``` 
sudo vi ifcfg-ens33
```

修改或添加以下内容（根据你的网络环境调整）：

``` 
BOOTPROTO=static        设置为静态IP

ONBOOT=yes             开机自动启用网卡

IPADDR=192.168.1.100    静态IP地址

NETMASK=255.255.255.0   子网掩码

GATEWAY=192.168.1.1    默认网关

DNS1=8.8.8.8           主DNS

DNS2=8.8.4.4           备用DNS
```

或者使用交互式工具来修改网络：
``` 
sudo nmtui
```
选择 Edit a connection → 选择网卡 → 手动配置 IP、网关、DNS。

重启网络服务
```
sudo systemctl restart network
```
### 备份与恢复
#### 备份原配置：
``` 
sudo cp /etc/sysconfig/network-scripts/ifcfg-ens33 /root/ifcfg-ens33.bak
```
#### 恢复配置：
``` 
sudo mv /root/ifcfg-ens33.bak /etc/sysconfig/network-scripts/ifcfg-ens33
sudo systemctl restart network
```
### 查看内存信息
#### 使用 free 命令（推荐）
``` 
free -h
```
### 查看磁盘信息
#### 使用 df 命令（查看磁盘空间）
``` 
df -h
```
#### 使用 lsblk 命令（查看磁盘设备）
``` 
lsblk
```
#### 使用 fdisk 查看详细分区
``` 
sudo fdisk -l
```
### 常见问题
1. free 显示内存几乎耗尽？

Linux 会主动利用空闲内存作缓存（buff/cache），实际可用内存看 available 列。

2. 如何查看具体目录占用空间？

``` 
du -sh /var/log  # 查看 /var/log 目录大小
```
3. 磁盘显示 100% 但文件不多？

可能是小文件占满 inode，用 df -i 检查 inode 使用率。

### 初始化
#### (1) 更新系统所有软件包
``` 
sudo yum update -y
```
#### (2) 安装常用工具
``` 
sudo yum install -y epel-release  # 启用 EPEL 仓库（额外软件包）
sudo yum install -y wget curl vim net-tools lsof htop tmux git unzip
```
wget / curl：下载文件

vim：增强版文本编辑器

net-tools：ifconfig、netstat 等网络工具

htop：交互式进程查看器

tmux：终端多窗口管理

#### (3) 磁盘挂载（如果新增了数据盘）
(1) 查看当前磁盘情况
``` 
lsblk  # 查看磁盘及分区情况
sudo fdisk -l  # 查看详细磁盘信息
```
找到未挂载的磁盘（如 /dev/sdb）
(2) 分区并格式化（以 /dev/sdb 为例）
``` 
sudo fdisk /dev/sdb  # 进入分区工具
```

按 n → p（主分区）→ 1（分区号）→ 回车（默认全部空间）→ w 保存。
格式化分区（如 ext4）：
``` 
sudo mkfs.ext4 /dev/sdb1
```

### 如何操作剩余空间？
选项 1：扩展 LVM（推荐）
bash
# 创建新分区（使用 fdisk/gdisk）
sudo fdisk /dev/sda
# 在交互界面中：n → 新建分区（如 sda3），t → 类型改为 8e（Linux LVM），w 保存退出。

# 重读分区表
sudo partprobe /dev/sda

# 初始化新分区为物理卷（PV）
sudo pvcreate /dev/sda3

# 扩展卷组（VG）centos
sudo vgextend centos /dev/sda3

# 扩展逻辑卷（LV）centos-root
sudo lvextend -l +100%FREE /dev/mapper/centos-root

# 调整文件系统大小（根据文件系统类型选择命令）
sudo xfs_growfs /      # 如果是 XFS 文件系统（CentOS7 默认）
# 或
sudo resize2fs /dev/mapper/centos-root  # 如果是 ext4
选项 2：创建普通分区并挂载
bash
# 创建新分区（如 sda3），格式化为 ext4
sudo mkfs.ext4 /dev/sda3

# 创建挂载点并挂载
sudo mkdir /data
sudo mount /dev/sda3 /data

# 开机自动挂载
echo "/dev/sda3 /data ext4 defaults 0 0" | sudo tee -a /etc/fstab
4. 验证操作
使用 df -h 查看挂载情况。

使用 vgs 或 lvs 查看 LVM 状态。

总结：
如果不需要额外存储空间：无需操作，当前配置已满足系统运行。

如果需要使用剩余 85G：建议扩展 LVM 的 centos-root 或创建新分区挂载到其他目录（如 /data）。
