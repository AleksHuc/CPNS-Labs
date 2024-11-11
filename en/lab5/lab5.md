# 5. Lab: Booting the Linux operating system over the network

## Instructions

0. Use the network and virtual machines from the previous labs.
1. Prepare the kernel and initial virtual disk (initrd) in the correct folder.
2. Update the system bootloader configuration file to boot the kernel and initial virtual disk.
3. Prepare a Network File System (NFS) server to provide a network drive.
4. Update the bootloader configuration file to use a network drive to boot the operating system.

## Additional information

[Instructions](https://www.kernel.org/doc/html/latest/admin-guide/nfs/nfsroot.html) for mounting the root file system via NFS (nfsroot).

## Detailed instructions

### 1. Task

The kernel and initial virtual disk are also obtained from the installation image of the selected Linux operating system and transferred to the `/srv/tftp` folder.

    cd /srv/tftp
    mkdir casper
    cd casper
    cp /mnt/casper/vmlinuz .
    cp /mnt/casper/initrd.lz .
    cd ..

### 2. Task

Let's check the contents of the bootloader configuration file.

	nano pxelinux.cfg/default

	default vesamenu.c32
	include live.cfg

We can see that the `default` file does not contain direct settings but imports them from the `live.cfg` file. Therefore, we open it and look at the definition of the first line of the menu for bootinh the operating system named `Start Linux Mint`.

	nano live.cfg

    label live
        menu label Start Linux Mint
        menu default
        kernel /casper/vmlinuz
        append boot=casper initrd=/casper/initrd.lz username=mint hostname=mint quiet splash --

We can see that the paths to the kernel and the initial virtual disk are absolute and we need to change them to relative (without / in front). We also remove the `quiet splash --` flags to allow the logs to be printed during booting.

    label live
        menu label Start Linux Mint
        menu default
        kernel casper/vmlinuz
        append boot=casper initrd=casper/initrd.lz username=mint hostname=mint

### 3. Task

Let's install an [NFS](https://en.wikipedia.org/wiki/Network_File_System) server that allows us to efficiently access the file over the network. For example, let's install `nfs-kernel-server`.

    apt install nfs-kernel-server

In the configuration file of the NFS server `/etc/exports`, we set the folder that we will offer over the network, for example `/media/cdrom`, to everyone who accesses the server and in read-only mode `*(ro)`.

    nano /etc/exports 

    /media/cdrom *(ro)

For the settings to take effect, restart the `nfs-kernel-server` NFS server.

    systemctl restart nfs-kernel-server.service

Now let's make sure all the installation image files are available in this folder. Next, unmount the `/mnt` folder, where we currently have the installation image mounted.

    umount /mnt

Then mount the installation image in the `/media/cdrom` folder.

    mount /dev/sr0 /media/cdrom

Let's test the operation of the NFS server locally by trying to connect the network drive to the `/mnt` folder.

    mount -t nfs localhost:/media/cdrom /mnt

    ls /mnt

    umount /mnt

### 4. Task

Now we have to enable access to the file system of the operating system via the NFS protocol in the configuration file of the bootloader `live.cfg`. By setting the network boot protocol to NFS `netboot=nfs`, the root of the file system to the folder provided by the NFS server `nfsroot=10.0.0.1:/media/cdrom` and adding the automatic acquisition of the IP address when the operating system is booting ` ip=dhcp`.

    nano live.cfg

	label live
        menu label Start Linux Mint
        menu default
        kernel casper/vmlinuz
        append boot=casper initrd=casper/initrd.lz username=mint hostname=mint netboot=nfs nfsroot=10.0.0.1:/media/cdrom ip=dhcp