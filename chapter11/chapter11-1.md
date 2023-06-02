
## File Abstraction

The fs module is a file operation encapsulation that provides file reading, writing, renaming, deleting, directory traversal, and linking POSIX file system operations. Unlike other modules, all operations in the fs module provide both asynchronous and synchronous versions, such as the asynchronous method of reading file content function: readFile(), and the synchronous method readFileSync().

### Everything is a file
"Everything is a file" is one of the basic philosophies of Unix/Linux. Not only ordinary files, but also directories, character devices, block devices, sockets, etc. are treated as files in Unix/Linux; although they have different types, they provide the same set of operation interfaces.

![](unix-file.jpg)

A file is an abstract mechanism that abstracts the disk.

A file is a sequence of bytes, and each I/O device, including disks, keyboards, displays, and even networks, can be abstracted as a file. In the Unix/Linux system, all input and output are completed by calling the IO system call.

The file is an abstraction of IO, just as virtual memory is an abstraction of program storage, and the process is an abstraction of a running program. These are all important abstractions of the operating system.

The most important feature of the abstraction mechanism is the naming of the managed object, so the file has a file name, and the file name must comply with certain specifications.

### Main file operations
- open
- read
- write
- close

The above operations are relatively simple, so I won't go into detail. I will introduce the important operations of reading files, writing files, and refreshing data in the article later. If you are interested, you can use the command man 2 read to view the help documentation.

### File types
You can use ls -l to view the file type, mainly the following common ones.
- Ordinary files
  - Including text files and binary files
- Directory
  - Compared with ordinary files, directories are also stored on media, but directories do not store regular files. They are only used to organize and manage files.
- Proc file
  - Proc does not store, so it does not occupy any space. Proc allows the kernel to generate information related to system status and configuration. This information can be read by users and the system kernel from ordinary files without special tools.
There are many other file types that can be viewed through man ls.

### File attributes
File attributes include file permission information, creation time, last modification time, last read time, file size, file reference number and other information, and these file attributes are also called file metadata.

### Advanced file system reading and writing
#### File mapping mmap

`man 2 mmap` to view:

```c++

#include <sys/mman.h>

void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);

```
Through the mmap system call, a file is mapped to the process virtual address space. That is to say, the file on the disk is now a memory array in the system, so the application program does not need to call the system IO call to access the file, but directly reads the memory.

#### Advantages: 
* 1. Read and write from memory-mapped files to avoid unnecessary copying of read and write. 
* 2. Read and write from memory-mapped files to avoid unnecessary system calls and user-kernel mode switching. 
* 3. Multiple processes can share memory-mapped files.

#### Disadvantages: 
* 1. Memory mapping requires an integer multiple of the page size. If the file is small, it will waste memory. 
* 2. Memory mapping requires process address space. Large memory mapping may cause address space fragmentation and cannot find enough large free contiguous areas for other uses. 


#### Discrete I/O 
The readv and writev functions allow us to read from or write to multiple discontinuous buffers in a single function call. These operations are called scatter read and gather write.

```c++

#include <sys/uio.h>

ssize_t readv(int filedes, const struct iovec *iov, int iovcnt);

ssize_t writev(int filedes, const struct iovec *iov, int iovcnt);

Both return the number of bytes read or written, and -1 on error.
```
The second parameter of these two functions is a pointer to an iovec structure array:
```c++
struct iovec {
  void   *iov_base;      /* starting address of buffer */
  size_t iov_len;        /* size of buffer */
};
```
The number of elements in the iov array is indicated by iovcnt. Its maximum value is limited by IOV_MAX.
![](14fig27.gif)

writev outputs data from the buffer in order of iov[0], iov[1] to iov[iovcnt-1]. writev returns the total number of output bytes, usually equal to the sum of the lengths of all buffers.

readv scatters the read data into the buffer in the same order. readv always fills one buffer first, and then fills the next one. readv returns the total number of bytes read. If there is no data to read because the end of the file has been reached, it returns 0.

### Summary
* Zero-copy technology can reduce the number of data copies and shared bus operations, eliminate unnecessary intermediate copy times for transferring data between storage, and effectively improve data transfer efficiency. Moreover, zero-copy technology reduces the overhead of user application address space and operating system kernel address space caused by context switching.

It is an important optional optimization method that user programs try to optimize.

* Vector I/O operations can replace multiple linear I/O operations, with better performance

 - In addition to reducing the number of initiated system calls, vector I/O can provide better performance
