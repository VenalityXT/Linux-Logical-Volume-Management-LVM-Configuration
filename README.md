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

<img width="502" height="235" alt="image" src="https://github.com/user-attachments/assets/21ba9748-6819-45da-893f-bf30a6c79db5" />

The output shows the main system disk (`/dev/sda`) along with the newly added drives (`/dev/sdb` and `/dev/sdc`). Their appearance here confirms that the virtual hardware was properly attached with 2GB and is ready for partitioning in the next step.

---

## **Step 3: Partitioning the Disks Using Fdisk**

fdisk is a command-line utility used to create and manage disk partitions. Here, `/dev/sdb` was divided into three partitions (`sdb1`, `sdb2`, `sdb3`), each approximately 682 MB in size.  

<img width="1002" height="400" alt="image" src="https://github.com/user-attachments/assets/f804e08e-db49-45f1-b854-a8580ee231c7" />

After creating each partition, the **w command** writes the changes to the disk’s partition table.  

<img width="942" height="576" alt="image" src="https://github.com/user-attachments/assets/641ba7b6-4669-4261-8a1e-b0cc6efe59b7" />

The same process was performed on `/dev/sdc`, creating a single **954 MB partition (`sdc1`)**.  

<img width="507" height="325" alt="image" src="https://github.com/user-attachments/assets/f0cfd134-9216-432c-bbfa-9e6dbde4959a" />

Running lsblk again confirms all partitions were successfully created.  

-

---

## **Step 4: Initializing Physical Volumes (PVs)**

### Definition:
A **Physical Volume (PV)** is the lowest-level storage unit in LVM, typically corresponding to a disk partition or entire disk that LVM can use for volume grouping.  

Each partition was initialized as a physical volume using the pvcreate command:  

X
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdb2
sudo pvcreate /dev/sdb3
sudo pvcreate /dev/sdc1
X

The command writes LVM metadata to the specified partitions, preparing them for use in a **Volume Group (VG)**.  

Verification with `sudo lvmdiskscan -l` shows the four recognized LVM physical volumes.  

<img width="568" height="209" alt="image" src="https://github.com/user-attachments/assets/a63edac8-d831-4398-b70c-acc18928f48d" />

---

## **Step 5: Creating a Volume Group (VG)**

### Definition:
A **Volume Group (VG)** combines multiple physical volumes into a single pool of storage. Logical volumes are created from this shared pool, allowing flexible allocation of disk space.  

The vgcreate command was used to create a volume group named **my_vg** from the four physical volumes:  

```bash
sudo vgcreate my_vg /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdc1
```

<img width="718" height="108" alt="image" src="https://github.com/user-attachments/assets/8efba53f-6adb-499d-bad6-3905b6a4c80d" />

Verification with vgdisplay confirms the VG was successfully created, totaling **2.92 GiB** in size.  

<img width="718" height="521" alt="image" src="https://github.com/user-attachments/assets/105f0851-54b4-4bbe-ad4e-2f69956b63a0" />

---

## **Step 6: Creating a Logical Volume (LV)**

### Definition:
A **Logical Volume (LV)** acts like a partition within the volume group, but it is more flexible. It can be resized, extended, or reduced without directly affecting the physical disks underneath.  

The command below creates a logical volume named `my_lv` of size 2 GB within the `my_vg` volume group:  

```bash
sudo lvcreate -L 2G -n my_lv my_vg
```

- `-L` specifies the logical volume size  
- `-n` specifies the logical volume name  

<img width="440" height="81" alt="image" src="https://github.com/user-attachments/assets/0dacabc6-aac0-4217-8cd5-3a8397243eb1" />

---

## **Step 7: Extending Logical Volume Capacity**

lvextend increases the size of an existing logical volume without destroying its data.  

```bash
sudo lvextend -L +500MB /dev/my_vg/my_lv
```

- `-L` specifies how much to extend the LV  
- `+500MB` indicates that 500 MB should be added to the current size  

<img width="1117" height="116" alt="image" src="https://github.com/user-attachments/assets/e6e84491-624f-4d52-bb70-cd65d7f713e2" />

The logical volume’s size increased from **2.00 GiB to 2.49 GiB**, confirming dynamic expansion.

---

## **Step 8: Verifying the Configuration**

Final verification combines multiple LVM and filesystem commands to review the configuration.  

```bash
sudo pvs
sudo vgs
sudo lvs
lsblk
df -hT
```

- pvs, vgs, and lvs display physical, volume group, and logical volume information respectively.  
- lsblk shows the hierarchy of devices and partitions.  
- df -hT lists mounted filesystems with size and type.  

<img width="997" height="827" alt="image" src="https://github.com/user-attachments/assets/cec8e56d-b399-43c2-a6ff-70897cf913a7" />

This confirms all volumes are active and functioning correctly.

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
