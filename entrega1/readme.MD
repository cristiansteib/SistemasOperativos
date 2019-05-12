# Build kernel in ubuntu  18.04

### 1) Install all the requeriments:
Assuming that your project is under the folder /tmp/project


```bash
sudo apt-get install libncurses-dev git qemu-kvm
mkdir /tmp/project
cd /tmp/project
git clone --depth=1 --branch=2018.08 git://git.busybox.net/buildroor
```

### 2) Configure buildroot
First of all configure the building. To do that use the next commands:
```bash
cd /tmp/project/buildroot
make menuconfig
```

Complete the configuration with this options:
```
BuildRoot Configuration
│   
└─── Kernel
│   │   
│   │─── Kernel version
│   │     │
│   │     └(X) Latest version 
│   │   
│   └─── Kernel Configuration
│         │
│         └(X) Using a custom(def)config file
│           │
│           └ $(TOPDIR)/board/qemu/x86/linux.config       
│
└─── Filesystem images
│   │   
│   │─── (X) cpio the root filesystem (...)
│   │     │
│   │     └ Compression method (xz)
│   │   
│   └─── ext2/3/4 root filesystem
│         │
│         └ (X) ext2/3/4 varian (ext4)
│
└─── Bootloaders
│   │   
│   └─── (X) syslinux
│         │        
│         └ (X)instal mbr
```

### 3) Time to build the kernel 
```bash
make
```
This step take a while... Prepare a cup coffee and wait. 

When the operation finalize, it produces the following output files:
```bash
$ ls tmp/project/buildroot/output/images
bzImage
rootfs.cpio
rootfs.cpio.xz
rootfs.ext2
rootfs.ext4
syslinux 
```

### 4) Switch Root

Edit the files to mount the rootfs.ext4 as new root "/"

Unpack the cpio file, and replace the binary file sbin/init with a modified version.
```bash
cd tmp/project/buildroot/output/images
mkdir initramfs
cd initramfs
cpio -idv < ../rootfs.cpio
mv sbin/init sbin/init.bkp
touch sbin/init
chmod +x sbin/init
nano sbin/init

```
The next is the content of the file for the new sbin/init 
Copy && paste: 
```bash
#!/bin/sh
/bin/mount -t proc proc /proc                       # This should be present for the switch root. 
/bin/mount -o remount,rw /
/bin/mkdir -p /dev/pts /dev/shm
/bin/mount -a
/sbin/swapon -a
/bin/ln -sf /proc/self/fd /dev/fd
/bin/ln -sf /proc/self/fd/0 /dev/stdin
/bin/ln -sf /proc/self/fd/1 /dev/stdout
/bin/ln -sf /proc/self/fd/2 /dev/stderr
/bin/hostname -F /etc/hostname

echo -e "\e[33m\e[1m>>>>>>>>>> Obteniendo root\e[0m"
root=$(cat /proc/cmdline | sed -e 's/^.*root=//' -e 's/ .*$//')

echo -e "\e[33m\e[1m>>>>>>>>>> Montando rootfs\e[0m"
mkdir /mnt/rootfs                                   # Create a folder to mount the new filesystem
mount ${root} /mnt/rootfs/
mount --move /dev /mnt/rootfs/dev                   # This must be present, switch_root use this
echo -e "\e[33m\e[1m>>>>>>>>>> Rootfs montado\e[0m"
echo -e "\e[33m\e[1m>>>>>>>>>> Switch root\e[0m"
exec switch_root /mnt/rootfs/ /sbin/init            # Call switch_root {NEW ROOT FILE SYSTEM} {INIT MODIFIED}
```
The most important thing to do above is mount the `proc` to be able to have information about the kernel, like /proc/cmdline.
Then create a folder to mount the new root filesystem for then make the switch_root and run the new "init". 

Package the modified cpio

```bash
cd tmp/project/buildroot/output/images/initramfs
find . | cpio -H newc -o | xz --check=crc32 -v9 >../rootfs_mod2.cpio.xz
```

Add script to the SO initialization
```bash
cd tmp/project/buildroot/output/images
mkdir /mnt/temp1
sudo mount rootfs.ext2 /mnt/tmp1
sudo nano /mnt/tmp1/etc/init.d/S020FileNew
sudo chmod +x /mnt/tmp1/etc/init.d/S020FileNew 
```

Example script:
```bash
#!/bin/sh
echo "el mensaje"
```

then umount the filesystem:
```bash
sudo umount /mnt/tmp1
``` 

### 5) Run the machine
```bash
cd tmp/project/buildroot/output/
kvm -kernel bzImage -initrd rootfs_mod2.cpio.xz -append root=/dev/sda rootfs.ext4 -m 512
```

Recommendation.
- Don't change the permission of the files to root. The recommended permission is the current user with sudo privileges.
- Check the files generated with the buildroot before make any modification with the command.

```bash
kvm -kernel bzImage -initrd rootfs.cpio.xz -append root=/dev/sda rootfs.ext4 -m 512
```
The above mount the filesystem image "rootfs.ext4" as /dev/sda. If is not detected maybe something is missing in the configuration of the buildroot (Step 2).


If everything is ok you must see something like this:


## Credits:
- Muiña Sebastian Gabriel
- Steib Cristian