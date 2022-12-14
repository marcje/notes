# RHCSA study notes

This document contains my RHCSA study notes.

## Table of Contents

* [RHCSA study notes](#rhcsa-study-notes)
  * [Table of Contents](#table-of-contents)
  * [Essentials](#essentials)
    * [Documentation](#documentation)
    * [Shell access](#shell-access)
    * [Bash](#bash)
      * [Output redirection](#output-redirection)
      * [Scripting](#scripting)
    * [File management](#file-management)
      * [Links](#links)
      * [Compression and decompression](#compression-and-decompression)
      * [Text editing](#text-editing)
    * [Scheduled tasks](#scheduled-tasks)
  * [Boot management](#boot-management)
    * [Power commands](#power-commands)
    * [GRUB2](#grub2)
    * [Change root password from emergency shell](#change-root-password-from-emergency-shell)
    * [Change systemd target](#change-systemd-target)
    * [Change default kernel](#change-default-kernel)
  * [Disk management](#disk-management)
    * [Filesystems overview](#filesystems-overview)
    * [Partition and mount disks](#partition-and-mount-disks)
    * [Create SWAP](#create-swap)
    * [LVM](#lvm)
    * [Stratis](#stratis)
    * [VDO Virtual Data Optimizer](#vdo-virtual-data-optimizer)
    * [LVM on top of VDO](#lvm-on-top-of-vdo)
    * [NFS](#nfs)
  * [systemd](#systemd)
  * [journalctl](#journalctl)
    * [Storage modes](#storage-modes)
  * [Network management](#network-management)
    * [Firewall](#firewall)
    * [Time client](#time-client)
  * [Process management](#process-management)
    * [Process tuning with tuned](#process-tuning-with-tuned)
  * [User and permission management](#user-and-permission-management)
    * [Permissions overview](#permissions-overview)
    * [Users](#users)
      * [Passwords](#passwords)
      * [Super user access](#super-user-access)
    * [Groups](#groups)
    * [Permissions](#permissions)
    * [Shared folder](#shared-folder)
    * [File Access Control Lists](#file-access-control-lists)
    * [SELinux](#selinux)
  * [package management](#package-management)
    * [Application streams](#application-streams)
  * [Container management](#container-management)
    * [Container examples](#container-examples)

## Essentials

### Documentation

Documentation can be found using:

* manual pages
  * `man [COMMAND]`
  * Traditional format
* info command
  * `info [COMMAND]`
  * Default format with detailed information
* Text documentation
  * `/usr/share/doc/`
  * Includes text documents

### Shell access

It is possible to login on TTY (console), but it is common (especially for remote systems / servers) to access Linux using secure shell (SSH).

For SSH it is sometimes possible to login using your username and password, but this is highly discouraged. Better is to use keybased authentication.

To generate a SSH key pair using default settings:

```bash
ssh-keygen
```

To copy the public key to the remote system:

```bash
ssh-copy-id [USER]@[HOST_NAME]
```

To connect using SSH with verbose (for debugging purposes):

```bash
ssh [USER]@[HOST_NAME] -v
```

It is possible to setup useful aliases and defaults in the configuration file `~/.ssh/.config`.

To copy a local file using SSH:

```bash
scp [FILE] [USER]@[HOST_NAME]:[PATH/TO/FILE]
```

Or the other way around:

```bash
scp [USER]@[HOST_NAME]:[PATH/TO/FILE] [FILE]
```

It is also possible to use FTP (using sftp) for FTP based transfers.

Both on console and through SSH it is useful to be aware of terminal multiplexers like screen (traditional) and tmux (modern variant).

To open a named screen session:

```bash
screen -S [NAME]
```

To exit (detach) from the screen session and keep it running in the background:

```kbd
Ctrl-a + d
```

To show currently running screen sessions

```bash
screen -ls
```

To resume a running screen session:

```bash
screen -r [ID_OR_NAME]
```

To stop a session use `exit` from within the session.

### Bash

Bash is used as the default SHELL environment.

In bash it is possible to use curly brackets to expand a parameter:

```bash
grep '[PATTERN1]\|[PATTERN2]\|[PATTERN3]' /etc/{passwd,group,shadow}
mkdir -p /example/{[DIR1],[DIR2]}
```

#### Output redirection

It also is possible to redirect the output of a command, where Linux knows three I/O streams:

* 1: stdin
  * The standard input stream
* 2: stdout
  * The standard output stream
* 3: stderr
  * The standard error stream

To redirect output to a file:

```bash
[COMMAND] > [FILE]
```

To redirect output to a output file and errors to a error file:

```bash
[COMMAND] 2> error.txt 1> output.txt
```

Or to send both to the same file:

```bash
[COMMAND] > [FILE] 2>&1
```

To surpress error messages we can throw them in `/dev/null`:

```bash
[COMMAND] 2> /dev/null
```

#### Scripting

I will need to be able to create some basic BASH scripts from the top of my head. I left out functions, advanced syntax, shellcheck linter, etc. in order to be able to remember this more easily and created a small example script for my own reference:

```bash
#!/bin/bash

# A small example script.

# Access first parameter
USERFILE=$1

# Check if running as root
if [ "$EUID" -ne 0 ]
  then echo "Please run as root"
  exit 50
fi

# Additional for loop with sequence example
for i in `seq -w 1 100 do
    echo "Counting.. $i"
done

# Check if file is given as input
if [ "$USERFILE" = "" ] then
    echo "Please specify an input file"
    exit 10
# If file exists add every user within that file
elif test -e $USERFILE then
    for user in `cat $USERFILE` do
        echo "creating the "$user" user..."
        useradd -m $user; echo "$user:linux" | chpasswd
    done
    exit 20
# If file does not exist exit with an error
else
    echo "Invalid input file specified"
    exit 30
fi
```

### File management

#### Links

A link to a file can be created in two ways:

* Soft link
  * ln -s [SOURCE] [TARGET]
  * Actual link to the file
* Hard link
  * ln [SOURCE] [TARGET]
  * A mirror copy of the file with the same inode number

Find files by inode number (useful for finding hard links):

``` bash
find / -inum [INUM] -exec ls -li {} \; 2> /dev/null
```

#### Compression and decompression

RHEL8 allows for archiving and compression of files and folders:

* Archive
  * `tar` or `star`
  * Useful to create a single file from multiple folders and files
* Compression
  * `gzip` or `bzip2`
  * Used to reduce space by using compression algorithms.

It is possible to create a compressed archive, for example using `tar` and `gzip` together:

```bash
tar -cvfz [FILE_NAME].tar.gz [FOLDER]
```

To list the contents of a compress archive:

```bash
tar -tvf [FILE_NAME].tar.gz
```

To extract a compressed archive:

```bash
tar -xvf [FILE_NAME].tar.gz
```

To extract two specific files from a compressed archive:

```bash
tar -xvf [FILE_NAME].tar.gz [FILE_1] [FILE_2]
```

#### Text editing

VIM is the default text editor on RHEL8. It has three operating modes:

* Standard / command
  * Allows for navigating, standard operations and executing commands
* Insert
  * Allows for typing
* Visual
  * Allows for selecting text blocks.

I will leave out most VIM defaults, since I have worked with VIM for a long time. If additional practice is useful execute the `vimtutor` command on a system, which provides a VIM based tutorial.

Two small tips I have found during the course is you can open VIM in a dualpane mode:

```bash
vim -O [FILE]
```

And you can execute Linux commands and get the output immediately in the opened file, for example IP / network information:

```bash
:!ip a s eth1
```

### Scheduled tasks

Scheduled tasks in Linux can be managed by:

* AT
  * One time tasks
  * Absolute date / time or relative time (e.g. +7 days)
  * atq to manage jobs
* CRON
  * Recurring tasks
  * crontab and files (/etc/cron*) to manage jobs
* Systemd timers
  * Modern alternative to both AT and CRON

Most importantly for me will be cronjobs, which basically have the following syntax:

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
# minute, hour, day of month, month, day of week
* * * * * [USER] [PATH_TO_SCRIPT|COMMAND_TO_EXECUTE] [OUTPUT_REDIRECTION]
```

## Boot management

### Power commands

To check if systemctl is responsible for power commands:

```bash
which {shutdown,poweroff,reboot,halt} | xargs ls -l
which {shutdown,poweroff,reboot,halt} | xargs file
echo {shutdown,poweroff,reboot,halt} | xargs man
```

Reboot with info message:

```bash
systemctl reboot --message="[MESSAGE]"
```

Or with a delay of 15 minutes (not supported by systemctl):

```bash
sudo shutdown -r +15 "[MESSAGE]"
```

Cancel a scheduled shutdown (root permissions needed to broadcast a message to other users on the system):

```bash
sudo shutdown -c
```

### GRUB2

GRUB2 is the default bootloader for RHEL 8. The configuration files can be found in:

* /etc/default/grub
  * Standard configuration file for GRUB
* /etc/grub.d/
  * More advanced configuration for GRUB
* /boot/grub2/grub.cfg
  * grub2 bootloader configuration as used on boot, generated by grub2-mkconfig

A default example of editing the standard GRUB configuration:

```bash
vi /etc/default/grub
    grub_timeout=10
    grub_timeout_style=hidden
    # remove 'rhgb quiet' from the cmdline_linux line
mv /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak
mv /boot/grub2/grubenv /boot/grub2/grubenv.bak
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Change root password from emergency shell

During boot, edit GRUB, add `rd.break` to the kernel cmdline and continue the boot process to enter the emergency shell. From there we can chroot, change the password and reload SELinux:

```bash
mount -o rw,remount /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
mount -o ro,remount /sysroot
```

Continuing boot after this will force a relabel of SELinux across the system. After this we can reboot and login as root using the newly set password.

### Change systemd target

We can change either the default target or on boot change the target onetime in GRUB to boot into a different (graphical or shell) environment.

In grub we can specify the target at the end of the kernel cmdline by setting either `systemd.unit=multi-user.target` (shell) or `systemd.unit=graphical.target` (gui).

To make permanent changes we can retrieve the current default and change it through systemd:

```bash
systemctl get-default
systemctl set-default graphical
systemctl set-default multi-user
```

### Change default kernel

List installed kernels:

```bash
sudo rpm -qa | grep kernel-[0-9]
```

List default / active kernel:

```bash
sudo grubby --default-kernel; sudo grubby --default-title
```

List information on all kernel entries:

```bash
sudo grubby --info
```

List information on specific kernel entry:

```bash
sudo grubby --info=[INDEX_ID]
```

Set entry as default kernel and reboot to activate:

```bash
sudo grubby --set-default-index=[INDEX_ID]
```

## Disk management

### Filesystems overview

* XFS
  * Default on RHEL
  * Can grow, not reduce
  * Good for big systems
* EXT4
  * Smaller files and systems
  * Can grow and reduce
  * Good for systems with limited i/o capability and CPU bound workloads
* VFAT / FAT32
  * Good for sharing between different OS
* NFS
  * Network based storage
  * NFSv3
    * Additional services
    * Many firewall ports
    * TCP and UDP
  * NFSv4
    * Better perfomance and more options (like ACL's)
    * One service
    * One port
    * TCP
* SWAP
  * Used to extend RAM on disk
  * Necessary for suspending systems
* LVM
  * Layered storage
  * Dynamic (re)sizing
  * Snapshots
* Stratis
  * Comparable to LVM, developed by Red Hat
  * Layered storage
  * Dynamic (re)sizing (thin provisioned)
  * Snapshots
* VDO (Virtual Data Optimizer)
  * Layered storage
  * Data reduction (deduplication / compression)

### Partition and mount disks

For persistent mounts we can edit the `/etc/fstab` file, the last two fields in every line are used for backup and priority options. If this is set on 0 it will be skipped, on 1 backup and highest priority and on 2 for the last field only on lowest priority.

We use `fdisk` to partition `MBR` type partitions, `gdisk` for `GPT` type.

An example to partition a disk / partition as ext4:

```bash
[fdisk | gdisk] /dev/[DEVICE] # Partition as ext4 / Linux
mkfs.ext4 /dev/[DEVICE_PARTITION]
partprobe # Inform kernel of changes after fdisk
fsck.ext4 /dev/[DEVICE_PARTITION]
```

To mount the partition permanently:

```bash
mkdir [DIR]
lsblk -fs # get UUID for partition
vi /etc/fstab
  UUID=[UUID] [DIR] [FSTYPE] defaults 0 0
mount -a; mount
```

### Create SWAP

Create a SWAP file of 1 GB:

```bash
dd if=/dev/zero of=/swapfile bs=1024 count=1048576
```

Or create a SWAP partition:

```bash
[fdisk | gdisk] /dev/[DEVICE] # Partition as SWAP
```

And configure above as SWAP and mount persistently:

```bash
mkswap -L [LABEL] [SWAP_POINT]
swapon [SWAP_POINT]
vi /etc/fstab
  UUID=[UUID] [SWAP_POINT] swap defaults 0 0
  # Or if we have a label:
  LABEL=[LABEL] [SWAP_POINT] swap defaults 0 0
swapon -a
```

### LVM

Create a LVM storage pool:

```bash
pvs # Show configured physical volumes
pvcreate /dev/[DEVICE_ID] # Add physical volume to LVM
vgs # Show configured volume groups
vgcreate [GROUP_NAME] /dev/[DEVICE_ID] # Add physical volume to group
lvs # Show configured logical volumes
lvcreate -l 100%FREE -n [NAME] [GROUP_NAME] # Create logical volume from 100% free space in group
mkfs.ext4 /dev/mapper/[GROUP_NAME]-[LV_NAME] # Format as ext4
mkdir [DIR] # Create mount dir
# Mount persistently:
vi /etc/fstab:
  /dev/mapper[GROUP_NAME]-[LV_NAME] [MOUNT_DIR] ext4 defaults 0 0
mount -a # Mount all
mount # Show mounts
```

Extend a LVM storage pool:

```bash
lsblk # Get device name
pvcreate /dev/[DEVICE_NAME] # Add physical volume to lvm
vgextend [GROUP_NAME] /dev/[DEVICE_NAME] # Add physical volume to existing group
lvextend -L +1G [GROUP_NAME]/[VOLUME_NAME] # Extend size of group with 1 GB
resize2fs /dev/mapper/[GROUP_NAME]-[VOLUME_NAME] # Extend size of partition
```

Reduce a LVM storage pool:

```bash
lvreduce --resizefs --size [SIZE] /dev/mapper/[GROUP_NAME]-[VOLUME_NAME] # Reduce size of partition
vgreduce /dev/[VOLUME_GROUP] /dev/[DEVICE_ID] # Reduce size of group
pvremove /dev/[DEVICE_NAME] /dev/[DEVICE_NAME2] # Remove physical volume from group
```

Remove a LVM storage pool:

```bash
umount [MOUNT_DIR] # Unmount partition
lvremove [GROUP_NAME]/[VOLUME_NAME] # Remove partition from LVM
vgremove -f [GROUP_NAME] # Remove group from LVM
pvremove /dev/[DEVICE_NAME] /dev/[DEVICE_NAME2] # Remove physical volume(s) from LVM
wipefs -a /dev/[DEVICE_NAME] /dev/[DEVICE_NAME2] # Clean physical devices
```

Create and rollback a LVM snapshot:

```bash
lvcreate -L [SIZE] -s -n [SNAPSHOT_NAME] /dev/mapper/[GROUP_NAME]-[VOLUME_NAME] # Create a snapshot from an existing volume
lvs # Show LVM volumes, now including the snapshot
umount [MOUNT_POINT] # Unmount the original LVM volume mount point
lvconvert --merge /dev/mapper/[GROUP_NAME][SNAPSHOT_NAME] # Merge the snapshot into the original volume
lvs # Show LVM volumes, should have deleted the snapshot
mount /dev/mapper[GROUP_NAME]-[VOLUME_NAME] [MOUNT_DIR] # Remount the volume
```

### Stratis

Enable stratis service:

```bash
systemctl enable --now stratisd
```

Unlike LVM stratis does not need to initiate a physical device, they can be added directly to a pool.

To create a Stratis pool:

```bash
stratis pool create [GROUP_NAME] /dev/[DEVICE_ID] # Create a stratis pool
stratis pool list # Show current pools
stratis blockdev list # Show which physical devices belong to stratis 
statis fs create [GROUP_NAME] [FS_NAME] # Create a stratis filesystem
stratis fs # List stratis filesystems
mkdir [DIR] # Create mount dir
vi /etc/fstab:
    /stratis/[POOL_NAME]/[FILESYSTEM_NAME] [MOUNT_DIR] xfs defaults,x-systemd.requires=stratisd.service    0 0
```

Extend a Stratis pool:

```bash
lsblk # Get device name
stratis pool add-data [POOL_NAME] /dev/[DEVICE_ID] # Add device to pool
```

Remove a Stratis pool:

```bash
umount [MOUNT_DIR] # Unmount partition
stratis filesystem destroy [POOL_NAME] [FS_NAME] # Remove partition from stratis
stratis pool destroy [POOL_NAME] # Remove pool from stratis
wipefs -a /dev/[DEVICE_NAME] # Clean physical device
```

Create and rollback a stratis snapshot:

```bash
stratis fs snapshot [POOL_NAME] [FS_NAME] [SNAPSHOT_NAME] # Create a snapshot
stratis fs # Show stratis filesystems, now including the snapshot
stratis fs rename [POOL_NAME] [FS_NAME] [FS_NAME]-orig # Rename the original filesystem
stratis fs rename [POOL_NAME] [SNAPSHOT_NAME] [FS_NAME] # Rename the snapshot to the original filesystem name
umount [MOUNT_DIR]; mount /stratis/[POOL_NAME]/[FS_NAME] [MOUNT_DIR] # Remount the filesystem
```

### VDO (Virtual Data Optimizer)

VDO can be used to reduce data usage by using deduplication and compression. In terms of sizes and VDO it is important to know that:

* physical size
  * Is the size of the underlying block device
* available physical size
  * Is the portion of the device that is actually available
* logical size
  * Is the provisioned size which is generally larger than the available physical size

To create a VDO:

```bash
vdo create --name=[VOLUME_NAME] --device=/dev/[DEVICE_ID] --vdoLogicalSize=10G # Create the VDO volume
mkfs.xfs -K /dev/mapper[VOLUME_NAME] # Create a XFS partition
udevadm settle # Wait for all device events to be finished
vdostats --human-readable # Show vdo overview
vi /etc/fstab
    /dev/mapper/[VOLUME_NAME] [MOUNT_DIR] xfs defaults,_netdev,x-systemd.device-timeout=0,x-systemd.requires=vdo.service 0 0
mount -a
```

### LVM on top of VDO

To ensure the best of both worlds it is possible to configure LVM on top of a VDO volume, thus giving us deduplication and compression and the dynamic features of LVM.

```bash
systemctl enable --now vdo
systemctl status vdo
lsblk # Show block devices
vdo create --name=[VOLUME_NAME] --device=/dev/[DEVICE_ID] --vdoLogicalSize=10G # Create a VDO volume
udevadm settle # Wait for all device events to be finished
vdostats --human-readable # Show VDO overview
pvcreate /dev/mapper[VDO_VOLUME] # Add the volume to a LVM physical volume
vgcreate [GROUP_NAME] /dev/mapper/[VDO_VOLUME] # Create the LVM group
lvcreate -L 2G -n [LV_NAME] [GROUP_NAME] # Create the LVM logical volume
mkfs.xfs /dev/mapper/[GROUP_NAME]-[LV_NAME] # Partition the LV as XFS
mkdir [MOUNT_DIR] # Create the mount point
vi /etc/fstab
    /dev/mapper/[GROUP_NAME]-[LV_NAME] [MOUNT_DIR]  xfs defaults 0 0
mount -a
mount
df -h
vdostats --human-readable
```

### NFS

NFS can be used to create and mount network shared devices:

To setup a NFS mount:

```bash
vi /etc/exports
    [MOUNT_DIR] [EXTERN_HOST](rw, sync, no_root_squash) # Setup a network mount and allow access to a given host
systemctl enable --now nfs-server # Enable the service
systemctl status nfs-server # Status of service
showmount -e # List NFS mounts
```

To mount a NFS mount:

```bash
vi /etc/fstab
  [NFS_IP]:[FOLDER]   [MOUNT_DIR] nfs defaults,_netdev 0 0
```

Or using autofs, which will automatically mount the device when it is available or on demand (for example, when a user logs in):

```bash
vi /etc/auto.master
    /export/home /etc/auto.home
vi /etc/auto.home
    * 127.0.0.1:/home/& 
systemctl enable --now autofs
systemctl status autofs
```

## systemd

Systemd unit files can be found in:

* /usr/lib/systemd/system
  * Default for installed software
* /etc/systemd/system
  * Default for global systemd
* /run/systemd/system
  * (temporarily) files created at runtime
* ~/.config/systemd/user/
  * User specific unit files

To persistently enable and start a systemd unit:

```bash
systemctl enable --now [UNIT_NAME]
```

It is possible the unit file has been completely disabled (masked). In order to change this:

```bash
systemctl unmask [UNIT_NAME]
```

A (very) basic example of a custom unit file:

```bash
[Unit]
Description=Example systemd unit

[Service]
Type=simple
ExecStart=[START_COMMAND]
ExecStop=[STOP_COMMAND]

[Install]
WantedBy=multi-user.target
```

## journalctl

Lookup logs from specific unit file:

```bash
journalctl -u [UNIT_NAME]
```

Pattern matching within journalctl:

```bash
journalctl -g  [PATTERN]
journalctl -g  "[PATTERN1]|[PATTERN2]"
```

Search within specific time frame:

```bash
journalctl -S [STARTTIME] -U [ENDTTIME]
```

See various boot entries and retrieve logging from specific entries:

```bash
journalctl --list-boots
journalctl -b [BOOT-ID]
```

Flush journal, if storage is persistent this will write to file, otherwise journal gets wiped:

```bash
journalctl --flush
```

### Storage modes

The storage mode of journalctl is configured in /etc/systemd/journald.conf:

```bash
grep -i storage /etc/systemd/journald.conf
```

* volatile
  * /run/log/journal (wiped on reboot)
* persistent
  * /var/log/journal (persistent)
* auto
  * Persistent if /var/log/journal exists, otherwise volatile
* none
  * Drop logging

## Network management

On RHEL8 we can use `nmtui` (terminal user interface) or `nmcli` (CLI) to configure networking.

To setup an (currently unconfigured) additional interface:

```bash
ls -lha /etc/sysconfig/network-scripts/if*
cp /etc/sysconfig/network-scripts/if-[INTERFACE1] /etc/sysconfig/network-scripts/if-[INTERFACE2]
vi /etc/sysconfig/network-scripts/if-[INTERFACE2]
    # Edit name
    # Make sure "ONBOOT" is set to ensure the interface is enabled on boot
systemctl restart NetworkManager
nmtui # Configure and activate connection
```

To get a nice overview of both interfaces:

```bash
{ip a s eth0; ip a s eth1;} | grep inet\
{ip a s eth0; ip a s eth1;} | grep 'inet\inet6'
```

Test connectivity between both local interfaces:

```bash
ping -I [LOCAL_IP1] [LOCAL_IP2]
```

To delete an existing IP address using `nmcli`:

```bash
nmcli connection show 
nmcli device status
nmcli connection show [DEVICE]
nmcli connection down [DEVICE]
# Or alternatively use ifdown:
ifdown [DEVICE]
nmcli del [IP]
systemctl restart NetworkManager
nmcli connection modify [DEVICE] [OPTIONS]
nmcli connection up [DEVICE]
# or alternatively use ifup:
ifup [DEVICE]
```

### Firewall

RHEL8 uses firewall / firewall-cmd as a frontend to nftables.

Print current state:

```bash
firewall-cmd --state
```

Rule overview:

```bash
firewall-cmd --list-all
```

List default zone:

```bash
firewall-cmd --get-default-zone
```

Check current configuration:

```bash
firewall-cmd --check-config
```

Disable zone drifting:

```bash
vi /etc/firewalld/firewalld.conf
  AllowZoneDrifting=No
```

Add service by name:

```bash
firewall-cmd --add-service=[SERVICE] --permanent
```

Add service by name in specific zone:

```bash
firewall-cmd --zone=[ZONE] --add-service=[SERVICE] --permanent
```

Add service by port:

```bash
firewall-cmd --add-port=[PORT]/[PROTOCOL] --permanent
```

Reload firewall while keeping current state:

```bash
firewall-cmd --reload
```

Fully reload firewall, loses current state:

```bash
firewall-cmd --complete-reload
```

### Time client

RHEL8 uses chronyd as a time (NTP) synchronization client:

* chronyd
  * Background daemon
* chronyc
  * CLI interface
* ntpstat
  * Status tool

The configuration can be found in `/etc/chrony.{conf,keys}`.

Enable chronyd service:

```bash
systemctl enable --now chronyd
```

Show configured time servers:

```bash
chronyc sources
```

Show configured time servers with verbose info:

```bash
chronyc sources -v 
```

Additional information on timesync:

```bash
chronyc sourcestats
```

Statistics on servers:

```bash
chronyc serverstats
```

Sync status:

```bash
chronyc tracking
```

Force sync (discouraged to use in regards to running services and logs):

```bash
chronyc makestep
```

Systemd information on timesync:

```bash
timedatectl
```

ntpstats on timesync:

```bash
ntpstats
```

## Process management

We can set process priorities using (re)nice, either from using renice from within top (press `r` and set priority) or setting (re)nice on a specific process on startup / running PID:

```bash
nice -n [VALUE] [COMMAND]
renice -n [VALUE] [PID]
```

With nice -20 will give the process the highest priority, 19 the lowest, with 0 as the default.

We can also use `chrt` to set scheduling policies, like:

```bash
chrt --max
chrt -f -p [POLICY] [PID] 
chrt -p [PID]
```

The `chrt` command allows to set schedule priorities (see also `sched(7)`). The most important ones:

* SCHED_FIFO
  * Fixed priority for each thread
* SCHED_RR
  * Round robin variant of above
  * Useful when multiple threads need to run at the same priority level
* SCHED_OTHER
  * Default policy

We can kill (stop) processes by PID:

```bash
kill [SIGNAL_ID] [PID]
```

To make life easy we can kill (stop) processes by name as well:

```bash
pgrep [NAME]
pkill [NAME]
```

### Process tuning with tuned

Configuration files:

* Main:
  * /etc/tuned/tuned-main.conf
* Distro specific:
  * /usr/lib/tuned
* custom configs (overrides distro):
  * /etc/tuned/[FILENAME].conf

Show help information:

```bash
tuned-adm --help
```

Show current active profile:

```bash
tuned-adm active
```

Show available profiles:

```bash
tuned-adm list profiles
```

Show recommended profile:

```bash
tuned-adm recommend
```

Select specific profile:

```bash
tuned-adm profile [NAME]
```

Merge multiple profiles:

```bash
tuned-adm profile [NAME1] [NAME2]
```

Setup dynamic tuning to allow monitoring and change profile based on measurements we will have to set `dynamic_tuning = 1` in the main config and `systemctl restart tuned`.

## User and permission management

### Permissions overview

* Masks:
  * 1: execute
  * 2: write
  * 4: read
* Umask represents permissions that are NOT set, so a umask of 027 means that owner has all, group has read and execute and other has no rights
* setuid
  * chmod u+s
  * Execute a file with the privileges of the owner
* setgid
  * chmod g+s
  * File: execute a file with the authority of the group
  * Directory: ensures that all created files in that directory will inherit the group
* Sticky bit
  * chmod +t
  * Only root, the directory owner or the owner of the file can remove the file
* executable
  * chmod +x
  * Users can execute the file, useful for scripts

### Users

Switch user:

```bash
su - [USER]
```

Set default settings in:

```bash
/etc/login.defs
```

Show default settings for adding a new user:

```bash
sudo useradd -D
```

Add user with multiple groups (leave primary group to create user group):

```bash
useradd -m -g [PRIMARY_GROUP] -G [ADDITIONAL_GROUP],[ADDITIONAL_GROUP_2] [USER]; passwd [USER]
```

Add user with default password (linux):

```bash
useradd -m [USER]; echo "[USER]:linux" | chpasswd
```

Change additional groups for user:

```bash
usermod -G [GROUPS] [user]
```

Delete user and home directory:

```bash
userdel -r [USER]
```

#### Passwords

List password aging:

```bash
chage -l [USER]
```

Update password aging:

```bash
chage -m [MINIMUM_AGE] -M [MAXIMUM_AGE] [USER]
```

Update password aging on specific date:

```bash
chage -E $(date -d +60days +%Y-%m-%d) [USER]
```

Update password with given password using usermod:

```bash
usermod -p $(openssl passwd [PASSWORD]) [USER]
```

#### Super user access

Edit users with access:

```bash
visudo
```

Within /etc/sudoers we can activate and specify command aliases to easily grant access to users to specific commands. We can define users and groups and grant them specific privileges:

```bash
%[GROUP] ALL=[ALIAS1, ALIAS2, COMMAND]
```

### Groups

Add a new group:

```bash
groupadd [GROUP_NAME]
```

Change name of an existing group:

```bash
groupmod -n [NEW_NAME] [OLD_NAME]
```

Delete existing group:

```bash
groupdel [GROUP_NAME]
```

### Permissions

Change user and group on folder:

```bash
chown [USER]:[GROUP] [FOLDER]
```

Change only group on folder:

```bash
chown :[GROUP] [FOLDER]
```

### Shared folder

Setup shared folder permissions:

```bash
# Add write capabilities to group on folder
chmod g+w [FOLDER]
# By default users create files with group ownership of their primary group.
# We can correct this by setting the setgid bit
chmod g+s [FOLDER]
# To prevent users from deleting files owned by users in the same group
# we set the sticky bit.
chmod +t [FOLDER]
# We can now see the 's' and 't' added to the permissions on the folder
ls -ld
```

### File Access Control Lists

ACL's allow us to set permissions outside of the default user, group, other lists. For example allowing a specific user read access to a folder.

To list the existing / current ACL's on a folder:

```bash
getfacl [FOLDER]
```

To specify a new default ACL (read, write and execute) on a folder for a given group and update existing permissions:

```bash
setfacl -Rm d:g:[GROUP]:rwx,g:[GROUP]:rwx [FOLDER]
```

To specify a new default ACL (read only) on a user and update existing permissions:

```bash
setfacl -Rm d:u:[USER]:r--,u:[USER]:r-- [FOLDER]
```

Backup permissions to a given file:

```bash
getfacl -R [FOLDER] > [BACKUP_FILE]
```

Restore permissions from a backup file:

```bash
setfacl --restore=[BACKUP_FILE]
```

Disable (set to null) permissions from a specific user:

```bash
setfacl -m u:[USER]:0 [FOLDER]
```

Completely remove a user from ACL's:

```bash
setfacl -x u:[USER] [FOLDER]
```

### SELinux

SELinux is an additional layer of system security. It is responsible for "May [SUBJECT] do [ACTION] to [OBJECT]?", for example; "May a webserver access files in a user's home directory?".

SELinux modes:

* Enforcive
  * Default mode
  * Enforces loaded policies on the entire system
* Permissive
  * Good for debugging
  * Acts like enforcive mode, but does not deny any actions
* Disabled
  * Fully disable SELinux
  * Strongly discouraged

Get full SELinux status:

```bash
sestatus
```

Get current SELinux mode:

```bash
getenforce
```

Set current SELinux mode to permissive:

```bash
setenforce 0
```

Set current SELinux mode to enforcive:

```bash
setenforce 1
```

Relabel SELinux context, which is important after emergency shell changes, policy changes and other out of context SELinux updates:

```bash
touch /.autorelabel
```

If only a single file has been updated we can also use `restorecon`.

Retrieve audit information:

```bash
journalctl -xe [UNIT_NAME]
ausearch -c [UNIT_NAME] --raw | less
```

Create a custom policy module:

```bash
ausearch -c [UNIT_NAME] --raw | audit2allow -M [MODULE_FILE]
semodule -i [MODULE_FILE]
```

Change the SELinux file context to HTTPD context on a folder:

```bash
ls -laZ [FOLDER]
semanage fcontext -a -t httpd_sys_content_t "[FOLDER](/.*)?"
restorecon -Rv [FOLDER]
```

## package management

RHEL8 uses yum (based on DNF) as a package manager.

To add a yum repository from scratch:

```bash
vi /etc/yum.repos.d/[REPO_NAME].repo
    [[REPO_NAME]]
    name=[CUSTOM_DESCRIPTION]
    baseurl=[REPO_URL]
    enabled=1
    gpgcheck=0
sudo yum clean all # Clean cache
sudo yum repolist --all # List repositories and check if the new repository is enabled
```

Check for available updates:

```bash
sudo yum check-update
```

Install available updates:

```bash
sudo yum -y update
```

To search for a package:

```bash
sudy yum list --available [PACKAGE_NAME]
```

Check which package provides a certain tool (for example the tool `dig` in the package `bind-utils`):

```bash
sudo yum whatprovides [TOOL]
```

To see if a package has been locally installed using a RPM package:

```bash
rpm -q [PACKAGE_NAME]
```

To perform a regular install:

```bash
sudo yum -y install [PACKAGE_NAME]
```

Or to download a RPM file to local disk and install from there:

``` bash
sudo yum install --downloadonly --downloaddir [DOWNLOAD_DIR] [PACKAGE_NAME] # Download RPM file to local dir
sudo yum install ./[FILE_NAME] # To install from local file
sudo yum info [PACKAGE_NAME] # Get package info
sudy yum list --installed [PACKAGE_NAME] # Check if package has been installed
```

To remove a package from yum:

```bash
yum remove [PACKAGE_NAME] # To remove the package
yum autoremove [PACKAGE_NAME] # To remove the package and its dependencies
```

### Application streams

With application streams RHEL8 allows support for and easy switching between multiple versions of applications. It is for example easily possible to switch between MariaDB 10.3 and MariaDB 10.5.

Application streams have modules and profiles. Modules are a collection of RPM packages that make up a component and profiles are a collection of packages for particular use cases. Module streams can be active / inactive and only one stream can be active at a time. Each module can have a default stream and certain modules can depend on other streams.

To install a specific version using application streams:

```bash
sudo yum clean all # Clean cache
sudo yum module list # List all modules
sudo yum module list [PACKAGE_NAME] # List available modules for a given package
sudo yum module install [PACKAGE_NAME]:[VERSION]/[PROFILE] # Install specific module
```

To list installed modules:

```bash
sudo yum module list --installed [PACKAGE_NAME]
```

To reset a module to the default:

```bash
sudo yum module reset [PACKAGE_NAME]
```

## Container management

RHEL8 provides support for containers and their management:

* Podman
  * Directly manage containers and container images
* Skopeo
  * Inspect, copy, delete and sign container images
* Buildah
  * Create container images

RHEL8 supports the following registries out of the box:

* registry.redhat.io
  * Official redhat products containers
* registry.connect.redhat.com
  * Containers based on 3rd party packages
* docker.io (as configured in /etc/containers/registry.conf)
  * Official docker hub

To install container tools on RHEL8:

```bash
sudo yum -y module list --available container-tools
sudo yum -y module install container-tools
```

Display default help and information overview:

```bash
podman --help | less
podman info | less
```

Login at the Red Hat registry:

```bash
podman login registry.access.redhat.com
```

List available images on the system:

```bash
podman images
```

Show information on the RHEL8 based Universal Base Image:

```bash
skopeo inspect docker://registry.access.redhat.com/ubi8/ubi:latest
```

Search for and pull the RHEL8 based Universal Base Image:

```bash
podman search registry.access.redhat.com/ubi8
podman pull registry.access.redhat.com/ubi8/ubi:latest
```

Directly run the RHEL8 based Universal Base Image:

```bash
podman run -it registry.access.redhat.com/ubi8/ubi:latest
```

Run a container and startup / connect to a terminal session inside:

```bash
podman run -it registry.access.redhat.com/ubi8/ubi:latest /bin/bash
```

Run a command in a running container:

```bash
podman exec -it [CONTAINER_NAME] [COMMAND]
```

Show running containers:

```bash
podman ps
```

Show running and inactive containers:

```bash
podman ps --all
```

Stop all running containers (pass `container ID` or `name` instead of `-a` to stop a specific container):

```bash
podman stop -a
```

Remove all containers (pass `container ID` or `name` instead of `-a` to remove a specific container):

```bash
podman rm -a
```

Remove the RHEL8 based Universal Base Image:

```bash
podman rmi registry.access.redhat.com/ubi8/ubi:latest
```

### Container examples

Setup a webserver container:

```bash
mkdir ~/web_container_1_data/
skopeo inspect docker://registry.access.redhat.com/rhscl/httpd-24-rhel7 | less
# Run the HTTPD container, name it, expose port 8000 and route to 8080 in the container and bind to a persistent storage volume:
podman run -d --name web_container_1 -p 8000:8080 -v ~/web_container_1_data/:Z registry.access.redhat.com/rhscl/httpd-24-rhel7
podman port -a # Show opened ports and port mapping
curl http://127.0.0.1:8000 # Connect to webserver in container
```

Setup a container managed as a systemd user service:

```bash
loginctl enable-linger # Allow user services to be started at boot
loginctl show-user [USER] | grep -i linger # Check if enable-linger is enabled
mkdir -p ~/.config/systemd/user/ # Create the systemd user service file directory
mkdir -p [PERSISTENT_STORAGE] # Create the persistent storage directory for the container
podman run -d --name [CONTAINER_NAME] -p [EXPOSED_PORT:CONTAINER_PORT] -v [PERSISTENT_STORAGE]:Z [CONTAINER_IMAGE] # Setup and run the container
podman generate systemd --name [UNIT_NAME] --files --new # Generate a systemd unit file from the running container
systemctl --user daemon-reload # Loads new systemd service files
podman stop [CONTAINER_NAME] # Stop the running container
podman rm [CONTAINER_NAME] # Remove the created container
systemctl --user enable --now  [UNIT_NAME] # Start the container through systemd
systemctl --user status [UNIT_NAME] # Check if the service has started
podman ps -a # Check if the container has started
```
