# Disk operations

## Extend virtual disk lvm -> xfs

```sh
echo 1 > /sys/block/sda/device/rescan
growpart /dev/sda 3
pvresize /dev/sda3
lvextend -l +100%FREE /dev/mapper/lvmvol-root
xfs_growfs /
```

## Extend virtual disk lvm -> ext4

```sh
echo 1 > /sys/block/sda/device/rescan
growpart /dev/sda 3
pvresize /dev/sda3
lvextend -l +100%FREE /dev/mapper/lvmvol-root
resize2fs /dev/mapper/lvmvol-root
```

## Extend virtual disk lvm -> btrfs

```sh
echo 1 > /sys/block/sda/device/rescan
growpart /dev/sda 3
pvresize /dev/sda3
lvextend -l +100%FREE /dev/mapper/lvmvol-root
btrfs filesystem resize max /
```

## Extend virtual disk lvm -> swap

```sh
echo 1 > /sys/block/sda/device/rescan
growpart /dev/sda 3
pvresize /dev/sda3
lvextend -l +100%FREE /dev/mapper/lvmvol-swap
swapoff /dev/mapper/lvmvol-swap
mkswap /dev/mapper/lvmvol-swap
swapon /dev/mapper/lvmvol-swap
```
