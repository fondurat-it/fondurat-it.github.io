---
title: "Саша тест"
categories:
  - Blog
tags:
  - Post Formats
  - notice
---

A notice displays information that explains nearby content. Often used to call attention to a particular detail.

When using Kramdown `{: .notice}` can be added after a sentence to assign the `.notice` to the `<p></p>` element. 

**Changes in Service:** We just updated our [privacy policy](#) here to better service our customers. We recommend reviewing the changes.
{: .notice}

**Primary Notice:** Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer nec odio. [Praesent libero](#). Sed cursus ante dapibus diam. Sed nisi. Nulla quis sem at nibh elementum imperdiet.
{: .notice--primary}

**Info Notice:** Lorem ipsum dolor sit amet, [consectetur adipiscing elit](#). Integer nec odio. Praesent libero. Sed cursus ante dapibus diam. Sed nisi. Nulla quis sem at nibh elementum imperdiet.
{: .notice--info}

**Warning Notice:** Lorem ipsum dolor sit amet, consectetur adipiscing elit. [Integer nec odio](#). Praesent libero. Sed cursus ante dapibus diam. Sed nisi. Nulla quis sem at nibh elementum imperdiet.
{: .notice--warning}

**Danger Notice:** Lorem ipsum dolor sit amet, [consectetur adipiscing](#) elit. Integer nec odio. Praesent libero. Sed cursus ante dapibus diam. Sed nisi. Nulla quis sem at nibh elementum imperdiet.
{: .notice--danger}

**Success Notice:** Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer nec odio. Praesent libero. Sed cursus ante dapibus diam. Sed nisi. Nulla quis sem at [nibh elementum](#) imperdiet.
{: .notice--success}

# 1.	Побудова спільного сховища кластера з двома вузлами на основі DRBD з мережею на основі RDMA. Proxmox, Debian, DRBD, RDMA. Увімкнути rdma для drbd. Жива міграція.  
  ## 4.1. Початок. В зв'язку з здешевленням мережевих карт (пару двоxпортових я придбав з парою мідних кабелів за 100$ з доставкою) з'явилося бажання побудувати кластер з двох вузлів з двома точками відмови, з'єднаних напряму трьома дротовими мережами і однією WiFi, маючими по одному nvme диску для спільного сховища. Це повинен  був бути варіант кластера для дому, чи то для невеликої фірми. Вибір впав на drbd сховище з rdma.  

  ## 4.2. Реалізація.  
   ### 4.2.1. Залізо. Система. Конфігурація.  
    Два вузли з встановленим Proxmox 8.3.2:
```html
root@pve1:~# pveversion 
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
[root@pve99 ~]$ pveversion
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
```

   з двупортовими картами Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT зв’язані напряму мідним кабелем. 





---
Want to wrap several paragraphs or other elements in a notice? Using Liquid to capture the content and then filter it with `markdownify` is a good way to go.
---
```html
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
