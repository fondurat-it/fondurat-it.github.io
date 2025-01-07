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
`
nano /etc/modules
`
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
