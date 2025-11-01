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

In VirtualBox, additional virtual disks were created and attached to the Kali Linux VM. Each disk was configured with a **2 GB dynamic allocation** to simulate multiple storage devices.  

- **Virtual Size** represents the total space allocated to the virtual disk.  
- **Actual Size** is the physical disk usage on the host system.  

<img width="1569" height="768" alt="image" src="https://github.com/user-attachments/assets/e7d41afe-4d1f-4c13-902d-ede30d532eaa" />

---

## **Step 2: Verifying Disk Attachments in Linux**

The lsblk command lists all available storage devices and their partitions. Newly added disks appear as `/dev/sdb` and `/dev/sdc`, confirming successful attachment.  

<img width="502" height="235" alt="image" src="https://github.com/user-attachments/assets/d9d666aa-ce6a-473b-a7b4-ecd36a292651" />

---

## **Step 3: Partitioning the Disks Using Fdisk**

fdisk is a command-line utility used to create and manage disk partitions. Here, `/dev/sdb` was divided into three partitions (`sdb1`, `sdb2`, `sdb3`), each approximately 682 MB in size.  

<img width="1002" height="400" alt="image" src="https://github.com/user-attachments/assets/5ea97c5e-5c8e-480d-a937-35aaf0a547ae" />

After creating each partition, the **w command** writes the changes to the disk’s partition table.  

<img width="911" height="348" alt="image" src="https://github.com/user-attachments/assets/a54de95c-b16f-45f7-a54e-5ed0e037a903" />

The same process was performed on `/dev/sdc`, creating a single **954 MB partition (`sdc1`)**.  

<img width="942" height="576" alt="image" src="https://github.com/user-attachments/assets/f8179064-878f-400a-9c72-75faebff2c59" />

Running lsblk again confirms all partitions were successfully created.  

<img width="507" height="325" alt="image" src="https://github.com/user-attachments/assets/d0fad887-674a-4638-be52-05634b65922c" />

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

<img width="568" height="209" alt="image" src="https://github.com/user-attachments/assets/85276594-0524-4f13-85ef-cf95b1aa4ff3" />

---

## **Step 5: Creating a Volume Group (VG)**

### Definition:
A **Volume Group (VG)** combines multiple physical volumes into a single pool of storage. Logical volumes are created from this shared pool, allowing flexible allocation of disk space.  

The vgcreate command was used to create a volume group named **my_vg** from the four physical volumes:  

X
sudo vgcreate my_vg /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdc1
X

<img width="718" height="108" alt="image" src="https://github.com/user-attachments/assets/26512b50-6145-4862-9718-9db4475d6239" />

Verification with vgdisplay confirms the VG was successfully created, totaling **2.92 GiB** in size.  

<img width="718" height="521" alt="image" src="https://github.com/user-attachments/assets/d7414f94-4c1d-46d9-9e50-ba219ac0d6e4" />

---

## **Step 6: Creating a Logical Volume (LV)**

### Definition:
A **Logical Volume (LV)** acts like a partition within the volume group, but it is more flexible. It can be resized, extended, or reduced without directly affecting the physical disks underneath.  

The command below creates a logical volume named `my_lv` of size 2 GB within the `my_vg` volume group:  

X
sudo lvcreate -L 2G -n my_lv my_vg
X

- `-L` specifies the logical volume size  
- `-n` specifies the logical volume name  

<img width="440" height="81" alt="image" src="https://github.com/user-attachments/assets/62d37232-5beb-4a55-b454-2df8126299a5" />

---

## **Step 7: Extending Logical Volume Capacity**

lvextend increases the size of an existing logical volume without destroying its data.  

X
sudo lvextend -L +500MB /dev/my_vg/my_lv
X

- `-L` specifies how much to extend the LV  
- `+500MB` indicates that 500 MB should be added to the current size  

<img width="1117" height="116" alt="image" src="https://github.com/user-attachments/assets/c3ba9742-c1d4-47d2-a1ab-d0ba35573e6e" />

The logical volume’s size increased from **2.00 GiB to 2.49 GiB**, confirming dynamic expansion.

---

## **Step 8: Verifying the Configuration**

Final verification combines multiple LVM and filesystem commands to review the configuration.  

X
sudo pvs
sudo vgs
sudo lvs
lsblk
df -hT
X

- pvs, vgs, and lvs display physical, volume group, and logical volume information respectively.  
- lsblk shows the hierarchy of devices and partitions.  
- df -hT lists mounted filesystems with size and type.  

<img width="925" height="768" alt="image" src="https://github.com/user-attachments/assets/1bae8aa3-b58d-4d2f-98f3-48a31dcaddf7" />

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
















---

# Linux-Logical-Volume-Management-LVM-Configuration
This project demonstrates configuring Linux Logical Volume Management (LVM) to create, organize, and resize storage volumes. It highlights practical skills in disk preparation, volume grouping, and logical volume management for efficient Linux storage administration and improved system scalability.

Creating extra disks in VirtualBox to do this project with:
<img width="1915" height="937" alt="image" src="https://github.com/user-attachments/assets/1fd9f58d-1af6-4b1f-8591-a6d7ca6444f4" />

What we start with:
<img width="502" height="235" alt="image" src="https://github.com/user-attachments/assets/21ba9748-6819-45da-893f-bf30a6c79db5" />

Using fdisk to create the first partition of 300MB:
<img width="1002" height="400" alt="image" src="https://github.com/user-attachments/assets/f804e08e-db49-45f1-b854-a8580ee231c7" />

Creating the other 2 identical partitions and writing it to the partition table:
<img width="911" height="348" alt="image" src="https://github.com/user-attachments/assets/9e59d7ae-4b6c-46ba-941f-71d07a7a2eb5" />

Creating 1GB partition for drive C:
<img width="942" height="576" alt="image" src="https://github.com/user-attachments/assets/641ba7b6-4669-4261-8a1e-b0cc6efe59b7" />

Rerunning lsblk (Notice even though I did +1GB it only says theres 954MB available
<img width="507" height="325" alt="image" src="https://github.com/user-attachments/assets/f0cfd134-9216-432c-bbfa-9e6dbde4959a" />

Adding partitions to LV manager with pvcreate
<img width="568" height="209" alt="image" src="https://github.com/user-attachments/assets/a63edac8-d831-4398-b70c-acc18928f48d" />

Use of vgcreate command showing that we run it on all the partitions we created with pvcreate specifically
<img width="718" height="108" alt="image" src="https://github.com/user-attachments/assets/8efba53f-6adb-499d-bad6-3905b6a4c80d" />

Use of vgdisplay
<img width="718" height="521" alt="image" src="https://github.com/user-attachments/assets/105f0851-54b4-4bbe-ad4e-2f69956b63a0" />

Use of lvcreate with -L and -n options
<img width="440" height="81" alt="image" src="https://github.com/user-attachments/assets/0dacabc6-aac0-4217-8cd5-3a8397243eb1" />

Use of lvextend to expand it 500MB (Keep it mind of the total space you are able to use)
<img width="1117" height="116" alt="image" src="https://github.com/user-attachments/assets/e6e84491-624f-4d52-bb70-cd65d7f713e2" />

Use sudo pvs; sudo vgs; sudo lvs; lsblk; df -hT
<img width="997" height="827" alt="image" src="https://github.com/user-attachments/assets/cec8e56d-b399-43c2-a6ff-70897cf913a7" />

