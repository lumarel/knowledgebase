# Disk operations

## Extend virtual disk lvm -> xfs

- `echo 1 > /sys/block/sda/device/rescan`
- `growpart /dev/sda 3`
- `pvresize /dev/sda3`
- `lvextend -l +100%FREE /dev/mapper/cl-root`
- `xfs_growpart /`
