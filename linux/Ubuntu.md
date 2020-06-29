---
typora-root-url: img
typora-copy-images-to: img
---

# Ubuntu

## 一、 安装系统，重启进入grub界面

> 该问题，应该是启动引导问题，和grub相关。解决思路，在grub界面进行找到linux的对应本地磁盘，设置系统的root的路径。

```
1. 先使用ls命令，找到Ubuntu的安装在哪个分区：
 grub>ls

会罗列所有的磁盘分区信息，比方说：

(hd0,1),(hd0,5),(hd0,3),(hd0,2)

2. 然后依次调用如下命令： X表示各个分区号码
如果/boot没有单独分区，用以下命令：
ls (hd0,X)/boot/grub

如果/boot单独分区，则用下列命令：

ls （hd0,X)/grub

正常情况下，会列出来几百个文件，很多文件的扩展名是.mod和.lst和.img，还有一个文件是grub.cfg。假设找到（hd0,5）时，显示了文件夹中的文件，则表示Linux安装在这个分区。

3，如果找到了正确的grub目录，则设法临时性将grub的两部分关联起来，方法如下：

grub>set root=(hd0,5)
grub>set prefix=(hd0,5)/boot/grub
然后调用如下命令，就可以显示出丢失的grub菜单了。

grub>normal
然后会出来启动的图形界面，点击进入Linux中，对grub进行修复。
进入ubuntu之后，在终端执行：
sudo update-grub

sudo grub-install /dev/sda
（sda是你的硬盘号码，千万不要指定分区号码，例如sda1，sda5等都不对）
重启测试是否已经恢复了grub的启动菜单。
```

二、 ubuntu修改网络

1. 查看网关 

   `netstat -rn or route -n`

2. 修改网络配置文件

​      `sudo vim /etc/netplan/01-network-manager-all.yaml `

```yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
  ethernets:
          enp0s31f6:
                  addresses:
                  - 10.4.176.24/24
                  dhcp4: false
                  gateway4: 10.4.176.1
                  nameservers:
                          addresses:
                          - 114.114.114.114
```

3.常用命令

```shell
#打开资源管理器
nautilus
```

