# Vagrant Base Box creation process

## VMware

### Create VM image

- Create a ISO with the label `OEMDRV` with the [kickstart file](https://git.rockylinux.org/rocky/kickstarts/-/blob/r8/Rocky-8-x86_64-Vagrant.ks) `ks.cfg` in it (make sure the file has Linux file endings!)
- Create VM hull through VMware Workstation UI
- Use the following parameters:
  - Custom
  - Compatility: Workstation 10.x
  - Guest Operating System Installation: later
  - Guest Operating System: Linux - CentOS 8 64-bit
  - Virtual Machine Name: rockylinux-8.5-x86_64
  - Default CPU / Default memory
  - Network Type: Use NAT
  - I/O Controller: Paravirtualized SCSI
  - Disk Type: SCSI
  - Create new disk
  - Disk Capacity: 20G - single disk
- Remove VM from VMware Workstation
- Replace vmx file with the `install.vmx` as VM configuration
- Hook VM into VMware Workstation (easily possible with scan for VMs)
- Change CD paths to:
  - First disk: Rocky-8.5-x86_64-dvd1.iso path
  - Second disk: previously generated OEMDRV ISO file
- Boot
- After the automatic shutdown go to the Hard disk settings and compact the disk
- Remove machine from VMware Workstation

### Prepare image for distribution

- Replace vmx config with `distribute.vmx`
- Remove all files beside `*.vmdk` and `*.vmx`
- Add `metadata.json` and `VagrantFile` to the folder
- Place current directory into the root of the VM
- `tar cvzf ..\rockylinux-8.5-x86_64.box .\*`

### Differently

If the kickstart ISOs and a donor vmdk is already available it even shrinks the whole process to a bare minimum:

- Create directory for new vm `mkdir -p ~/vmware/rockylinux-9.0-x86_64`
- Copy donor vmdk file to this directory as `rockylinux-9.0-x86_64.vmdk`
- Copy `install.vmx` to this directory as `rockylinux-9.0-x86_64.vmx`
- Set the `displayName`, `nvram`, `scsi0:0.fileName`, `ide0:0.fileName` and `ide1:0.fileName` properties
- Run `vmware -qx ~/vmware/rockylinux-9.0-x86_64/rockylinux-9.0-x86_64.vmx` and wait until it closes again (it will exit the process if VMware Workstation was closed)
- Overwrite `rockylinux-9.0-x86_64.vmx` with `distribute.vmx`
- Set the `displayName`, `nvram` and `scsi0:0.fileName` properties
- Remove all files beside `*.vmdk` and `*.vmx`
- Add `metadata.json` and `VagrantFile` to the folder
- Place current directory into the root of the VM
- `tar cvzf ..\rockylinux-9.0-x86_64.box .\*`

## Testing the image

- Create a folder for the Vagrant environment
- Place current directory into the created folder
- `vagrant box add --name rockylinux-8.5-x86_64 <path-to-box-file>`
- `vagrant init rockylinux-8.5-x86_64`
- `vagrant up`
