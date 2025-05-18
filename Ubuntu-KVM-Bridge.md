# Ubuntu下用KVM搭建Bridge网络

## 准备工作

* CPU是否开启了虚拟化指令集（intel VT-x或AMD-V）

````shell
sudo egrep -c '(vmx|svm)' /proc/cpuinfo
````
如果返回不是0，则说明CPU开启了intel VT-x或AMD-V。

* 更新软件包列表和升级所有可更新的软件包

````shell
#更新软件包
sudo apt update
sudo apt upgrade
````
## Ubuntu下安装KVM

````shell
#安装KVM
sudo apt install -y qemu-kvm virt-manager libvirt-daemon-system virtinst libvirt-clients bridge-utils
````

* qemu-kvm – 一个提供硬件仿真的开源仿真器和虚拟化包
* virt-manager – 一款通过 libvirt 守护进程，基于 QT 的图形界面的虚拟机管理工具
* libvirt-daemon-system – 为运行 libvirt 进程提供必要配置文件的工具
* virtinst – 一套为置备和修改虚拟机提供的命令行工具
* libvirt-clients – 一组客户端的库和API，用于从命令行管理和控制虚拟机和管理程序
* bridge-utils – 一套用于创建和管理桥接设备的工具

````shell
#启用虚拟化守护进程（libvirtd）
sudo systemctl enable --now libvirtd
sudo systemctl start libvirtd

#查看结果
sudo systemctl status libvirtd
````

````shell
#当前登录用户加入kvm和libvirt用户组
sudo usermod -aG kvm $USER
sudo usermod -aG libvirt $USER
````
只有加入到kvm和libvirt用户组才能管理KVM虚拟机，加入之后需要注销再登录一次才能挂上新分配的权限，如果是链接远程桌面用户则可能需要重启电脑。

````shell
#在远程桌面下可以执行命令打开KVM管理界面
virt-manager
````

## 创建Bridge网络

如果你打算从本机之外访问KVM虚拟机，必须搭建网桥（Bridge），通过在 /etc/netplan目录下修改配置文件。

````shell
sudo vi /etc/netplan/01-network-manager-all.yaml
````

````yaml
network:
  ethernets:
    #物理网卡
    enp2s0:
      #物理网卡不设置IP地址，仅作为链路层设备，网络层IP、网关、DNS都交给网桥负责
      dhcp4: false
      dhcp6: false
  # add configuration for bridge interface
  bridges:
    #virbr0网桥是KVM安装完成后自动创建
    virbr0:
      #enp2s0是物理网卡
      interfaces: [enp2s0]
      dhcp4: false
      #网桥IP
      addresses: [192.168.0.5/24]
      macaddress: 10:7b:44:81:88:ce
      routes:
        - to: default
          #网关
          via: 192.168.0.1
          metric: 100
      nameservers:
        #DNS服务器
        addresses: [192.168.0.1]
      parameters:
        stp: true
      dhcp6: false
  version: 2
````

````shell
#变更生效
sudo netplan apply

#如果显示Permissions for /etc/netplan/01-network-manager-all.yaml are too open. Netplan configuration should NOT be accessible by others.
#说明文件权限太宽松
sudo chmod 600 /etc/netplan/01-network-manager-all.yaml

#在执行变更生效
sudo netplan apply

#验证生效
sudo ip add show
````
名为virbr0的网桥IP已经改为192.168.0.5，物理网卡enp2s0接入网桥并没有IP地址了（网卡只负责数据发送和接收）。

## 安装虚拟机

远程桌面登录宿主机打开virt-manager管理软件。
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-01.png)

点击“文件”并选择“新建虚拟机”
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-02.png)

选择本地ISO安装
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-03.png)

设置CPU和内存
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-04.png)

设置硬盘
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-05.png)

设置网络,虚拟机网关必须配置为真实物理网关，否则HTTPS会出问题。
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-06.png)

开始安装操作系统
![Alt text](http://static.bluersw.com/images/Kvm/U-KVM-B-07.png)

# 安装完成之后查看虚拟机

````shell
#查看虚拟机列表
virsh list

#关闭虚拟机
virsh shutdown k8s-master

#查看虚拟机信息 40G
sudo qemu-img info k8s-master.qcow2

#制作基础盘并收缩大小
sudo qemu-img convert -O qcow2 k8s-master.qcow2 ubuntu-system.qcow2

#查看转化后的虚拟机信息 4.19G
sudo qemu-img info ubuntu-system.qcow2

#删除原有文件
sudo rm k8s-master.qcow2

#从基础盘复制一份
sudo cp ubuntu-system.qcow2 k8s-master.qcow2

#启动虚拟机
virsh start k8s-master
````

