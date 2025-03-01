# WEB_SOLUTION_WITH_WORDPRESS

## Preparing storage infrastructure on two Linux servers and implementing a basic WEB SOLUTION USING WORDPRESS

### Introdution:

This project involves preparing storage infrastructure on two Linux servers and implement a basic web solution using WORDPRESS.

WORDPRESS is a free and open-source content management system written in PHP and paired with MySQL or MariaDB as its backend Relational
Database Management System (RDBMS).

It consists of two parts:

1. Configure storage subsystem for Web and Database servers based on Linux OS. The focus of
   this part is to give you practical experience of working with disks, partitions and volumes in
   Linux.

2. Install WordPress and connect it to a remote MySQL database server. This part of the project
   will solidify your skills of deploying Web and DB tiers of Web solution.

Generally, web, or mobile solutions are implemented based on what is called the **Three-tier Architecture.**

Three-tier Architecture is a client-server software architecture pattern that comprise of 3 separate layers.

In this project will be implementing this solution based on what is called the **Three-tier Architecture.**

**Three-tier Architecture** is a client-server software architecture pattern that comprise of 3 separate layers.

## PREREQUISITE

### The 3-Tier Setup

1. A Laptop or PC to serve as a client.
2. An EC2 Linux Server as a web server with 3 EBS Volumes: 10GB each(This is where you will install WordPress).
3. An EC2 Linux server as a database (DB) server with 3 EBS Volumes: 10GB each.

### STEPS

1. Configure storage subsystem for Web and Database servers based on Linux OS.
2. Install WordPress and connect it to a remote MySQL database server.

## Steps Involved:

- **Step 0:** Preparing Prerequisite. Creating Two AWS EC2 instances account with Red Hat Enterprise Linux 9.4.
- **Step 1:** Prepare a Web server
- **Step 2:** Prepare a Database Server
- **Step 3:** Install Wordpress on the Web Server EC2
- **Step 4:** Install MySQL on the DB Server EC2
- **Step 5:** Configure DB to work with WordPress
- **Step 6:** Configure WordPress to connect to remote database.

## Step 0: Preparing Prerequisite.

Create Two AWS EC2 instances account with Red Hat Enterprise Linux 9.4.

- A Web Server Instance
- A DB Server Instance

![instances](./web_server_image/instances.png)

## Step 1: Prepare a Web server

1. In the Web Server EC2, create 3 volumes each of 10GB
2. Attach all 3 volumes to the Web Server EC2 instance

![Volumes](./web_server_image/webserver%20EBS.png)

3.  Inspect the Block devices

             lsblk

    ![lsblk](./web_server_image/lsblk.png)

4.  Create a single Partition on each of the 3 disks using gdisk utility

             sudo gdisk /dev/nvme1n1
             sudo gdisk /dev/nvme2n1
             sudo gdisk /dev/nvme3n1

![partition](./web_server_image/partition%20nvme1n1.png)
![partition](./web_server_image/partition%20nvme2n1.png)
![partition](./web_server_image/partition%20nvme3n1.png)

5.  Use 1sblk utility to view the newly configured partition on each of the 3 disks.

         lsblk

![lsblk_partitioned](./web_server_image/partitioned%20lsblk.png)

6. Install Logical Volume Manager 2

LVM2 (Logical Volume Manager version 2) is a storage management system used in Linux to create flexible and scalable disk partitions. It allows you to manage disk space more efficiently compared to traditional partitioning methods.

         sudo yum install lvm2

Run `sudo lvmdiskscan` command to check for available partition

         sudo lvmdiskscan

![lvmdiskscan](./web_server_image/lvmdiskscan.png)

7.  Create Physical Volumes using pvcreate and verify using `sudo pvs`

         sudo pvcreate /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1

         sudo pvs

![sudo pvs](./web_server_image/sudo%20pvs.png)

8.  Add all 3 PVs to a volume group (VG) using vgcreate utility. Name the VG **webdata-vg**. Verify it using `sudo vgs`

         sudo vgcreate webdata-vg /dev/nvme3n1p1 /dev/nvme2n1p1 /dev/nvme1n1p1

         sudo vgs

![sudo vgs](./web_server_image/vgs.png)

9.  Craate 2 Logical Volumes **apps-Iv** (Use half of the PV size), and **logs-Iv** Use the remaining space of the PV size. Use `sudo lvs` to verify.
    _NOTE: apps-lv will be used to store data for the Website while, logs-Iv will be used to store data for logs._

         sudo lvcreate -n apps-lv -L 14G webdata-vg
         sudo lvcreate -n logs-lv -L 14G webdata-vg
         sudo lvs

![sudo lvs](./web_server_image/lvs.png)

10. Verify the entire setup

         sudo vgdisplay -v #view complete setup - VG, PV, and LV
         sudo 1sblk

![set up](./web_server_image/lsblk%20entire%20setup.png)

11. Use mkfs.ext4 to format the logical volumes with ext4 filesystem

         sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
         sudo mkfs -t ext4 /dev/webdata-vg/logs-lv

Outcome:
![ext4 apps_lv](./web_server_image/ext4%20apps-lv.png)
![ext4 logs_lv](./web_server_image/ext4%20logs-lv.png)

12. Create /var/www/html directory to store website files

         sudo mkdir -p /var/www/html

13. Create /home/recovery/logs to store backup of log data

         sudo mkdir -p /home/recovery/logs

14. Mount /var/www/html on apps-lv logical volume

         sudo mount /dev/webdata-vg/apps-lv /var/www/html/

15. Use rsync utility to backup all the files in the log directory _/var/log_ into _/home/recovery/logs_ (This is required before mounting the filesystem)

         sudo rsync -av /var/log/ /home/recovery/logs/

16. Mount _/var/log_ on _logs-lv_ logical volume. (Note that all the existing data on /var/log will be deleted.)

         sudo mount /dev/webdata-vg/logs-lv /var/log

17. Restore log files back into _/var/log_ directory

         sudo rsync -av /home/recovery/logs/ /var/log

18. Update /etc/fstab file so that the mount configuration will persist after restart of the server.

    The UUID of the device will be used to update the /etc/fstab file;

         sudo blkid

![blkid](./web_server_image/blkid.png)

19. Update /etc/fstab in this format using your own UUID and rememeber to remove the leading and ending quotes.

         sudo vi /etc/fstab

Paste the code below

**Mount for word press**
UUID="4f62026b-9ba3-4e02-9fe3-9890b7cfd6d8" /var/log ext4 defaults 0 0
UUID="5f266627-8962-4ce4-a41f-b15ce26489a1" /var/www/html ext4 defaults 0 0

20. Test the configuration and reload the daemon

         sudo mount -a

         sudo systemctl daemon-reload

21. Verify your setup by running df -h, output must look like this:

![output](./web_server_image/df%20-h2.png)
