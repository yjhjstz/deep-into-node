
## File System Implementation Principles

File systems are stored on disks, which are usually divided into multiple partitions, each of which can be an independent file system.

The 0th sector of the disk is the Master Boot Record (MBR), which is used to boot the computer.

At the end of the MBR is the partition table, which identifies the starting and ending addresses of each partition. One partition in the table is identified as the active partition. When the computer is booted, the BIOS reads and executes the MBR. The first thing the MBR does is to determine the active partition, read its first block, called the boot block, and execute it.

### A Common File System (Partition) Structure Diagram

![](filesystem.png)

- Superblock
  - At system startup, the Superblock is read, which contains important parameters of the file system, and multiple backups are usually made.
- i-bmap block
  - Used to manage inodes, identifying whether inodes are free or in use.
- d-bmap block
  - Used to manage disk blocks, identifying whether disk blocks are free or in use.

### File System Implementation
An important function of a file system is to record which disk blocks are used by a file and to manage disk blocks, identifying whether they are in use or free. There are several implementation ideas.

* Contiguous allocation
  * Store each file on adjacent disk blocks. For example, if the disk block size is 2KB and the file size is 200KB, 100 disk blocks are needed. If the disk block size is 4KB, only 50 disk blocks are needed.

    Since each file starts from a new disk block, if a file occupies only half the disk block size, the other half is wasted and cannot be used by other files. However, the implementation of contiguous allocation is relatively simple, only need to record the position and number of the first disk block of the file, and the reading is fast because only one seek is required, and there is no need for track-to-track and rotational delays afterwards.

    As time goes by, disk fragmentation is more serious. Because files are repeatedly written and deleted, it is easy to form holes on disk blocks.

* Linked allocation
  * Construct a disk block linked list for each file, with each block pointing to the next disk block of the file.

    Because it is a linked list, sequential reading is very fast, but random reading is very slow, and the pointer to the next disk block in each block occupies space, which means that the amount of data that can be stored in each disk block is no longer a power of 2.

    However, when programs read and write files, they generally read and write disks in powers of 2, which causes additional overhead because reading the data of one block requires reading two disk blocks.

* Inode node scheme
  * Use a special thing to record which disk blocks are used by each file. This special thing is called the inode data structure, which stores some attributes of the file and the disk blocks used by the file.

   The inode only exists in memory when the corresponding file is opened, so even if there are many files in the file system, as long as few files are opened, it will not occupy too much memory.

   The metadata and file data of the file are stored separately, that is, a file has two attributes: inode and data file.

   The disk block space for storing inodes is limited, and there is only one inode for each file. So when a file is relatively large, how to solve it? Have you thought of pointers and pointers to pointers in C language?

   One solution is to reserve some data blocks for storing pointers to disk blocks, rather than directly pointing to disk blocks.


### Implementation of Linux File System
The implementation of Unix/Linux file system uses the inode scheme. The file systems ext2 and ext3 are like this, and ext4 has made many optimizations compared to the previous two file systems, which will not be discussed in this book.

### Cooperation between File System and Operating System

The file system is part of the operating system. The operating system is relatively stable, but there are many file systems, such as ext2, ext3, ext4, ZFS, etc. How does the operating system be compatible with these different file systems to provide unified services to the outside world?

C++ and JAVA programmers are easy to think of using polymorphism to implement it. Yes, the operating system also adopts a similar idea. For different file systems, abstract the basic, conceptually supported data structures and interfaces that all file systems support, such as the basic operations on files and directories described above.

These unified abstract components are called the Virtual File System (VFS). As a kernel component, VFS provides file and file system-related interfaces for user space programs.

Users who use VFS can directly call file system calls such as write and read without considering what the underlying file system is.

Looking horizontally at their relationship, as shown in the figure below
![](read-write.png)

### Summary

The file system is an abstraction of the disk device, shielding the specific storage type, such as disk and memory.


