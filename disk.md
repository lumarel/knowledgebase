# Disk operations

## Extend virtual disk lvm -> xfs

- `echo 1 > /sys/block/sda/device/rescan`
- `growpart /dev/sda 3`
- `pvresize /dev/sda3`
- `lvextend -l +100%FREE /dev/mapper/lvmvol-root`
- `xfs_growfs /`

## Extend virtual disk lvm -> ext4

- `echo 1 > /sys/block/sda/device/rescan`
- `growpart /dev/sda 3`
- `pvresize /dev/sda3`
- `lvextend -l +100%FREE /dev/mapper/lvmvol-root`
- `resize2fs /dev/mapper/lvmvol-root`
