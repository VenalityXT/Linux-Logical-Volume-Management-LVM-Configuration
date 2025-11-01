# **Linux Logical Volume Management (LVM) Configuration**

[![Linux](https://img.shields.io/badge/OS-Linux-blue?logo=linux)](https://www.linux.org/)
[![Storage](https://img.shields.io/badge/Focus-Storage%20Administration-orange)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[![VirtualBox](https://img.shields.io/badge/Platform-VirtualBox-yellow?logo=virtualbox)](https://www.virtualbox.org/)
[![File System](https://img.shields.io/badge/Topic-File%20System%20Management-green)](https://tldp.org/HOWTO/LVM-HOWTO/)

---

## **Project Overview**
This project demonstrates the setup and configuration of Linux Logical Volume Management (LVM) within a virtualized environment. The lab uses **VirtualBox** and **Kali Linux** to simulate disk management operations such as creating partitions, initializing physical volumes, grouping them into a logical pool, and dynamically resizing storage. The purpose of this exercise is to develop proficiency in flexible storage administration, a core concept in Linux systems management.

---

## **Objectives**
- Configure and manage multiple virtual disks in a Linux environment  
- Partition and prepare disks using fdisk  
- Initialize and manage **Physical Volumes (PVs)**  
- Create and configure a **Volume Group (VG)** and **Logical Volume (LV)**  
- Demonstrate dynamic resizing using lvextend  
- Verify configuration using pvs, vgs, lvs, lsblk, and df -hT

---

## **Step 1: Creating Virtual Disks in VirtualBox**

To simulate working with multiple physical drives (and to avoid accidentally wiping my boot volume... again), two additional **2 GB virtual disks** were added to the VM using VirtualBox. These disks were configured as **dynamically allocated**, which we'll circle back to later.

The **2 GB size** was chosen intentionally as it’s the minimum practical amount for Logical Volume Management (LVM). Smaller disks make partitioning and metadata allocation difficult, while 2 GB provides enough space to perform key LVM operations effectively.  

This amount of space allows us to:
- Create multiple partitions on one drive  
- Combine them into a volume group  
- Leave room for future extensions or resizing tests  

### Understanding Virtual Size vs Actual Size
- **Virtual Size**: The total capacity the VM perceives (e.g., 2 GB or 200 GB).  
- **Actual Size**: The current disk space currently being utilized on the host.

Because of this dynamic allocation, you can safely allocate larger virtual disks for testing without immediately using that same amount of physical storage on your host. This allows flexible experimentation with multiple drives and configurations while keeping your system efficient.

> This setup actually mirrors a real multi-disk environment where admins manage multiple storage devices instead of one monolithic disk!
<img width="1915" height="937" alt="image" src="https://github.com/user-attachments/assets/1fd9f58d-1af6-4b1f-8591-a6d7ca6444f4" />

> You can think of it like potential and and kinetic energy for my physics nerds out there!

---

### **Verifying Disk Attachments in Linux**

The `lsblk` command lists all available block storage devices connected to the system. It displays information such as the device name, size, type, and mount point, helping identify which disks are currently in use and which are available for configuration. Here’s a screenshot of the output for reference:

<img width="498" height="234" alt="image" src="https://github.com/user-attachments/assets/a6a22e61-3e0b-4020-aab7-0d11694b16f1" />

The output shows the main system disk (`/dev/sda`) along with the newly added drives (`/dev/sdb` and `/dev/sdc`). Their appearance here confirms that the virtual hardware was properly attached with 2GB and is ready for partitioning in the next step.

---

## **Step 2: Partitioning the Disks Using Fdisk**

To prepare the virtual disks for use with LVM, they must first be divided into smaller sections called **partitions**. Partitioning allows us to separate physical space on a disk into logical segments that can be individually managed, formatted, or combined later in a volume group.

The **fdisk** utility is a command-line tool used to create and manage disk partitions in Linux. It supports both MBR (Master Boot Record) and GPT (GUID Partition Table) formats, though for this project, we’re using the simpler **MBR** layout since the goal is to practice foundational LVM concepts, not modern bootloader configurations.

Here, the command below was used to create partitions on the `/dev/sdb` disk:

```bash
sudo fdisk /dev/sdb
```

Inside fdisk:
- Type **n** to create a new partition  
- Choose **p** for a primary partition  
- Press **Enter** to accept the default partition number and first sector  
- Specify the size (for example, `+682MB` for a 682 MB partition)  
- Repeat this process to create three equal partitions on `/dev/sdb`  
- Finally, use **w** to write changes to the partition table  

> Think of partitions as the Lego bricks that LVM is built upon; stack them right, and everything else just clicks together (Prof M ☝️)

<img width="1002" height="400" alt="image" src="https://github.com/user-attachments/assets/ef64a626-9250-4696-8944-7530a3127264" />

After creating the three partitions (`sdb1`, `sdb2`, and `sdb3`), the same process was performed on `/dev/sdc` to create a single **954 MB partition** (`sdc1`):

```bash
sudo fdisk /dev/sdc
```

Once both disks were partitioned, the `lsblk` command was used to verify the new layout and confirm that all four partitions were successfully created and ready for LVM initialization.

<img width="508" height="327" alt="image" src="https://github.com/user-attachments/assets/ffa92dae-c603-4fa7-8c0e-0a7a1ea60ffb" />

> The output shows `/dev/sdb` split into three partitions and `/dev/sdc` with one, providing four total partitions that will be converted into LVM physical volumes in the next step.

Now I'm wondering why the partition sizes aren’t exactly what we entered for the end sector? That’s because Linux measures in binary units (MiB/GiB) instead of decimal (MB/GB), so 1GB shows up as about 954MB!

---

## **Step 3: Initializing Physical Volumes (PVs)**

### Definition:
A **Physical Volume (PV)** is the lowest-level storage unit in LVM, typically corresponding to a disk partition or entire disk that LVM can use for volume grouping.  

Each partition was initialized as a physical volume using the pvcreate command:  

```bash
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdb2
sudo pvcreate /dev/sdb3
sudo pvcreate /dev/sdc1
```

The command writes LVM metadata to the specified partitions, preparing them for use in a **Volume Group (VG)**.  

Verification with `sudo lvmdiskscan -l` shows the four recognized LVM physical volumes.  

<img width="624" height="829" alt="image" src="https://github.com/user-attachments/assets/6176cf13-b0ea-48b4-8b6a-8c154fc06833" />

---

## **Step 4: Creating a Volume Group (VG)**

The next step is to combine all four partitions into a single shared storage pool using a **Volume Group (VG)**. A Volume Group is one of the core components of LVM; it unifies multiple **Physical Volumes (PVs)** into a single logical space that can later be divided into **Logical Volumes (LVs)** for actual use.  

To create the group, we use the `vgcreate` command:

```bash
sudo vgcreate my_vg /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdc1
```

This command initializes a new volume group named **my_vg**, combining the four physical partitions created earlier into one flexible pool of storage. If successful, it will output a confirmation with `Volume group "my_vg" successfully created`.

<img width="718" height="108" alt="image" src="https://github.com/user-attachments/assets/8efba53f-6adb-499d-bad6-3905b6a4c80d" />

To further verify the group was created, we can use the `vgdisplay` command. This provides detailed information about the new volume group such as its name, size, and available space.  

```bash
sudo vgdisplay
```

<img width="718" height="521" alt="image" src="https://github.com/user-attachments/assets/105f0851-54b4-4bbe-ad4e-2f69956b63a0" />

From this output, we can see:  
- **VG Name** – our volume group is named **my_vg**  
- **VG Size** – the total combined size of all four partitions is about **2.92 GiB**  
- **PE Size (Physical Extent)** – each extent is 4 MiB, meaning storage is managed in 4 MB chunks  
- **Free PE / Size** – all extents are currently unallocated, ready for logical volumes  

> Think of a Volume Group as the “warehouse” that stores all your available space, while Logical Volumes are the shelves you’ll build next to organize and use it.

---

## **Step 5: Creating a Logical Volume (LV)**

Now that we’ve built our Volume Group, it’s time to carve out a portion of that space into something usable like a **Logical Volume (LV)**.  
Think of this like setting up a “virtual partition” inside your warehouse (the VG) where actual data will live.

A **Logical Volume** behaves much like a normal disk partition, but with far more flexibility. It can be resized, extended, or even moved across drives without affecting the underlying physical disks.

To create a new logical volume named **my_lv** with a size of **2 GB** from our **my_vg** volume group, use the following command:

```bash
sudo lvcreate -L 2G -n my_lv my_vg
```

Here’s what each flag does:
- `-L` specifies the size of the logical volume  
- `-n` specifies the name to assign to it  

<img width="440" height="81" alt="image" src="https://github.com/user-attachments/assets/0dacabc6-aac0-4217-8cd5-3a8397243eb1" />

At this point, we’ve officially created our first virtual partition within the LVM structure! A fully functional storage space that can grow, shrink, or move dynamically as our system evolves. Pretty cool, right?

---

## **Step 6: Extending Logical Volume Capacity**

One of the biggest advantages of **LVM** over traditional partitioning is flexibility, you can increase storage on the fly without losing data or reformatting anything. That’s exactly what we’ll do here using the **lvextend** command.  

The following command expands the existing logical volume (*my_lv*) by **500 MB**, extending its total capacity beyond the original 2 GB allocation:

```bash
sudo lvextend -L +500MB /dev/my_vg/my_lv
```

Here’s the breakdown:
- `-L` specifies the size change  
- `+500MB` tells LVM to add (not replace) 500 MB to the current volume size  

<img width="1117" height="116" alt="image" src="https://github.com/user-attachments/assets/e6e84491-624f-4d52-bb70-cd65d7f713e2" />

> As shown above, the logical volume **my_lv** successfully expanded from **2.00 GiB** to **2.49 GiB**.  

You can dynamically scale storage to meet your system’s needs without downtime or data loss. Try doing that with a static partition!

---

## **Step 7: Verifying the Configuration**

With everything configured, the final step is to verify that our Logical Volume setup works as intended. To make the process efficient, we can chain multiple commands together using semicolons (`;`).  
This lets us execute several commands in a single line, perfect for quickly displaying all key LVM and disk information at once.

```bash
sudo pvs; sudo vgs; sudo lvs; lsblk; df -hT
```

Here’s what each command does:
- **pvs** — Displays information about **Physical Volumes (PVs)**, showing which devices are part of LVM and how much space they contribute.  
- **vgs** — Summarizes the **Volume Group (VG)** details such as total size, free space, and number of physical volumes.  
- **lvs** — Lists all **Logical Volumes (LVs)** along with their attributes, size, and associated volume groups.  
- **lsblk** — Provides a hierarchical view of block devices, making it easy to see how partitions, LVM layers, and volumes are connected.  
- **df -hT** — Displays mounted filesystems, their types, and usage in a human-readable format.

<img width="997" height="827" alt="image" src="https://github.com/user-attachments/assets/cec8e56d-b399-43c2-a6ff-70897cf913a7" />

From the output, we can confirm:
- Each **Physical Volume** (*sdb1, sdb2, sdb3, sdc1*) was successfully added to the Volume Group **my_vg**.  
- The Volume Group (*my_vg*) reports a total size of **2.92 GiB** with **444 MB free**, aligning with our earlier calculations.  
- The Logical Volume (*my_lv*) shows as **2.49 GiB**, matching the 2 GB initial allocation plus the 500 MB extension from Step 6.  
- The `lsblk` output visually maps how the LV sits atop multiple partitions across disks **sdb** and **sdc**.  
- Finally, `df -hT` verifies that all filesystems are intact, confirming there are no mount or read errors after resizing.

This full verification step ties everything together by proving that the Logical Volume, Volume Group, and Physical Volumes are all linked properly and actively managed by LVM. 
In short, we built a flexible, scalable storage architecture from the ground up!

---

## **Cybersecurity Implications**

Understanding Logical Volume Management is important for cybersecurity professionals working in system hardening, forensics, and server management. LVM plays a critical role in:  
- **Data recovery and integrity** – Analysts can clone or snapshot volumes for forensic investigation.  
- **Secure storage allocation** – Administrators can isolate sensitive data on dedicated logical volumes.  
- **System resilience** – Enables quick recovery from disk failures or capacity exhaustion.  
- **Incident response** – Knowledge of LVM helps investigators understand where data resides and how it’s structured, especially in compromised systems.

---

## **Summary**

This lab demonstrated the complete process of configuring Logical Volume Management on a Linux system. It included creating and partitioning disks, initializing physical volumes, combining them into a volume group, and allocating logical volumes. The project also showed how to dynamically extend storage and verify system configuration using LVM utilities.  

This foundational knowledge reinforces critical system administration and cybersecurity skills such as secure data management, disk recovery, and scalable storage planning.
