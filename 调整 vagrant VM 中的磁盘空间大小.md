# 调整 vagrant VM 中的磁盘空间大小

> 原文地址：[Resize a Hard Disk for a Virtual Machine](https://gist.github.com/christopher-hopper/9755310)

## 0x01 确定调整前 storage 大小

调整前

```
➜  ~ cd workspace/vagrant_playground/ubuntu-17.04-server-cloudimg-amd64-vagrant
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant vagrant up
...
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant vagrant ssh
...
ubuntu@ubuntu-1704:~$ sudo -i
[#1#root@ubuntu-1704 ~]$
[#1#root@ubuntu-1704 ~]$df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            486M     0  486M   0% /dev
tmpfs            99M  3.3M   96M   4% /run
/dev/sda1       9.7G  6.7G  3.1G  69% /            -- here
tmpfs           495M     0  495M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           495M     0  495M   0% /sys/fs/cgroup
overlay         9.7G  6.7G  3.1G  69% /var/lib/docker/overlay2/74a42e340f304ee77b1a2de744d15072fd030be82a3b1e49228bd96032a75670/merged
overlay         9.7G  6.7G  3.1G  69% /var/lib/docker/overlay2/2ca55d1cd2029b74bea2a4e90d828dcabc74321a7dd35a7c9e1afbe8131c5eaa/merged
shm              64M     0   64M   0% /var/lib/docker/containers/5c9a2700c33c8058d32ab775cab290f3eadd4baed32291c99504adc9bbe62cc9/shm
shm              64M     0   64M   0% /var/lib/docker/containers/d2487450b4e8c5678a9e67d22e2be9bfe8e56a0ef5222e5bb2fdb99e4d075b35/shm
vagrant         234G   73G  161G  32% /vagrant
vagrant_data    234G   73G  161G  32% /vagrant_data
tmpfs            99M     0   99M   0% /run/user/1000
[#2#root@ubuntu-1704 ~]$
```

## 0x02 Stop the virtual machine using Vagrant.


```
➜  ~ cd workspace/vagrant_playground/ubuntu-17.04-server-cloudimg-amd64-vagrant
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant vagrant halt
==> default: Attempting graceful shutdown of VM...
```

## 0x03 Locate the VirtuaBox VM and the HDD attached to its SATA (or SCSI) Controller. 

```
➜  ~ ll
total 0
drwx------@  5 sunfei  staff   160B 10 18 14:53 Applications
drwx------@ 22 sunfei  staff   704B 11 30 22:52 Desktop
drwx------@  6 sunfei  staff   192B 10 31 16:32 Documents
drwx------+ 29 sunfei  staff   928B 12  1 00:26 Downloads
drwx------@ 64 sunfei  staff   2.0K 11 17 11:43 Library
drwx------+  3 sunfei  staff    96B 10 16 11:26 Movies
drwx------+  4 sunfei  staff   128B 11 23 23:09 Music
drwx------+  3 sunfei  staff    96B 10 16 11:26 Pictures
drwxr-xr-x+  5 sunfei  staff   160B 10 16 11:26 Public
drwx------   7 sunfei  staff   224B 12  5 16:12 VirtualBox VMs
drwxr-xr-x   5 sunfei  staff   160B 10 16 13:10 go
drwxr-xr-x   8 sunfei  staff   256B 11  7 15:04 workspace
➜  ~ 
➜  ~ cd VirtualBox\ VMs/ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
➜  ~ 
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932

➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 ll
total 15132432
drwx------  6 sunfei  staff   192B 12  5 16:03 Logs
-rw-------  1 sunfei  staff   3.8K 12  5 16:14 ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932.vbox
-rw-------  1 sunfei  staff   3.8K 12  5 16:03 ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932.vbox-prev
-rw-------  1 sunfei  staff   192K 12  5 16:14 ubuntu-zesty-17.04-cloudimg-configdrive.vmdk
-rw-------  1 sunfei  staff   7.2G 12  5 16:14 ubuntu-zesty-17.04-cloudimg.vmdk
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932

➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage showvminfo ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 | grep ".vmdk"
SCSI (0, 0): /Users/sunfei/VirtualBox VMs/ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932/ubuntu-zesty-17.04-cloudimg.vmdk (UUID: 81842dee-06b6-4453-a1d9-23d9c0d40e0d)
SCSI (1, 0): /Users/sunfei/VirtualBox VMs/ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932/ubuntu-zesty-17.04-cloudimg-configdrive.vmdk (UUID: c13d6cfa-e446-40dd-b1fc-da3451811254)
```

The `showvminfo` command should show you the location on the file-system of the HDD of type VMDK along with the name of the Controller it is attached to


## 0x04 clone the VMDK type disk to a VDI type disk so it can be resized.

> NOTE: We do this because VMDK type disks cannot be resized by VirtualBox. It has the added benefit of allowing us to **keep our original disk backed-up during the resize operation**.

```
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage clonehd "ubuntu-zesty-17.04-cloudimg.vmdk" "clone-ubuntu-zesty-17.04-cloudimg.vdi" --format vdi
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Clone medium created in format 'vdi'. UUID: 82bdf361-e1e9-4b5e-98fd-ca56f8c8db4d

➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 ll
total 30089064
drwx------  6 sunfei  staff   192B 12  5 16:03 Logs
-rw-------  1 sunfei  staff   7.1G 12  5 16:31 clone-ubuntu-zesty-17.04-cloudimg.vdi
-rw-------  1 sunfei  staff   3.8K 12  5 16:31 ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932.vbox
-rw-------  1 sunfei  staff   3.8K 12  5 16:14 ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932.vbox-prev
-rw-------  1 sunfei  staff   192K 12  5 16:14 ubuntu-zesty-17.04-cloudimg-configdrive.vmdk
-rw-------  1 sunfei  staff   7.2G 12  5 16:14 ubuntu-zesty-17.04-cloudimg.vmdk
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
```

## 0x05 Find out how big the disk is currently, to determine how large to make it when resized. 

The information will show the **current size** and the `Format variant`. If **Dynamic Allocation** was used to create the disk, the Format variant will be "`dynamic default`".


```
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage showhdinfo clone-ubuntu-zesty-17.04-cloudimg.vdi
UUID:           82bdf361-e1e9-4b5e-98fd-ca56f8c8db4d
Parent UUID:    base
State:          created
Type:           normal (base)
Location:       /Users/sunfei/VirtualBox VMs/ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932/clone-ubuntu-zesty-17.04-cloudimg.vdi
Storage format: vdi
Format variant: dynamic default
Capacity:       10240 MBytes
Size on disk:   7305 MBytes
Encryption:     disabled
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
```


## 0x06 Resize the cloned disk to give it more space. 

The size argument below is given in Megabytes (1024 Bytes = 1 Megabyte). Because this disk was created using dynamic allocation I'm going to resize it to 200 Gigabytes.

```
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage modifyhd "clone-ubuntu-zesty-17.04-cloudimg.vdi" --resize 204800
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
```

> NOTE: If the disk was created using dynamic allocation (see previous step) then the physical size of the disk will not need to match its logical size - meaning you can create a very large logical disk that will increase in physical size only as space is used.


## 0x07  Find out the name of the Storage Controller to attach the newly resized disk to.


```
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage showvminfo ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 | grep "Storage"
Storage Controller Name (0):            IDE
Storage Controller Type (0):            PIIX4
Storage Controller Instance Number (0): 0
Storage Controller Max Port Count (0):  2
Storage Controller Port Count (0):      2
Storage Controller Bootable (0):        on
Storage Controller Name (1):            SCSI          -- here
Storage Controller Type (1):            LsiLogic
Storage Controller Instance Number (1): 0
Storage Controller Max Port Count (1):  16
Storage Controller Port Count (1):      16
Storage Controller Bootable (1):        on
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
```

## 0x08 Attach the newly resized disk to the Storage Controller of the Virtual Machine. 

In our case we're going to use the same name for the Storage Controller, **SATA Controller** (or **SCSI**), as revealed in the step above.

> 在我的 mac 上为 **SCSI** 设备，因此调整如下

```
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 VBoxManage storageattach ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932 --storagectl "SCSI" --port 0 --device 0 --type hdd --medium clone-ubuntu-zesty-17.04-cloudimg.vdi
➜  ubuntu-1704-server-cloudimg-amd64-vagrant_default_1508603202704_48932
```


## 0x09 Reboot the Virtual Machine using Vagrant.

- `df`: Find the name of the **logical volume** mapping the file-system is on (ie. `/dev/mapper/VolGroupOS-lv_root`).
- `fdisk -l`: Find the name of the **physical volume** (or device) that all the partitions are created on (ie. `/dev/sda`).


```
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant vagrant up
...
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant vagrant ssh
...
ubuntu@ubuntu-1704:~$ sudo -i
[#1#root@ubuntu-1704 ~]$df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            486M     0  486M   0% /dev
tmpfs            99M  3.7M   96M   4% /run
/dev/sda1       194G  6.7G  188G   4% /              -- here
tmpfs           495M     0  495M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           495M     0  495M   0% /sys/fs/cgroup
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/8d0d177a883e3f1896b6410a72e3c9e8c6d920992bc592c746f2ca7ee4e38bcd/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/edde8461d28a877b12991fb53888adc8312c139df0c0f7ab9c37c89639ee07e4/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/74a42e340f304ee77b1a2de744d15072fd030be82a3b1e49228bd96032a75670/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/2d690464e1903e9de3918be44b5c4a7191d476ba226b032e1cd26590f1bc9f53/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/f77dbf79a757d420a38b11fc42024a517c5174d27f42c22ada8fc648a80aa12b/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/4a714fbbd3b56982605a5b456701291171e46e0f5b415ca5f241d5433586d60a/merged
overlay         194G  6.7G  188G   4% /var/lib/docker/overlay2/2ca55d1cd2029b74bea2a4e90d828dcabc74321a7dd35a7c9e1afbe8131c5eaa/merged
shm              64M     0   64M   0% /var/lib/docker/containers/d86529e2f2493d9f028f8001b0c76010a83842e2e5a8255026fe920ed79f0ede/shm
shm              64M     0   64M   0% /var/lib/docker/containers/ed59e1d96023a56d46b50ae0879802c751dd2a8eef3abe014fea5058cb0fe680/shm
shm              64M     0   64M   0% /var/lib/docker/containers/7ba8b39c8e13273294d1270730a898137cf0b72cd72b524a0eadffeebfbf59fc/shm
shm              64M     0   64M   0% /var/lib/docker/containers/da639921258b8049474e85e211725d0516a4156c2e62e8e6999947ad3305673a/shm
shm              64M     0   64M   0% /var/lib/docker/containers/5c9a2700c33c8058d32ab775cab290f3eadd4baed32291c99504adc9bbe62cc9/shm
shm              64M     0   64M   0% /var/lib/docker/containers/b86ac186da0d1ed972fce74105106acf85d557e3fb24f64c09053601b6d76172/shm
shm              64M     0   64M   0% /var/lib/docker/containers/d2487450b4e8c5678a9e67d22e2be9bfe8e56a0ef5222e5bb2fdb99e4d075b35/shm
vagrant         234G   81G  154G  35% /vagrant
vagrant_data    234G   81G  154G  35% /vagrant_data
tmpfs            99M     0   99M   0% /run/user/1000
[#2#root@ubuntu-1704 ~]$
[#2#root@ubuntu-1704 ~]$
[#2#root@ubuntu-1704 ~]$fdisk -l
Disk /dev/sdb: 10 MiB, 10485760 bytes, 20480 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 200 GiB, 214748364800 bytes, 419430400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x79224dbc

Device     Boot Start       End   Sectors  Size Id Type
/dev/sda1  *     2048 419430366 419428319  200G 83 Linux
[#3#root@ubuntu-1704 ~]$
```

原文后面还有几步，但在我的环境下，似乎不需要；



----------

其他文章：

- [修改Vagrant box磁盘大小](http://blog.csdn.net/jackyzy823/article/details/50096587)
- [在 Mac/win7 下上使用 Vagrant 打造本地开发环境](https://segmentfault.com/a/1190000002645737)
- [Virtualbox扩充硬盘容量](http://blog.gongzhenhua.com/virtualbox-harddisk-resize/)
- [Resizing Vagrant box disk space](http://tuhrig.de/resizing-vagrant-box-disk-space/)
- [How can I increase disk size on a Vagrant VM?](https://askubuntu.com/questions/317338/how-can-i-increase-disk-size-on-a-vagrant-vm)
- [Resize /dev/sda1 disk of your  vagrant/virtualbox VM](https://tvi.al/resize-sda1-disk-of-your-vagrant-virtualbox-vm/)
- [Resize Your Vagrant / VirtualBox Disk](https://medium.com/@phirschybar/resize-your-vagrant-virtualbox-disk-3c0fbc607817)

