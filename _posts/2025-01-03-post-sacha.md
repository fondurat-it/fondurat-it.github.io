---
title: "1.	Побудова спільного сховища кластера з двома вузлами на основі DRBD з мережею на основі RDMA."
categories:
  - Blog
tags:
  - DRBD 
  - RDMA
---

# 1.	1.	Побудова спільного сховища кластера з двома вузлами на основі DRBD з мережею на основі RDMA. 
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
### 4.3.3. Створюємо drbd пристрої: *`drbd0`* з фізичного пристрою: `nvme0n1p5 (а точніше з PARTUUID="a00c2c46-01f3-1b48-9ab2-b458fdadc3d7")`, *`drbd1`* з фізичного пристрою:  `/dev/nvme0n1p6 (PARTUUID="207cee45-7593-6441-9faf-99e294452177")`. Це робимо на обох вузлах(файли робимо на одному вузлі і копіюємо на інший вузол).
`nano /etc/drbd.d/r0.res`  

**Вигляд файлу:**
```
resource r0 {
    protocol  C;
#    device    /dev/drbd0 minor 0;
#    meta-disk internal;
    on pve1 {
            address 10.10.2.1:7788;
                volume 0 {
                    device    /dev/drbd0 minor 0;
                    disk /dev/disk/by-partuuid/a00c2c46-01f3-1b48-9ab2-b458fdadc3d7;
                    #disk /dev/nvme0n1p5;
                    meta-disk internal;
                }
    }
    on pve99 {
            address 10.10.2.2:7788;
                volume 0 {
                    device    /dev/drbd0 minor 0;
                    disk /dev/disk/by-partuuid/a00c2c46-01f3-1b48-9ab2-b458fdadc3d7;
                    #disk /dev/nvme0n1p5;
                    meta-disk internal;
                }
    }
    startup {
        degr-wfc-timeout 60;
        become-primary-on both;
    }
    disk {
        on-io-error   detach;
        c-plan-ahead  10;
        c-fill-target 100K;
        c-min-rate    500M;
        c-max-rate    1000M;

        no-disk-flushes;
        no-disk-barrier;
    }
    net {
        transport   rdma;
        max-buffers 36k;
        sndbuf-size 10M;
        rcvbuf-size 10M;
        allow-two-primaries;
    }
}
```
**Файл для другого ресурсу:**

`nano /etc/drbd.d/r1.res`  

**Вигляд файла:**
```
resource r1 {
    protocol  C;
#    device    /dev/drbd1 minor 1;
#    meta-disk internal;
    on pve1 {
            address 10.10.2.1:7789;
        volume 1 {
            # device name
            device /dev/drbd1  minor 1;
            # specify disk to be used for devide above
            disk /dev/disk/by-partuuid/207cee45-7593-6441-9faf-99e294452177;
            #disk /dev/nvme0n1p6;
            # where to create metadata
            # specify the block device name when using a different disk
            meta-disk internal;
        }
    }
    on pve99 {
             address 10.10.2.2:7789;
        volume 1 {
            device /dev/drbd1  minor 1;
            disk /dev/disk/by-partuuid/207cee45-7593-6441-9faf-99e294452177;
            #disk /dev/nvme0n1p6;
            meta-disk internal;
        }
    }
    startup {
        degr-wfc-timeout 60;
        become-primary-on both;
    }
    disk {
        on-io-error   detach;
        c-plan-ahead  10;
        c-fill-target 100K;
        c-min-rate    500M;
        c-max-rate    1000M;

        no-disk-flushes;
        no-disk-barrier;
    }
    net {
        transport   rdma;
        max-buffers 36k;
        sndbuf-size 10M;
        rcvbuf-size 10M;
        allow-two-primaries;
    }
}
```
### 4.3.4. Запускаємо службу та створюємо ресурси, ми робимо це на обох вузлах:
```
 # systemctl enable --now drbd

# systemctl restart drbd
# drbdadm create-md r{0,1}
# drbdadm up r{0,1}
```
**Потім лише на одному вузлі ми встановлюємо ресурси в первинний стан і запускаємо початкову синхронізацію:**  
```
root@pve1:~# drbdadm primary --force r{0,1}
```
**Очікуємо на синхронізацію.**  

**Робимо теж саме на другому вузлі.**  
```
root@pve99:~# drbdadm primary --force r{0,1}
```
**Перевіряємо:**  
```
[root@pve99 ~]$ drbdadm status

r0 role:Primary
  disk:UpToDate open:no
  pve1 role:Primary
    peer-disk:UpToDate

r1 role:Primary
  volume:1 disk:UpToDate open:no
  pve1 role:Primary
    volume:1 peer-disk:UpToDate

[root@pve99 ~]$
```
### 4.3.5. Далі ми створюємо фізичні пристрої LVM DRBD на обох вузлах:
```
root@pve1:~# pvcreate /dev/drbd{0,1}
  Physical volume "/dev/drbd0" successfully created
  Physical volume "/dev/drbd1" successfully created
 
root@ve99:~# pvcreate /dev/drbd{0,1}
  Physical volume "/dev/drbd0" successfully created
  Physical volume "/dev/drbd1" successfully created

і створіть групи томів лише на одному з вузлів:
root@pve1:~# vgcreate vg_drbd0 /dev/drbd0
  Volume group "vg_drbd0" successfully created
 
root@pve1:~# vgcreate vg_drbd1 /dev/drbd1
  Volume group "vg_drbd1" successfully created
```
**Тепер групи можна побачити на обох вузлах завдяки реплікації DRBD:**
```
root@pve1:~# vgs
  VG       #PV #LV #SN Attr   VSize    VFree  
  os         1  17   0 wz--n- <465.76g  71.73g
  pve        1   8   0 wz--n-  231.88g  16.00g
  vg_drbd0   1   5   0 wz--n-  159.99g 106.99g
  vg_drbd1   1   2   0 wz--n-  159.99g 147.99g
root@pve1:~#
```
**Другий:**  
```
[root@pve99 ~]$ root@pve1:~pvs
  PV         VG       Fmt  Attr PSize   PFree  
  /dev/drbd0 vg_drbd0 lvm2 a--  159.99g 106.99g
  /dev/drbd1 vg_drbd1 lvm2 a--  159.99g 147.99g
  /dev/sda3  pve      lvm2 a--  <36.76g   4.50g
[root@pve99 ~]$
```
### 4.3.6. Спільне сховище.  
**Ми переходимо до веб-консолі адміністратора PVE і додаємо сховище LVM у Datacenter, вибираємо vg_drbd0 зі спадного списку та встановлюємо прапорці для активних і спільних. У розкривному списку *Вузли* ми вибираємо обидва вузли pve1 і pve2 і натискаємо *Додати*. Повторюємо те саме для vg_drbd1.**

### 4.3.7. Створимо віртуальну машину з диском на спільному сховищі.  
**Зробимо тест:**  
```
root@debvsan:/home/vov# fio --filename=/dev/sda1 --direct=1 --rw=read --bs=1m --size=20G --numjobs=200 --runtime=60 --group_reporting --name=file1 

file1: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
...
fio-3.33
Starting 200 processes
Jobs: 200 (f=200): [R(200)][100.0%][r=1368MiB/s][r=1368 IOPS][eta 00m:00s]
file1: (groupid=0, jobs=200): err= 0: pid=1092: Sun Dec 29 13:12:07 2024
  read: IOPS=1367, BW=1368MiB/s (1434MB/s)(80.3GiB/60130msec)
    clat (msec): min=2, max=1177, avg=145.78, stdev=97.77
     lat (msec): min=2, max=1177, avg=145.78, stdev=97.77
    clat percentiles (msec):
     |  1.00th=[   30],  5.00th=[   52], 10.00th=[   73], 20.00th=[   94],
     | 30.00th=[   97], 40.00th=[  103], 50.00th=[  113], 60.00th=[  131],
     | 70.00th=[  153], 80.00th=[  188], 90.00th=[  253], 95.00th=[  330],
     | 99.00th=[  514], 99.50th=[  592], 99.90th=[  936], 99.95th=[  995],
     | 99.99th=[ 1150]
   bw (  MiB/s): min=  399, max= 2673, per=100.00%, avg=1385.68, stdev= 2.71, samples=23547
   iops        : min=  202, max= 2648, avg=1308.01, stdev= 2.75, samples=23547
  lat (msec)   : 4=0.01%, 10=0.02%, 20=0.11%, 50=4.66%, 100=32.25%
  lat (msec)   : 250=52.77%, 500=9.10%, 750=0.77%, 1000=0.28%, 2000=0.05%
  cpu          : usr=0.00%, sys=0.04%, ctx=92967, majf=0, minf=53984
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=82238,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1368MiB/s (1434MB/s), 1368MiB/s-1368MiB/s (1434MB/s-1434MB/s), io=80.3GiB (86.2GB), run=60130-60130msec

Disk stats (read/write):
  sda: ios=81909/617, merge=30/54, ticks=11925926/1508, in_queue=11927523, util=88.78%
root@debvsan:/home/vov#
```
**Враховуючи те що це максимум на вузлі pve99 (pcie 2.0 x 4), результат хороший. Це максимум, що може надати диск на цьому вузлі.**

**Зробимо переміщення диска віртуальної машини з одного сховища в інше.**  
```
root@pve1:~# qm move-disk 105 scsi0 vg_drbd0 

create full clone of drive scsi0 (vg_drbd1:vm-105-disk-0)
  Logical volume "vm-105-disk-2" created.
drive mirror is starting for drive-scsi0
drive-scsi0: transferred 6.0 MiB of 8.0 GiB (0.07%) in 0s
drive-scsi0: transferred 1013.0 MiB of 8.0 GiB (12.37%) in 1s
drive-scsi0: transferred 2.0 GiB of 8.0 GiB (24.40%) in 2s
drive-scsi0: transferred 2.9 GiB of 8.0 GiB (36.51%) in 3s
drive-scsi0: transferred 3.9 GiB of 8.0 GiB (48.63%) in 4s
drive-scsi0: transferred 4.9 GiB of 8.0 GiB (60.84%) in 5s
drive-scsi0: transferred 5.8 GiB of 8.0 GiB (72.97%) in 6s
drive-scsi0: transferred 6.8 GiB of 8.0 GiB (84.95%) in 7s
drive-scsi0: transferred 7.8 GiB of 8.0 GiB (97.08%) in 8s
drive-scsi0: transferred 8.0 GiB of 8.0 GiB (100.00%) in 9s, ready
all 'mirror' jobs are ready
drive-scsi0: Completing block job...
drive-scsi0: Completed successfully.
drive-scsi0: mirror-job finished
root@pve1:~#
```








    





---
Далі буде
---
