---
layout: post
title: "Docker vs VM"
---
![Process making a system call](/images/containers-vs-virtual-machines.jpeg)

## Docker

Docker is a container based technology and containers are just instances of 
OS-level virtualization, which is an operating system paradigm in which the kernel allows 
the existence of multiple isolated user space instances.

#### User Space vs Kernel Space
**User Space** refers to all of the code in an operating system that is outside of the kernel.
This means all applications, utilities, tools etc. 

For example when you pull a containerized linux image, you usually get a pre-packaged 
somewhat minimal user space, with some common utilities such as ``ls``, ``cat`` , ``grep`` ...

**Kernel Space** is where the kernel (i.e., the core of the operating system) 
executes (i.e., runs) and provides its services.
Kernel space can be accessed by user processes only through the use of **system calls**.

#### Back to containers
So containers basically package all application code + dependencies together.
We can have multiple containers running on the same host and share the same **Kernel Space** with
other containers, but each one of them will be an isolated **User Space** process.

***Note***: Despite sometimes being treated as virtual machines, unlike those, the kernel is the 
only layer of abstraction between the applications and the resources (RAM, Disk, External Devices)
that they need to access.

All processes, including containers, which are themselves, also processes make system calls:
![Process making a system call](/images/ScreenshotDocker.png)

![Process making a system call](/images/ScreenshotDocker2.png)

Some user space commands, and their mapping to system calls:
![Process making a system call](/images/ScreenshotDocker3.png)

#### Back to containers (again)

Due to this sharing, the containers are usually very small and very performant
when compared to a VM.

## Virtual Machine

VM is a system that pretends to act exactly as an abstraction of physical hardware.
The hypervisor allows multiple VMs to run on a single physical machine.
Each VM includes a full copy of an operating system + all business applications 
and their dependencies.
They indeed take more space, and are usually slower to start.

## A (very small and resumed) comparison
***Architecture***

Docker containers usually are used in order to run multiple applications, in somewhat hermetic(if needed)
environment, whereas, VM are usually used if the services are required to run on different OS.

***Security***

In terms of security, one needs to keep in mind that, as containers share the same kernel space,
one infected application with elevated permissions can cause serious damage.

***Portability***

In terms of portability, of course, by being self-contained, and not having a full guest-OS, the
dockerized applications can be easily deployed across different platforms.

But, a good question is, will the containerized applications of today will run in the hosts of 2030?
Will the user-space of today still be compatible?

***Performance***

By the reasons stated already in this post usually the containers are less resource intensive.

### References

[Architecting Containers Part 1: Why Understanding User Space vs. Kernel Space Matters ](https://www.redhat.com/en/blog/architecting-containers-part-1-why-understanding-user-space-vs-kernel-space-matters)

[OS-level_virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization)

[Kernel Space Definition](http://www.linfo.org/kernel_space.html)

[Difference between Docker Container and Virtual Machine (VM)](https://medium.com/devops-mojo/difference-between-docker-container-and-virtual-machine-comparision-between-docker-and-vm-697330d0761c#:~:text=Virtual%20machines%20have%20host%20OS%20and%20the%20guest%20OS%20inside%20each%20VM.&text=Docker%20containers%20are%20considered%20suitable,to%20run%20on%20different%20OS.)
