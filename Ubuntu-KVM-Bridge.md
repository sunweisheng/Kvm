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

如果你打算从本机之外访问KVM虚拟机，必须搭建网桥（Bridge），通过在 /etc/netplan目录下创建文件01-netcfg.yaml来新建网桥。

````shell
sudo vi /etc/netplan/01-netcfg.yaml
````

````yaml
network:
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
  # add configuration for bridge interface
  bridges:
    #virbr0网桥是KVM安装完成后自动创建
    br0:
      #enp0s3是物理网卡
      interfaces: [enp0s3]
      dhcp4: false
      #虚拟机IP
      addresses: [192.168.1.162/24]
      macaddress: 08:00:27:4b:1d:45
      routes:
        - to: default
          #网关
          via: 192.168.1.1
          metric: 100
      nameservers:
        #DNS服务器
        addresses: [4.2.2.2]
      parameters:
        stp: false
      dhcp6: false
  version: 2
````

````shell
#变更生效
sudo netplan apply
#验证生效
sudo ip add show
````
