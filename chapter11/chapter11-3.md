 
## Asynchronous I/O (AIO) API

The Asynchronous I/O (AIO) API is a relatively new enhancement provided in the Linux kernel. It is a standard feature of the 2.6 version kernel. The basic idea behind AIO is to allow processes to initiate many I/O operations without blocking or waiting for any operation to complete. Later, or when notified of the completion of an I/O operation, the process can retrieve the result of the I/O operation.

### I/O Models

Before delving into the AIO API, let's explore the different I/O models available on Linux. This is not an exhaustive introduction, but we will attempt to introduce the most commonly used models to explain their differences with asynchronous I/O. Figure 1 shows the synchronous and asynchronous models, as well as the blocking and non-blocking models.

![](figure1.gif)

Each I/O model has its own usage pattern, and they have their own advantages for specific applications. This section will briefly introduce each of them.

### Synchronous Blocking I/O

The most commonly used model is the synchronous blocking I/O model. In this model, the user-space application executes a system call, which causes the application to block. This means that the application will remain blocked until the system call completes (data transfer is complete or an error occurs). The calling application is in a state where it no longer consumes CPU and only waits for a response, so from a processing perspective, this is very efficient.
Figure 2 shows the traditional blocking I/O model, which is currently the most commonly used model in applications. When the read system call is called, the application blocks and performs a context switch to the kernel. Then, the read operation is triggered, and when the response returns (from the device we are reading from), the data is copied to the user-space buffer. Then the application unblocks (the read call returns).

![](figure2.gif)

From the application's perspective, the read call will last a long time. In fact, the application is blocked while the kernel performs the read operation and other work.

### Synchronous Non-Blocking I/O

A slightly less efficient variant of synchronous blocking I/O is synchronous non-blocking I/O. In this model, the device is opened in a non-blocking manner. This means that the I/O operation does not complete immediately, and the read operation may return an error code indicating that the command cannot be immediately satisfied (EAGAIN or EWOULDBLOCK), as shown in Figure 3.

![](figure3.gif)

The implementation of non-blocking is that the I/O command may not be immediately satisfied, and the application must call many times to wait for the operation to complete. This may not be efficient because in many cases, when the kernel executes this command, the application must be busy waiting until the data is available, or trying to perform other work. As shown in Figure 3, this method can introduce delays in I/O operations because there is a certain interval between when data becomes available in the kernel and when the user calls read to return data, which can lead to a decrease in overall data throughput.

### Asynchronous Blocking I/O

Another blocking solution is non-blocking I/O with blocking notifications. In this model, non-blocking I/O is configured, and then the blocking select system call is used to determine when an I/O descriptor has an operation. What makes the select call interesting is that it can be used to provide notifications for multiple descriptors, not just one descriptor. For each prompt, we can request notifications for whether this descriptor can write data, whether there is read data available, and whether an error has occurred.

![](figure4.gif)

The functionality provided by the select function (asynchronous blocking I/O) is similar to AIO. However, it blocks notification events rather than I/O calls.

### Asynchronous Non-Blocking I/O

The asynchronous non-blocking I/O model is a model for handling overlapping I/O. The read request returns immediately, indicating that the read request has been successfully initiated. When the read operation is completed in the background, the application then performs other processing operations. When the response to the read arrives, a signal is generated or a thread-based callback function is executed to complete the I/O processing.

![](figure5.gif)

The ability to overlap computational operations and I/O processing in a process to execute multiple I/O requests utilizes the difference between processing speed and I/O speed. When one or more I/O requests are suspended, the CPU can perform other tasks; or more commonly, while initiating other I/O, it can operate on completed I/O.

### Summary

How slow IO devices and fast CPUs collaborate is a science.
But this does not mean that synchronous blocking IO is always bad, it still needs to be flexibly selected according to the scene.

According to the author's experience, synchronous IO is suitable for scenarios with controllable time and a small number of calls.
Asynchronous IO is suitable for scenarios with uncontrollable time (such as network anomalies) and a large number of calls.

### Reference

- https://www.ibm.com/developerworks/cn/linux/l-async/

