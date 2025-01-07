---
title: "Саша тест"
categories:
  - Blog
tags:
  - Post Kod
  - notice
---

# 1.	Побудова спільного сховища кластера з двома вузлами на основі DRBD з мережею на основі RDMA. 
Proxmox, Debian, DRBD, RDMA.   
Увімкнути rdma для drbd.  
Жива міграція.
## 4.1. Початок. В зв'язку з здешевленням мережевих карт (пару двоxпортових я придбав з парою мідних кабелів за 100$ з доставкою) з'явилося бажання побудувати кластер з двох вузлів з двома точками відмови, з'єднаних напряму трьома дротовими мережами і однією WiFi, маючими по одному nvme диску для спільного сховища. Це повинен  був бути варіант кластера для дому, чи то для невеликої фірми. Вибір впав на drbd сховище з rdma.
 
**Начинаем:** Начнем наше продвижение!
{: .notice--success}

## 4.2. Реалізація.
### 4.2.1. Залізо. Система. Конфігурація.
  
  **Два вузли з встановленим Proxmox 8.3.2:**
    
```
root@pve1:~# pveversion 
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
[root@pve99 ~]$ pveversion
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)

```

з двупортовими картами Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT зв’язані напряму мідним кабелем.
    
  **Вузол pve1 : 10.10.1.1**

  Частина виводу команди:  

```
lspci -vvv  
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]  
Subsystem: Mellanox Technologies Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT
```
```
Kernel driver in use: mlx4_core
Kernel modules: mlx4_core
```
**Вузол pve99 : 10.10.1.2**  

Частина виводу команди:  
```
lspci -vvv
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
Subsystem: Mellanox Technologies Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT 
```
```
Kernel driver in use: mlx4_core
Kernel modules: mlx4_core
```
### 4.2.2. Перевірка працездатності. (Встановлення драйверів і налаштування, розглянуто окремо за посиланням:  

**Встановимо пакет з *rping*  на обох вузлах:**  
```
apt install rdmacm-utils
```
**На сервері:**  

**Вузол 1: pve1 (10.10.1.1):**  
```
root@pve1:~# rping -s -v 
```
**На клієнті :**  

**Вузол 2: pve99 (10.10.1.2):**  
```
[root@pve99 ~]$ rping -c -a 10.10.1.1 -v
```
***Побачимо приблизно такий вивід:***    
```
ping data: rdma-ping-54436: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRST
ping data: rdma-ping-54437: bcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTU
ping data: rdma-ping-54438: cdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUV
ping data: rdma-ping-54439: defghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVW
ping data: rdma-ping-54440: efghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWX
ping data: rdma-ping-54441: fghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXY
ping data: rdma-ping-54442: ghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
ping data: rdma-ping-54443: hijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[
ping data: rdma-ping-54444: ijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\
ping data: rdma-ping-54445: jklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\]
ping data: rdma-ping-54446: klmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^
^C
[root@pve99 ~]$
```
***Все добре.***  
### 4.2.3. На обох вузлах встановимо:   
```
apt install drbd-utils
```
```
apt install build-essential flex bison libssl-dev libnl-3-dev libnl-genl-3-dev libxml2-dev xmlto xsltproc python3-pytest python3-sphinx python3-yaml python3-jinja2 dkms
```
***Обовязково зупиняємо службу***  
```
systemctl list-units | grep drbd
drbdadm down all
```
**Вигружаємо модуль, обов’язково!!!:**  
```
lsmod | grep drbd
systemctl stop drbd
rmmod drbd
```
**Завантажуємо останню версію:**  
```
wget https://pkg.linbit.com//downloads/drbd/9/drbd-9.2.12.tar.gz
```
**Розпаковуємо:**  
```
tar xfz drbd-9.2.12.tar.gz
```

**Заходимо в директорію та компілюємо і встановлюємо:**  
```
cd drbd-9.2.12
make KVER=$(uname -r) all
make install
```
**Загружаємо модулі вручну:**  
```
modprobe drbd
modprobe drbd_transport_rdma
modprobe drbd_transport_lb-tcp
modprobe drbd_transport_tcp
modinfo drbd
```
**Перший вузол pve1:**  
```
root@pve1:~# cat /proc/drbd
version: 9.2.12 (api:2/proto:118-122)
GIT-hash: 2da6f528dc4ab3fd25c511f7b03531100e54ab08 build by root@pve1, 2024-12-17 19:11:24
Transports (api:21): rdma (9.2.12) lb-tcp (9.2.12) tcp (9.2.12)

root@pve1:~#
root@pve1:~# modinfo drbd
filename:       /lib/modules/6.8.12-5-pve/updates/drbd.ko
softdep:        post: handshake
alias:          block-major-147-*
license:        GPL
version:        9.2.12
description:    drbd - Distributed Replicated Block Device v9.2.12
author:         Philipp Reisner <phil@linbit.com>, Lars Ellenberg <lars@linbit.com>
srcversion:     C0FA687B694B5082F797130
depends:        lru_cache,libcrc32c
retpoline:      Y
name:           drbd
vermagic:       6.8.12-5-pve SMP preempt mod_unload modversions 
parm:           enable_faults:int
parm:           fault_rate:int
parm:           fault_count:int
parm:           fault_devs:int
parm:           disable_sendpage:bool
parm:           allow_oos:DONT USE! (bool)
parm:           minor_count:Approximate number of drbd devices (1U-255U) (uint)
parm:           usermode_helper:string
parm:           protocol_version_min:
                Reject DRBD dialects older than this.
                Supported: DRBD 8 [86-101]; DRBD 9 [118-122].
                Default: 86 (drbd_protocol_version)
parm:           strict_names:restrict resource and connection names to ascii alnum and a subset of punct (drbd_strict_names)
root@pve1:~#
```
**Другий вузол pve99:**  
```
[root@pve99 ~]$ cat /proc/drbd
version: 9.2.12 (api:2/proto:118-122)
GIT-hash: 2da6f528dc4ab3fd25c511f7b03531100e54ab08 build by root@pve99, 2024-12-17 23:30:23
Transports (api:21): rdma (9.2.12) lb-tcp (9.2.12) tcp (9.2.12)
[root@pve99 ~]$

[root@pve99 ~]$ modinfo drbd
filename:       /lib/modules/6.8.12-5-pve/updates/drbd.ko
softdep:        post: handshake
alias:          block-major-147-*
license:        GPL
version:        9.2.12
description:    drbd - Distributed Replicated Block Device v9.2.12
author:         Philipp Reisner <phil@linbit.com>, Lars Ellenberg <lars@linbit.com>
srcversion:     45B06AA2C33AAFE56B535F5
depends:        lru_cache,libcrc32c
retpoline:      Y
name:           drbd
vermagic:       6.8.12-5-pve SMP preempt mod_unload modversions 
parm:           enable_faults:int
parm:           fault_rate:int
parm:           fault_count:int
parm:           fault_devs:int
parm:           disable_sendpage:bool
parm:           allow_oos:DONT USE! (bool)
parm:           minor_count:Approximate number of drbd devices (1U-255U) (uint)
parm:           usermode_helper:string
parm:           protocol_version_min:
                Reject DRBD dialects older than this.
                Supported: DRBD 8 [86-101]; DRBD 9 [118-122].
                Default: 86 (drbd_protocol_version)
parm:           strict_names:restrict resource and connection names to ascii alnum and a subset of punct (drbd_strict_names)
[root@pve99 ~]$
```
**Припишемо завантаження модулів в файл:**  

`nano /etc/modules`  

**Приклад файла:**  
```
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
# Parameters can be specified after the module name.


# Generated by sensors-detect on Sun Sep 29 22:12:00 2024
# Chip drivers
it87
vfio
vfio_iommu_type1
vfio_pci
#vfio_virqfd #not necessary if kernel 6.2

nvmet
nvmet-tcp
nvme-rdma
rdma_ucm
rdma_cm
ib_uverbs
mlx4_ib
drbd
drbd_transport_rdma
drbd_transport_lb-tcp
drbd_transport_tcp

######################
```
## 4.3. Налаштування DRBD пристроїв.
**Є два вузла. У кожного вузла є диск: nvme0n1, до речі при завантаженні можлива зміна порядку: nvme0n1, nvme1n1…, якщо є декілька дисків, тому розглянемо випадок за міткою. Коли ми спарюємо диски, то краще мати їх ідентичні, ідентично розмічені.**   

### 4.3.1. За допомогою fdisk  розмітимо спочатку перший таким чином:  
```
root@pve1:~# fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 125026902 sectors
Disk model: KINGSTON SKC3000S512G                   
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: A1F37274-73E6-864F-B0B6-9BDD551BBD45

Device             Start       End  Sectors  Size Type
/dev/nvme0n1p1      4096    266239   262144    1G Linux swap
/dev/nvme0n1p2    266240  21237759 20971520   80G Linux filesystem
/dev/nvme0n1p3 105123840 121901055 16777216   64G Linux filesystem
/dev/nvme0n1p4 121901056 125026815  3125760 11.9G Linux swap
/dev/nvme0n1p5  21237760  63180799 41943040  160G Linux filesystem
/dev/nvme0n1p6  63180800 105123839 41943040  160G Linux filesystem

Partition table entries are not in disk order.
root@pve1:~#
```
**Збережемо розмітку диска в файл:**  

`root@pve1:~# sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512P6.dump`  

**Він виглядатиме, приблизно так:**
```
root@pve1:~# cat nvmeKINGSTON512P6.dump
label: gpt
label-id: A1F37274-73E6-864F-B0B6-9BDD551BBD45
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096  

/dev/nvme0n1p1 : start=        4096, size=      262144, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=B21C1B97-64EE-6948-AA0F-0BBA8797EB91
/dev/nvme0n1p2 : start=      266240, size=    20971520, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=8291281C-CA4C-0F4C-AD9E-C30C52FFBC04
/dev/nvme0n1p3 : start=   105123840, size=    16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=47A78E4D-FC8E-F54B-8DAF-D52A7908590C
/dev/nvme0n1p4 : start=   121901056, size=     3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=7F23E207-9E1A-1B42-962F-98BED3C1F479
/dev/nvme0n1p5 : start=    21237760, size=    41943040, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=A00C2C46-01F3-1B48-9AB2-B458FDADC3D7
/dev/nvme0n1p6 : start=    63180800, size=    41943040, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=207CEE45-7593-6441-9FAF-99E294452177
root@pve1:~#
```
**Тепер на іншому вузлі зразу збережемо розмітку на диск, так будемо робити і при заміні диска, що зіпсувався:**  

`sfdisk /dev/nvme0n1 < nvmeKINGSTON512P6.dump`  

**Відповідно маємо:**  
```
[root@pve99 ~]$ fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 125026902 sectors
Disk model: KINGSTON SKC3000S512G                   
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: A1F37274-73E6-864F-B0B6-9BDD551BBD45

Device             Start       End  Sectors  Size Type
/dev/nvme0n1p1      4096    266239   262144    1G Linux swap
/dev/nvme0n1p2    266240  21237759 20971520   80G Linux filesystem
/dev/nvme0n1p3 105123840 121901055 16777216   64G Linux filesystem
/dev/nvme0n1p4 121901056 125026815  3125760 11.9G Linux swap
/dev/nvme0n1p5  21237760  63180799 41943040  160G Linux filesystem
/dev/nvme0n1p6  63180800 105123839 41943040  160G Linux filesystem

Partition table entries are not in disk order.
[root@pve99 ~]$
```
**Також бачимо ідентичні мітки розділів диска, якими скористаємося далі:**  
```
[root@pve99 ~]$ blkid /dev/nvme0n1p5
/dev/nvme0n1p5: UUID="6b33cc5a02d7cc72" TYPE="drbd" PARTUUID="a00c2c46-01f3-1b48-9ab2-b458fdadc3d7"
[root@pve99 ~]$ blkid /dev/nvme0n1p6
/dev/nvme0n1p6: UUID="f0eb844c4d858ddc" TYPE="drbd" PARTUUID="207cee45-7593-6441-9faf-99e294452177"
[root@pve99 ~]$
```
### 4.3.2. Налаштування LVM фільтрів. Якщо ви будете використовувати lvm  для drbd пристроїв. ЦЕ ДУЖЕ ВАЖЛИВО! Щоб lvm працював поверх drbd пристрою  і не чіпав фізичних відповідних пристроїв, треба налаштувати файл /etc/lvm/lvm.conf .

**Редагуємо на обох вузлах:**  

`nano /etc/lvm/lvm.conf`  

**Вигляд частини файла, зазвичай його кінець :**  
```
devices {
     # added by pve-manager to avoid scanning ZFS zvols and Ceph rbds
     filter=["r|/dev/zd.*|","r|/dev/rbd.*|",
#"r|/dev/mapper/vg_drbd.*|",
#"r|/dev/.*vg_drbd.*|",
"r|.*a00c2c46-01f3-1b48-9ab2-b458fdadc3d7.*|",
"r|.*207cee45-7593-6441-9faf-99e294452177.*|",
"a|/dev/drbd.*|"]
     global_filter=["r|/dev/zd.*|","r|/dev/rbd.*|",
"r|.*a00c2c46-01f3-1b48-9ab2-b458fdadc3d7.*|",
"r|.*207cee45-7593-6441-9faf-99e294452177.*|",
"a|/dev/drbd.*|"]
}
```
**ОБОВ’ЯЗКОВО!** Щоб застосувати, треба виконати команду на обох вузлах, опісля потребує перезавантаження:
`update-initramfs -u`
{: .notice--danger}




    





---
Want to wrap several paragraphs or other elements in a notice? Using Liquid to capture the content and then filter it with `markdownify` is a good way to go.
---
```python
root@pve1:~# pveversion 
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
[root@pve99 ~]$ pveversion
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)


{% raw %}{% capture notice-2 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
{% endcapture %}{% endraw %}

<div class="notice">{% raw %}{{ notice-2 | markdownify }}{% endraw %}</div>
```

{% capture notice-2 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
{% endcapture %}

<div class="notice">
  {{ notice-2 | markdownify }}
</div>

Or you could skip the capture and stick with straight HTML.

```html
<div class="notice">
  <h4>Message</h4>
  <p>A basic message.</p>
</div>
```

<div class="notice">
  <h4>Message</h4>
  <p>A basic message.</p>
</div>
