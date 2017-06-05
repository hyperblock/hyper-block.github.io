---
title: Torwards Hardened Container Images
date: 5/25/2017
categories: [English, image]
tags: [container, image, hyperlayer]
keywords: container, docker, image, block, lvm, hyperlayer
---

Container images rely heavily on unstable technologies on file system.
We propose a block-level layered image system, which shares the same design 
of existing image system, but implemented in the level of block store,
namely lvm, instead of file system. All the dependant features are mature, 
stable, production-ready and maintained in mainline kernel of Linux.

<!--more-->

### The Image System of Container

Containers are becoming increasingly important in the era of cloud. 
More and more customers are interested in or already using containers, 
to ship and run their applications.

One of the key advantages of containers is the layered image system, 
which is based on storage technologies like aufs, btrfs, overlayfs, 
zfs, lvm and device mapper. There's a figure below, which is from the 
official docker site. We can see in it that there are various choices 
for different kinds of workloads, however, only one (lvm) of them is 
both production-ready and maintained in mainline kernel of Linux. 
Actually, lvm is the only advised productional choice for RHEL/CentOS 
series distributions, which is by far the most popular Linux distribution 
in production servers, especially in large enterprises.

![Storage Drivers](/images/driver-pros-cons.png)

LVM is a block level storage technology, however, the layered imaging 
system is designed with file system in mind, so there's a semantic gap 
between them: each layer in an image is constituted of modified files 
and meta-data, while each layer (snapshot) in LVM is constituted of 
modified blocks of data. In order to bridge the gap, the storage driver 
for lvm has to enumerate all the files in adjacent layers, and see whether 
each of them is changed or not. This is called `NaiveDiffDriver` in docker.

Besides, this design choice has some fundamental problems: 

1. file system features are quite complex to develop, which is why none of
the various file system drivers are ready today for productional evneironment;
2. some advanced file system features are not well supported by file 
system level drivers, like xattr, and thus SELinux; 
3. apps are not able to choose the type of file systems, even if they konw 
that a specific type fits much better than any other ones; 
4. file systems' meta data will get slower to read, as image layers increase; 
5. every first modification to large files has an extra long latency for 
copy-on-write of the file, which leads to quite inconsistent I/O performance.

### A Block-Level Image System

To address these problems,
we propose a new image system that is based on block-level storage system,
which supports all kinds of file systems, as well as their complete feature 
sets; and provides consistent I/O performance in all cases. It is 
based on long-established LVM, and we have created a new tool, called
`lvdiffer`, to create a block-level patch file between adjacent LVM thin 
volumes, as well as applying the patch to existing LVM thin volumes.

The patch file has a simple and serializable format as this:

```
HYPERLAYER/1.0
<key>: <value>
<key>: <value>
<key>: <value>
<key>: <value>

<offset> <length>
<data of $length*512 bytes>
<offset> <length>
<data of $length*512 bytes>
<offset> <length>
<data of $length*512 bytes>
...

```
In which:

1. keys and values are properties of the patch, consisting of letters (lower or 
upper), digits and underscore, but no leading digits;
2. offset and length are hex numbers with NO '0x' prefix, in unit of sector (512 bytes);
3. offsets are NOT required to be sorted;
4. new lines is a single '\n', without '\r';

For example:

```
HYPERLAYER/1.0
Parent: 68044d4a72dfcf15018cfa6b4baf89361913d93d
Author: Huiba Li <lihuiba@gmail.com>
Date: Thu May 4 15:34:37 2017 +0800 

0 8
<data of 4096 bytes>
800 80
<data of 65536 bytes>
...
```

This format is called *HyperLayer*. It‘s meant to be a generic diff/patch format 
for all block-level storage systems, including but not limited to LVM and/or QCoW2. 
With this kind of patch file, we are able to realize a block-level image system 
that is similar to docker's.




