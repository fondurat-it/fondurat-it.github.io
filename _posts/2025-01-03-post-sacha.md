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

 ##  4.2. Реалізація.
     4.2.1. Залізо. Система. Конфігурація.
  
       Два вузли з встановленим Proxmox 8.3.2:
    
```
root@pve1:~# pveversion 
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
[root@pve99 ~]$ pveversion
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)

```

    з двупортовими картами Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT зв’язані напряму мідним кабелем.
    
  **Вузол pve1 : 10.10.1.1**

  Частина виводу команди  

```
lspci -vvv  
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]  
Subsystem: Mellanox Technologies Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT
```
```
Kernel driver in use: mlx4_core
Kernel modules: mlx4_core
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
