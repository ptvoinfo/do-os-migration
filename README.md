# How migrate from Cent OS 6 to Ubuntu 22.04 and keep the IP address of the DigitalOcean's droplet

Migrating from CentOS 6 to Ubuntu 22.04 can offer several benefits. Here's a list of reasons why you might consider making this transition:

- End of Life for CentOS 6: CentOS 6 reached its end of life in November 2020, meaning it no longer receives security updates or support. This can make systems vulnerable to security problems.

- Long-Term Support (LTS): Ubuntu 22.04 is a Long-Term Support release, which means it will receive updates and support for five years from the date of its release, ensuring long-term stability and security.

- Modern Software Packages: Ubuntu 22.04 includes more recent software packages and features compared to CentOS 6, allowing access to newer versions of applications and tools.

- Easy Upgrades: You can upgrade to the next major version of Ubuntu without having to reinstall everything.

- System Performance: My Basic Digital Ocean droplet with 1 CPU and 1 GB RAM runs better on Ubuntu 22.04.

Moving from **CentOS** 6 to **Ubuntu 22.04** on a DigitalOcean server requires several important steps. 

This guide will walk you through the process, ensuring that you successfully transition your server while retaining your IP address.

You can't just move using DigitalOcean Web GUI (Web GUI) because it doesn't let you make a new snapshot with a different operating system. If you destroy a droplet, **you cannot retain the IP address**.

## Disclaimer: Reader Assumes Full Responsibility for Actions

**The author of this article bears no responsibility for any outcomes resulting from the reader's actions. 
All information is provided solely for informational purposes, and any use of this information is undertaken at the reader's own risk. 
The reader is advised to exercise independent judgment and seek professional advice if necessary before acting on any content presented. 
By proceeding with any actions based on this article, the reader acknowledges and accepts full responsibility for any potential consequences.**

## Prerequisites

1. Backup Data: Before beginning, ensure you have a complete backup of your data and configurations from the CentOS 6 droplet. Make a snapshot from Web GUI.
2. Access Details: Have SSH access to both your existing CentOS 6 droplet.
3. IP Address Information: Note down the current IP address and any related network configurations.

## Migration Steps

### 1. Create a New Ubuntu Droplet With a New IP Address

- Log in to DigitalOcean: Access your DigitalOcean account.

- Create Droplet: Navigate to the "Droplets" section and create a new droplet with Ubuntu 22.04.

- Select Plan and Region: The disk size of the new droplet should match the size of the old droplet. Select the same region for faster snapshot manipulations.

### 2. Transfer Data and Configurations

- Transfer Files: Use rsync or scp to transfer important data and configurations to the new Ubuntu droplet.

```
rsync -avz /path/to/data/ username@new_ubuntu_ip:/path/to/destination/
```

- Database Migration: If you are using databases, dump the databases on CentOS and import them into Ubuntu.

On CentOS
```
mysqldump -u root -p database_name &gt; db_backup.sql
```

On Ubuntu
```
mysql -u root -p database_name &lt; db_backup.sql
```

### 3. Configure Ubuntu Environment

- Install Necessary Software: On the new Ubuntu droplet, install any required software packages that were on your CentOS system.

```
sudo apt update
sudo apt install package_name
```

- Configure Services: Set up necessary services (e.g., web server, database server) to match the configurations from CentOS.

- Check that everything works as you expect.

- Power off the new droplet.

```
sudo poweroff
```

- Create a snapshot using Web GUI.

### 4. Overwriting The Old Droplet With The New One

- Power off the CentOS (old) droplet.

```
sudo poweroff
```

- Now, if you want to try to restore using the standard Web GUI, you cannot use a snapshot with another OS.

- Download the Digital Ocean Console Client: [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install/).

- Create an API token in Web GUI (Left menu - API - Generate New Token).

- Prepare and authenticate doctl.

```
doctl init
```

- Get IDs for your CentOS droplet and the new Ubuntu snapshot.

```
doctl compute droplet list
doctl compute snapshot list
```

- Rebuild the droplet with the following command line:

```
doctl compute droplet-action rebuild <CENTOD_DROPLET_ID> --image <UBUNTU_SNAPSHOT_ID>
```

- If you try to boot the droplet now, it won't start because the bootloader cannot find the correct "DOROOT" disk partition. You should update the bootloader.

### 5. Updating Bootloader

- Switch your old droplet with the newly prepared Ubuntu OS to the recovery mode with the ISO image (Web GUI - Recovery - Booting from a recovery ISO) because this droplet cannot boot without additional changes.

- Power on the droplet.

- Start the __recovery__ console (Web GUI - Access - Launch recovery console).

- Check that the droplet starts in the recovery mode where you can select from six options.

- Select "1 - Mount volume"

- Select "6 - Shell mode"

- Execute the following commands in the shell.

```
mount -o bind /dev /mnt/dev
mount -o bind /dev/pts /mnt/dev/pts
mount -o bind /proc /mnt/proc
mount -o bind /sys /mnt/sys
mount -o bind /run /mnt/run
chroot /mnt
```

- Check what your droplet’s disk is

```
fdisk -l
```

```
Disk /dev/vda: 25 GiB, 26843545600 bytes, 52428800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8A3333D6-33B3-4335-985-A352250488000
```

In that case, droplet’s disk is /dev/vda

- Check the disk label

```
lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
```
```
Output:
NAME   FSTYPE LABEL              UUID                                 MOUNTPOINT
vda                                                              
└─vda1 ext4   cloudimg-rootfs    050e1e34-39e6-4072-a03e-ae0bf90ba13a /
```

To mount the file system automatically each time the server boots, you’ll have the correct entry in the /etc/fstab file. 
This file contains information about all of your system’s permanent, or routinely mounted, disks. Open the file using **nano** or your favorite text editor:

```
nano /etc/fstab
```

- Find a record for a root syste "/":

```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
LABEL=UEFI      /boot/efi       vfat    defaults        0 0
```

- Change the label for the root partition "/" like "cloudimg-rootfs" to "DOROOT". Save changes.

- Change the partition label.

```
e2label /dev/vda1 DOROOT
```

- Install GRUB on your disk (replace /dev/vda with your droplet’s disk)

```
/usr/sbin/grub-install /dev/vda
update-grub
```

- Exit your chrooted environment.

```
exit
```

- Power off the droplet.

```
poweroff
```

### 6. Booting The Droplet In The Normal Mode


- Switch your droplet with the newly prepared Ubuntu OS to the normal mode (Web GUI - Recovery - Booting from Hard Drive).

- Power on the droplet.

- Check that the new droplet starts using the recovery console.

### 7. Post-Migration Steps

- IP Address: Check the restored snapshot can successfully take the old IP address.

```
ip address
```

- DNS Propagation: If there were changes in DNS settings, ensure they propagate correctly.

- Monitor Performance: Keep an eye on server performance to ensure everything runs smoothly.

- Ubuntu snapshot: Now, you can delete a temporary snapshot and the destroy temporary droplet that you used to spin up the new OS.
