---
author: "Viki Pranata"
title: "OpenStack Swift via Kolla Ansible"
description : "Menambahkan layanan swift object storage pada openstack kolla ansible"
date: "2022-12-7"
tags: ["linux", "openstack"]
showToc: true
---

Menambahkan layanan swift object storage pada openstack yang dideploy dengan kolla ansible pada postingan [openstack for lab](/posts/openstack-for-lab)

## Persiapan
### Compute Node
openstack swift membutuhkan block storage untuk media penyimpanan.Tambahkan 1 hardisk pada tiap node untuk di khususkan sebagai storage swift.
| Node Name | Swift Volume | Ip Address |
| ---- | ---- | ---- |
| openstack-controller | | 10.79.0.10 |
| openstack-compute01 | 20GB | 10.79.0.11 |
| openstack-compute02 | 20GB | 10.79.0.12 |

> jalankan pada setiap compute node
```bash
# <WARNING ALL DATA ON DISK will be LOST!>
index=0
for disk in sdd; do
    parted /dev/${disk} -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
    sudo mkfs.xfs -f -L d${index} /dev/${disk}1
    (( index++ ))
done
```

Verifikasi block storage yang sudah di format
```bash
lsblk -f
```

### Controller Node
Sebelum menjalankan layanan openstack swift, kita perlu membuat _Object Ring, Account ring, dan Container Ring_ yang berfungsi untuk memberi tahu berbagai layanan Swift di mana data berada di kluster.
```bash
cat <<EOF | tee ~/swift-rings.sh
#!/usr/bin/bash
STORAGE_NODES=(10.79.0.11 10.79.0.12)
KOLLA_SWIFT_BASE_IMAGE="kolla/oraclelinux-source-swift-base:4.0.0"

mkdir -p /etc/kolla/config/swift

# Object ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/object.builder create 10 3 1

for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/object.builder add r1z1-${node}:6000/d${i} 1;
    done
done

# Account ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/account.builder create 10 3 1

for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/account.builder add r1z1-${node}:6001/d${i} 1;
    done
done

# Container ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/container.builder create 10 3 1

for node in ${STORAGE_NODES[@]}; do
    for i in {0..2}; do
      docker run \
        --rm \
        -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
        $KOLLA_SWIFT_BASE_IMAGE \
        swift-ring-builder \
          /etc/kolla/config/swift/container.builder add r1z1-${node}:6002/d${i} 1;
    done
done

for ring in object account container; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/${ring}.builder rebalance;
done
EOF
```

tambahkan permission exec pada file ~/swift-rings.sh dan jalankan dengan user sudo
```bash
chown +x ~/swift-rings.sh
sudo ~/swift-rings.sh
```

## Deploy openstack Swift
Aktivasi kolla python virtual environment
```bash
source ~/kolla/bin/activate
```

Lalu edit konfigurasi `nano /etc/kolla/globals.yml`
```bash
enable_swift: "yes"
enable_swift_s3api: "yes"
glance_backend_file: "yes"
glance_backend_swift: "no"
```

Selanjutnya deploy layanan swift dengan perintah:
```bash
kolla-ansible -i ~/multinode deploy
```

Setelah selesai semua `deactivate` kolla python virtual environment lalu install swift cli client
```bash
sudo apt instll -y python3-swiftclient
```

Verifikasi hasil pemasangan layanan openstack swift dengan perintah
```bash
openstack service list --long
```
![image](/assets/images/openstack-service-swift.jpg)


## Sumber Referensi
- https://docs.openstack.org/kolla-ansible/pike/reference/swift-guide.html