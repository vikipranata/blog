---
author: "Viki Pranata"
title: "Memperbesar Kapasitas Partisi di Linux"
description : "Cara Memperbesar Kapasitas Partisi di Linux"
date: "09-10-2022"
tags: ["linux", "tools", "storage"]
---

Masalah utama dalam penggunaan data adalah kapasitas, dalam linux ada beberapa perintah yang memudahkan kita untuk menaikan kapasitas penyimpanan.

Setelah menaikan kapasitas penyimpanan dari sisi host mesin virtual server, tinggal kita update dari sisi host yang akan di tingkatkan dengan bantuan tools berikut :

> growpart - extend a partition in a partition table to fill available space <cite>[^1]</cite>
[^1]: Dikutip dari laman [ubuntu manual pages](https://manpages.ubuntu.com/manpages/bionic/man1/growpart.1.html)


### Partisi Tanpa LVM
Meningkatkan kapasitas partisi _root_ dengan perintah    
`growpart /dev/sda 1`

Lalu gunakan perintah berikut untuk mengisi semua ruang partisi _root_    
`resize2fs /dev/sda1`

### Partisi dengan LVM
Meningkatkan kapasitas partisi lvm dengan perintah    
`pvresize /dev/sda1`

Lalu gunakan perintah berikut untuk mengisi semua ruang partisi lvm yang tersedia    
`lvresize --extents +100%FREE --resizefs /dev/xxxx/root`

Jika ingin menggunakan hanya 20GB saja bisa menggunakan perintah berikut    
`lvresize --size +20G --resizefs /dev/xxxx/root`    

### Verifikasi
Lalu verifikasi penyimpanan apakah sudah sesuai atau belum    
`df -H`
> Catatan :    
jika _no space left on the block device_, bisa menggunakan perintah berikut:    
`mount -o size=10M,rw,nodev,nosuid -t tmpfs tmpfs /tmp`