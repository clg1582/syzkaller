# Setup: Ubuntu host, VMware vm, x86-64 kernel

These are the instructions on how to fuzz the x86-64 kernel in VMware Workstation with Ubuntu on the host machine and Debian Stretch in the virtual machines.

In the instructions below, the `$VAR` notation (e.g. `$GCC`, `$KERNEL`, etc.) is used to denote paths to directories that are either created when executing the instructions (e.g. when unpacking GCC archive, a directory will be created), or that you have to create yourself before running the instructions. Substitute the values for those variables manually.

## GCC and Kernel

You can follow the same [instructions](/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md) for obtaining GCC and building the Linux kernel as when using QEMU.

## Image

Install debootstrap:

``` bash
sudo apt-get install debootstrap
```

To create a Debian Stretch Linux user space in the $USERSPACE dir do:
```
mkdir -p $USERSPACE
sudo debootstrap --include=openssh-server,curl,tar,gcc,libc6-dev,time,strace,sudo,less,psmisc,selinux-utils,policycoreutils,checkpolicy,selinux-policy-default,firmware-atheros,open-vm-tools --components=main,contrib,non-free stretch $USERSPACE
```

Note: it is important to include the `open-vm-tools` package in the user space as it provides better VM management.

To create a Debian Stretch Linux VMDK do:

```
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-gce-image.sh -O create-gce-image.sh
chmod +x create-gce-image.sh
./create-gce-image.sh $USERSPACE $KERNEL/arch/x86/boot/bzImage
qemu-img convert disk.raw -O vmdk disk.vmdk
```

The result should be `disk.vmdk` for the disk image and `key` for the root SSH key. You can delete `disk.raw` if you want.

## VMware Workstation

Open VMware Workstation and start the New Virtual Machine Wizard.
Assuming you want to create the new VM in `$VMPATH`, complete the wizard as follows:

* Virtual Machine Configuration: Custom (advanced)
* Hardware compatibility: select the latest version
* Guest OS: select "I will install the operating system later"
* Guest OS type: Linux
* Virtual Machine Name and Location: select `$VMPATH` as location and "debian" as name
* Processors and Memory: select as appropriate
* Network connection: NAT
* I/O Controller Type: LSI Logic
* Virtual Disk Type: IDE
* Disk: select "Use an existing virtual disk"
* Existing Disk File: enter the path of `disk.vmdk` created above

When you complete the wizard, you should have `$VMPATH/debian.vmx`. From this point onward, you no longer need the Workstation UI.

Starting the Debian VM (headless):
``` bash
vmrun start $VMPATH/debian.vmx nogui
```

Getting the IP address of the Debian VM:
``` bash
vmrun getGuestIPAddress $VMPATH/debian.vmx -wait
```

SSH into the VM:
``` bash
ssh -i key root@<vm-ip-address>
```

Stopping the VM:
``` bash
vmrun stop $VMPATH/debian.vmx
```

## syzkaller

Once you start the VM and get its IP address, you can use syzkaller to fuzz the VM in [isolated](/docs/linux/setup_linux-host_isolated.md) mode.
