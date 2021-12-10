# Secureboot docs

## How to make sure that it works?

[https://access.redhat.com/articles/5337691](https://access.redhat.com/articles/5337691)

Run the following on your system to check if Secureboot is enabled:

```bash
mokutil --sb-state
```

The expected response is `Secureboot enabled`

When it is disabled or not configured it will show `Failed to read Secureboot` or `Secureboot disabled`

## What about the underlying UEFI?

Most systems force Secureboot, if it is enabled in the settings, so you are unable to boot as long as the key is not in the SB enclave.

## What about the key usage inside the OS beside the booting

Some applications will check the trustworthiness of their kernel module (i.e. kdump), so these need access to the certificates which are stored in the Secureboot store of the EFI firmware.

Which keys are available can be checked with `keyctl`, and mostly the platform keys are the only needed ones so this is the command which we need (this is the output on how it should look on VMware ESXi):

```bash
keyctl show %:.platform
Keyring
1029051156 ---lswrv      0     0  keyring: .platform
 963042462 ---lswrv      0     0   \_ asymmetric: VMware, Inc.: 4ad8ba0472073d28127706ddc6ccb9050441bbc7
 610794683 ---lswrv      0     0   \_ asymmetric: Rocky Enterprise Software Foundation: Rocky Linux UEFI CA TEST: 6bec785c36a1e260adf88d88afcc078250706822
 790197614 ---lswrv      0     0   \_ asymmetric: Rocky Enterprise Software Foundation: Rocky Linux Secure Boot Root CA: 4c2c6bd7d64ee81581cab8e986661f65e2166fc4
 933856209 ---lswrv      0     0   \_ asymmetric: Microsoft Windows Production PCA 2011: a92902398e16c49778cd90f99e4f9ae17c55af53
 408994474 ---lswrv      0     0   \_ asymmetric: Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4
 585418101 ---lswrv      0     0   \_ asymmetric: VMware, Inc.: VMware Secure Boot Signing: 04597f3e1ffb240bba0ff0f05d5eb05f3e15f6d7
```

Because of the current state it looks like that:

```bash
# keyctl show %:.platform
Keyring
  75387545 ---lswrv      0     0  keyring: .platform
  12012862 ---lswrv      0     0   \_ asymmetric: VMware, Inc.: 4ad8ba0472073d28127706ddc6ccb9050441bbc7
 261459087 ---lswrv      0     0   \_ asymmetric: CentOS Secure Boot CA 2: 70007f99209c126be14774eaec7b6d9631f34dca
 833827591 ---lswrv      0     0   \_ asymmetric: Microsoft Windows Production PCA 2011: a92902398e16c49778cd90f99e4f9ae17c55af53
1022550040 ---lswrv      0     0   \_ asymmetric: Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4
 587540411 ---lswrv      0     0   \_ asymmetric: VMware, Inc.: VMware Secure Boot Signing: 04597f3e1ffb240bba0ff0f05d5eb05f3e15f6d7
```

Also other platforms aren't fully compatible (ASUS consumer mainboard for Xeon E3):

```bash
$ sudo keyctl show %:.platform
Keyring
 954102119 ---lswrv      0     0  keyring: .platform
 696574850 ---lswrv      0     0   \_ asymmetric: ASUSTeK MotherBoard SW Key Certificate: da83b990422ebc8c441f8d8b039a65a2
 555459722 ---lswrv      0     0   \_ asymmetric: Canonical Ltd. Master Certificate Authority: ad91990bc22ab1f517048c23b6655a268e345a63
1006873854 ---lswrv      0     0   \_ asymmetric: Microsoft Windows Production PCA 2011: a92902398e16c49778cd90f99e4f9ae17c55af53
 417996348 ---lswrv      0     0   \_ asymmetric: Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4
  79139055 ---lswrv      0     0   \_ asymmetric: ASUSTeK Notebook SW Key Certificate: b8e581e4df77a5bb4282d5ccfc00c071
```

And here is the working output from Hyper-V:

```bash
keyctl show %:.platform
Keyring
 331307537 ---lswrv      0     0  keyring: .platform
 976156866 ---lswrv      0     0   _ asymmetric: Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4
 461020499 ---lswrv      0     0   _ asymmetric: Rocky Enterprise Software Foundation: Rocky Linux Secure Boot Root CA: 4c2c6bd7d64ee81581cab8e986661f65e2166fc4
  90244324 ---lswrv      0     0   _ asymmetric: CISD FW Update - Certificate: 068268b4a41c93854262d49b4f788f13
 126142696 ---lswrv      0     0   _ asymmetric: Microsoft Windows Production PCA 2011: a92902398e16c49778cd90f99e4f9ae17c55af53
```

And finally the keys can also be found in `/proc/keys` if they show up in the OS. Easiest to read with:

```bash
# cat /proc/keys  | grep -i rocky
0abff8bd I------     1 perm 1f030000     0     0 asymmetri Rocky kernel signing key: 288b6d785ae4ebfb9875f758558b01f5f2b5c7f8: X509.rsa f2b5c7f8 []
1b7a9d53 I------     1 perm 1f010000     0     0 asymmetri Rocky Enterprise Software Foundation: Rocky Linux Secure Boot Root CA: 4c2c6bd7d64ee81581cab8e986661f65e2166fc4: X509.rsa e2166fc4 []
1f793bda I------     1 perm 1f030000     0     0 asymmetri Rocky Enterprise Software Foundation: Rocky Linux Driver Update Signing Cert: b3c94fccbae32745b11dcd9a9a3926acfcef2540: X509.rsa fcef2540 []
3c58b5bf I------     1 perm 1f030000     0     0 asymmetri Rocky Enterprise Software Foundation: Rocky Linux kpatch Signing Cert: 7392f78c54ed85dfb1391b46b23a14dd29fc7514: X509.rsa 29fc7514 []
```

And for an Intel NUC:

```bash
keyctl show %:.platform
Keyring
 331307537 ---lswrv      0     0  keyring: .platform
 976156866 ---lswrv      0     0   _ asymmetric: Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4
 461020499 ---lswrv      0     0   _ asymmetric: Rocky Enterprise Software Foundation: Rocky Linux Secure Boot Root CA: 4c2c6bd7d64ee81581cab8e986661f65e2166fc4
  90244324 ---lswrv      0     0   _ asymmetric: CISD FW Update - Certificate: 068268b4a41c93854262d49b4f788f13
 126142696 ---lswrv      0     0   _ asymmetric: Microsoft Windows Production PCA 2011: a92902398e16c49778cd90f99e4f9ae17c55af53
```

Also, this is how kdump acts if it is unable to verify it's kernel module:

```bash
# kdumpctl restart
kdump: kexec: unloaded kdump kernel
kdump: Stopping kdump: [OK]
kdump: Secure Boot is enabled. Using kexec file based syscall.
kdump: kexec: loaded kdump kernel
kdump: Starting kdump: [OK]
```
