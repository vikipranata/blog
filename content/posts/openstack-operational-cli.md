---
author: "Viki Pranata"
title: "Mengoperasikan OpenStack Lewat CLI"
description : "Mengoperasikan OpenStack lewat command line untuk kebutuhan lab"
date: "31-10-2022"
tags: ["linux", "openstack"]
showToc: true
---

Artikel ini adalah lanjutan dari [Membuat cluster OpenStack dengan Kolla-Ansible untuk kebutuhan lab](/posts/openstack-for-lab)

Untuk mengoperasikan openstack ada beberapa langkah sebagai berikut ini :
## Menggunakan OpenStack RC File
RC file ini adalah kumpulan variable yang akan digunakan untuk mengakses openstack dengan user maupun project tertentu.
![image](/assets/images/openstack-lab-1.jpg)
```bash
source ~/admin-openrc.sh
```

## Membuat File Image 
Downlaoad terlebih dahulu file cloud image sebagai contoh disini akan menggunakan Ubuntu Focal.
```bash
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
```
Lalu memngimport image yang sudah didownload tadi ke openstack dengan perintah berikut :
```bash
openstack image create --disk-format qcow2 --container-format bare --public --file ./focal-server-cloudimg-amd64.img ubuntu-focal-20.04
```
>Untuk melihat hasil dari image yang sudah dibuat bisa menggunakan perintah `openstack image list` lalu untuk melihat detail informasi gunakan perintah `openstak image show <ID>`

```bash
openstack image list
+--------------------------------------+--------------------+--------+
| ID                                   | Name               | Status |
+--------------------------------------+--------------------+--------+
| 45ff7650-20aa-44af-951f-6ccd3b0074d0 | ubuntu-focal-20.04 | active |
+--------------------------------------+--------------------+--------+
```

## Membuat Keypair 
Keypair  disini adalah ssh keypair dimana kita biasa membuat nya dengan perintah `ssh-keygen` yang akan mendapatkan dua buah file public key dan private key, hal ini akan membantu cloudinit untuk memasukan public key ke dalam instance. Untuk membuat keypair gunakan perintah berikut :
```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub controllerkey
```
>Untuk melihat hasil dari keypair yang sudah dibuat bisa menggunakan perintah `openstack keypair list` lalu untuk melihat detail informasi gunakan perintah `openstak keypair show <ID>`

```bash
openstack keypair list
+---------------+-------------------------------------------------+
| Name          | Fingerprint                                     |
+---------------+-------------------------------------------------+
| controllerkey | 82:81:55:07:bf:b4:df:5c:41:9d:24:4b:98:e8:ea:25 |
+---------------+-------------------------------------------------+
```
## Membuat Flavor 
Flavor disini adalah spesifikasi sebuah instance yang akan kita terapkan. Untuk membuatnya sebagai contoh _1 VCPU_ dan _1GB RAM_ dengan _10GB Disk_ dengan perintah berikut :
```bash
openstack flavor create --ram 1024 --disk 10 --vcpus 1 --public c1-standard-01
```
>Untuk melihat hasil dari flavor yang sudah dibuat bisa menggunakan perintah `openstack flavor list` lalu untuk melihat detail informasi gunakan perintah `openstak flavor show <ID>`

```bash
openstack flavor list
+--------------------------------------+----------------+------+------+-----------+-------+-----------+
| ID                                   | Name           |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+----------------+------+------+-----------+-------+-----------+
| d0cd7a21-dea1-4b62-8c4d-6281f6f64c82 | c1-standard-01 | 1024 |   10 |         0 |     1 | True      |
+--------------------------------------+----------------+------+------+-----------+-------+-----------+
```

## Membuat Security Group 
Security group  adalah sebuah firewall pada level instance. Untuk membuatnya sebagai contoh kita akan menerapkan koneksi masuk _ingress_ dari port ssh tcp 22 dan port web service tcp 80,443 serta protocol icmp dengan perintah berikut :
```bash
openstack security group create secg-basic-web --description 'Allow SSH, ICMP, and Web Services'
openstack security group rule create --protocol icmp secg-basic-web
for i in {22,80,443}; do openstack security group rule create --protocol tcp --ingress --dst-port $i secg-basic-web; done
```
>Untuk melihat hasil dari security group yang sudah dibuat bisa menggunakan perintah `openstack security group list` lalu untuk melihat detail informasi gunakan perintah `openstack security group rule list <ID>`

```bash
openstack security group rule list secg-basic-web
+--------------------------------------+-------------+-----------+-----------+------------+-----------------------+
| ID                                   | IP Protocol | Ethertype | IP Range  | Port Range | Remote Security Group |
+--------------------------------------+-------------+-----------+-----------+------------+-----------------------+
| 15eae94e-43f5-4e09-b757-4a8a1314f69e | None        | IPv4      | 0.0.0.0/0 |            | None                  |
| 4de0b24d-a3e5-4eac-a52f-adcaec71b28f | None        | IPv6      | ::/0      |            | None                  |
| 725a2b48-fbc7-4342-851e-7d9ee75b06d9 | icmp        | IPv4      | 0.0.0.0/0 |            | None                  |
| aaba8864-b75e-4ec2-a65d-23d70abb78ea | tcp         | IPv4      | 0.0.0.0/0 | 80:80      | None                  |
| aade68de-df66-4d56-9763-bffc9e47256f | tcp         | IPv4      | 0.0.0.0/0 | 22:22      | None                  |
| cba81b4b-6c61-4558-aa5c-4c3bcc7f8ae5 | tcp         | IPv4      | 0.0.0.0/0 | 443:443    | None                  |
+--------------------------------------+-------------+-----------+-----------+------------+-----------------------+
```

## Membuat Internal Network 
Internal network digunakan untuk komunikasi antar instance lewat jaringan internal, untuk membuatnya kita perlu mendefinisikan network dan subnet dengan menggunakan perintah berikut :
```bash
openstack network create int-net01
openstack subnet create --network int-net01 \
  --allocation-pool start=192.168.0.2,end=192.168.0.254 \
  --dns-nameserver 1.1.1.1 --gateway 192.168.0.1 --subnet-range 192.168.0.0/24 int-subnet01
```
>Untuk melihat hasil dari network dan subnet yang sudah dibuat bisa menggunakan perintah `openstack network list` dan `openstack subnet list` lalu untuk melihat detail informasit gunakan perintah `openstack network show <ID>` dan `openstack subnet show <ID>`

## Membuat External Network 
External network digunakan untuk komunikasi keluar instance lewat external provider, untuk membuatnya kita perlu mengetahui _provider physical network_ nya terlebih dahulu dengan perintah berikut :
```bash
sudo cat /etc/kolla/neutron-server/ml2_conf.ini | grep flat_network | awk '{print $3}'
```
lalu mendefinisikan network dan subnet dengan menggunakan perintah berikut :
```bash
openstack network create --share --external --provider-physical-network physnet1 \
  --provider-network-type flat ext-net01
openstack subnet create --network ext-net01 --no-dhcp --gateway 172.16.0.1 \
  --subnet-range 172.16.0.0/24 ext-subnet01
```

>Untuk melihat hasil dari network dan subnet yang sudah dibuat bisa menggunakan perintah `openstack network list` dan `openstack subnet list` lalu untuk melihat detail informasi gunakan perintah `openstack network show <ID>` dan `openstack subnet show <ID>`

```bash
openstack network list
+--------------------------------------+-----------+--------------------------------------+
| ID                                   | Name      | Subnets                              |
+--------------------------------------+-----------+--------------------------------------+
| 909b6065-1a13-4838-b38e-aade5560c33a | int-net01 | e099dcba-fa06-45f0-a3e2-37e05ff8dd4e |
| b9c73955-4c43-4336-90c7-c05485d49864 | ext-net01 | 21a81423-802e-4d5a-b24c-047bc1984d3d |
+--------------------------------------+-----------+--------------------------------------+

openstack subnet list
+--------------------------------------+--------------+--------------------------------------+----------------+
| ID                                   | Name         | Network                              | Subnet         |
+--------------------------------------+--------------+--------------------------------------+----------------+
| 21a81423-802e-4d5a-b24c-047bc1984d3d | ext-subnet01 | b9c73955-4c43-4336-90c7-c05485d49864 | 172.16.0.0/24  |
| e099dcba-fa06-45f0-a3e2-37e05ff8dd4e | int-subnet01 | 909b6065-1a13-4838-b38e-aade5560c33a | 192.168.0.0/24 |
+--------------------------------------+--------------+--------------------------------------+----------------+
```
## Membuat Router 
Router pada openstack berfungsi untuk menghubungkan antara traffic internal ke external baik koneksi masuk _ingress_ maupun koneksi keluar _egress_, untuk membuat router gunakan perintah berikut :
```bash
openstack router create router01
openstack router set --external-gateway ext-net01 router01
openstack router add subnet router01 int-subnet01
```

>Untuk melihat hasil dari router yang sudah dibuat bisa menggunakan perintah `openstack router list` lalu untuk melihat detail informasi gunakan perintah `openstak router show <ID>`

```bash
openstack router list
+--------------------------------------+----------+--------+-------+----------------------------------+-------------+-------+
| ID                                   | Name     | Status | State | Project                          | Distributed | HA    |
+--------------------------------------+----------+--------+-------+----------------------------------+-------------+-------+
| 1f8866ce-e3cf-408e-98b0-37cfca016e0a | router01 | ACTIVE | UP    | 408180ab780d48febd417e589ae1b09f | False       | False |
+--------------------------------------+----------+--------+-------+----------------------------------+-------------+-------+
```

## Membuat Instance atau Virtual Machine 
Setelah persiapan langkah diatas sudah diterapkan, barulah kita bisa membuat instance dengan perintah berikut :
```bash
openstack server create --flavor c1-standard-01 \
  --image ubuntu-focal-20.04 \
  --key-name controllerkey \
  --security-group secg-basic-web \
  --network int-net01 --wait \
  web-server-ubuntu01
```
>Untuk melihat hasil dari instance yang sudah dibuat bisa menggunakan perintah `openstack instance list` lalu untuk melihat detail informasi gunakan perintah `openstak instance show <ID>`

## Memasang Floating IP pada Instance 
Floating IP difungsikan sebagai NAT untuk diterapkan ke internal ip pada instance, untuk membuat floating ip gunakan perintah berikut :
```bash
openstack floating ip create --floating-ip-address 172.16.0.10 ext-net01
openstack server add floating ip web-server-ubuntu 172.16.0.10
openstack server list
```
>Untuk melihat hasil dari floating ip yang sudah dibuat bisa menggunakan perintah `openstack floating ip list` lalu untuk melihat detail informasi gunakan perintah `openstak floating ip show <ID>`

## Membuat Volume Storage
Volume digunakan sebagai persistent disk pada instance berupa block storage, untuk membuat volume dan memasangkannya pada instance gunakan perintah berikut :
```bash
openstack volume create --size 10 --wait web-peristent-disk
openstack server add volume web-server-ubuntu web-persistent-disk --device /dev/vdb
```
Selanjutnya tinggal memformat volume /dev/vdb via ssh kedalam instance.

Untuk membuat persistent storage pada partisi root pada saat instance dibuat, gunakan perintah berikut :
```bash
openstack volume create --size 10 --image ubuntu-focal-20.04 web-server-ubuntu02
openstack server create --flavor c1-standard-01 \
  --key-name controllerkey \
  --security-group secg-basic-web \
  --network int-net01 \
  --volume web-server-ubuntu02 --wait \
  web-server-ubuntu02
```
>Untuk melihat hasil dari volume yang sudah dibuat bisa menggunakan perintah `openstack volume list` lalu untuk melihat detail informasi gunakan perintah `openstak volume show <ID>`

## Resize Instance dengan Flavor
Semakin berjalannya waktu spesifikasi instance yang kita buat terkadang perlu di upgrade untuk memenuhi kebutuhan sumber daya yang akan digunakan, untuk menerapkan nya ikuti perintah berikut :
```bash
openstack flavor create --vcpus 2 --ram 2048 --disk 15 --public c2-standard-01
openstack server resize --flavor c2-standard-01 web-server-ubuntu01
openstack server list | grep web-server-ubuntu01
openstack server resize confirm web-server-ubuntu01
```

## Referensi
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/image-v2.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/keypair.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/flavor.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/security-group.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/security-group-rule.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/network.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/server.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/floating-ip.html
- https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/volume.html
- https://docs.openstack.org/nova/ussuri/user/resize.html