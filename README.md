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

In this lab, additional virtual disks were created and attached to the Kali Linux VM in VirtualBox to simulate working with multiple physical drives. Each new disk was provisioned as a dynamically allocated virtual disk with a virtual size of 2 GB.

The reason 2 GB was chosen is intentional. When working with Logical Volume Management (LVM), you need enough raw space to actually demonstrate realistic storage management tasks: creating multiple partitions, turning those partitions into physical volumes, and then grouping them into one volume group. If the disk is too small, you cannot meaningfully split it into multiple partitions and still have usable storage in each. Around 2 GB is a good lower bound for this project because it lets you do things like:
- Split one disk into multiple ~600–700 MB partitions
- Create another disk with a ~1 GB partition
- Add them all into a single logical pool and still have space left to extend a logical volume later

In other words, 2 GB is not an arbitrary number. It is large enough to support multiple partitions and LVM operations but still small enough that you are not wasting host storage.

When adding these disks in VirtualBox, they were created as dynamically allocated. This means VirtualBox reserves a “Virtual Size,” but it does not immediately consume that full amount on the host system. Instead, it only consumes “Actual Size,” which grows as the guest OS actually writes data to the disk.

- Virtual Size is the maximum capacity the VM sees. In this case, for each new virtual hard drive, that is 2 GB
- Actual Size is how much space is currently being used on the physical host to store the contents of that drive. On a new, empty drive this is only a few megabytes

This has two practical implications for anyone repeating this project:
1. You can safely create multiple virtual disks at sizes that look “big” on paper without immediately filling your host machine’s real storage
2. You should not be afraid to over-provision slightly in VirtualBox. Giving yourself a 20 GB root disk and a few extra 2 GB practice disks does not mean you are instantly losing 26 GB of laptop SSD. You only pay for what is actually written inside the VM

This setup models a real multi-drive or multi-LUN Linux server, which is important when practicing LVM because LVM is most useful in environments that do not rely on a single physical disk.

<img width="1915" height="937" alt="image" src="https://github.com/user-attachments/assets/1fd9f58d-1af6-4b1f-8591-a6d7ca6444f4" />

---

## **Step 2: Verifying Disk Attachments in Linux**

The lsblk command lists all available storage devices and their partitions. Newly added disks appear as `/dev/sdb` and `/dev/sdc`, confirming successful attachment.  

<img width="502" height="235" alt="image" src="https://github.com/user-attachments/assets/21ba9748-6819-45da-893f-bf30a6c79db5" />

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
