ls /dev/sd*
/dev/sda   /dev/sda2  /dev/sda4  /dev/sda6  /dev/sdb1
/dev/sda1  /dev/sda3  /dev/sda5  /dev/sdb   /dev/sdb2

ls /dev/sd*
/dev/sda   /dev/sda2  /dev/sda4  /dev/sda6
/dev/sda1  /dev/sda3  /dev/sda5  /dev/sdb

$ sudo parted /dev/sdb
GNU Parted 3.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model: SanDisk Ultra USB 3.0 (scsi)
Disk /dev/sdb: 15.4GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start  End     Size    Type     File system  Flags
 4      131kB  15.4GB  15.4GB  primary  fat32        boot, lba

http://mirrors.163.com/centos/7/isos/x86_64/
