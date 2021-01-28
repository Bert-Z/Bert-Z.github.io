---
title: '[转载] DIRECT I/O'
date: 2021-01-28 06:34:02
tags: ['system']
categories:
cover: https://img-blog.csdnimg.cn/20200430035748976.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p5eG11c3Q=,size_16,color_FFFFFF,t_70
---
<meta name="referrer" content="no-referrer" />

# [转载] DIRECT I/O

Direct I/O is a feature of the file system whereby file reads and writes go directly from the applications to the storage device, bypassing the operating system read and write caches. Direct I/O is used only by applications (such as databases) that manage their own caches.

An application invokes direct I/O by opening a file with the `O_DIRECT` flag. Alternatively, GFS can attach a direct I/O attribute to a file, in which case direct I/O is used regardless of how the file is opened.

When a file is opened with `O_DIRECT`, or when a GFS direct I/O attribute is attached to a file, all I/O operations must be done in block-size multiples of 512 bytes. The memory being read from or written to must also be 512-byte aligned.

**Note**

Performing I/O through a memory mapping and also via direct I/O to the same file at the same time may result in the direct I/O being failed with an I/O error. This occurs because the page invalidation required for the direct I/O can race with a page fault generated through the mapping. This is a problem only when the memory mapped I/O and the direct I/O are both performed on the same node as each other, and to the same file at the same point in time. A workaround is to use file locking to ensure that memory mapped (i.e., page faults) and direct I/O do not occur simultaneously on the same file.

The Oracle database, which is one of the main direct I/O using applications, does not memory map the files to which it uses direct I/O and thus is unaffected. In addition, writing to a file that is memory mapped will succeed, as expected, unless there are page faults in flight at that point in time. The `mmap` system call on its own is safe when direct I/O is in use.

One of the following methods can be used to enable direct I/O on a file:

- `O_DIRECT`
- GFS file attribute
- GFS directory attribute

### 4.9.1. `O_DIRECT`

If an application uses the `O_DIRECT` flag on an `open()` system call, direct I/O is used for the opened file.

To cause the `O_DIRECT` flag to be defined with recent glibc libraries, define `_GNU_SOURCE` at the beginning of a source file before any includes, or define it on the **cc** line when compiling.