## 一、安装KVM虚拟机
#### 1、查看机器CPU是否支持虚拟化

    # egrep "(vmx|svm)" /proc/cpuinfo  

#### 2、加载kvm内核模块
	# modprobe kvm
#### 3、加载intel内核模块（如果cpu为AMD则加载kvm-amd）
	# modprobe kvm-intel   
#### 4、安装虚拟机管理软件
	# yum install qemu-kvm libvirt virt-manager bridge-utils libguestfs-tools   
#### 5、启动libvirtd服务，此服务会自动创建一个virbr0桥设备
	# service libvirtd start    
#### 6、查看网桥设备命令
	# brctl show   
#### 7、创建桥接设备，将eth0网卡桥接到br0上
	# virsh iface-bridge eth0 br0     
#### 8、查看kvm是否安装成功
	# virsh list    
#### 9、启用qemu自带的vnc
	# vim /etc/libvirt/qemu.conf
	vnc_listen = "0.0.0.0"
## 二、配置基础环境
    # service iptables stop
    # mkdir /iso
    # mkdir -p /kvm/imge
## 三、创建虚拟机
    # qemu-img create -f qcow2 /kvm/imge/CentOS6.7.qcow2 20G
    # qemu-img info /kvm/imge/CentOS6.7.qcow2
    # virt-install --name=CentOS6.7 --ram 1024 --vcpus 2 --disk path=/kvm/imge/CentOS6.7.qcow2,size=20,format=qcow2,bus=virtio,sparse --cdrom /iso/CentOS-6.7-x86_64-bin-DVD1.iso --graphic vnc,listen=0.0.0.0,port=5910 --os-type=linux --virt-type=kvm --hvm --accelerate --network bridge=br0 --autostart
## 四、安装操作系统
### 使用“TightVNC Viewer”连接安装系统
## 五、常用的KVM操作命令
    # virsh list --all列出所有虚拟机（包括关机的）
    # virsh list列出正在运行的虚拟机
    # virsh start CentOS6.7   启动虚拟机
    # virsh define /etc/libvirt/qemu/CentOS6.7.xml   定义虚拟机，导入虚拟机
## 六、操作示例：
    # virsh attach-interface CentOS6.7 --type bridge --source br1   添加网卡（不会修改xml文件，重启失效）
    # virsh dumpxml CentOS6.7 > /etc/libvirt/qemu/CentOS6.7.xml 保存修改到xml文件
## 七、KVM虚拟创建示例：
    # qemu-img create -f qcow2 /data/kvmdisk/Iknow-Client.qcow2 240G
    # virt-install --name=Iknow-Server --ram 24576 --vcpus 12 --disk path=/data/kvmdisk/Iknow-Server.qcow2,size=240,format=qcow2,bus=virtio,sparse --cdrom /iso/CentOS-6.8-x86_64-bin-DVD1.iso --graphic vnc,listen=0.0.0.0,port=5901 --os-type=linux --virt-type=kvm --hvm --accelerate --network bridge=br0 --autostart