# `mmap` internals & why it sucks in DBMS

> DBMS always knows more than the OS.

In this article, I will show you how `mmap` works in the OS (**O**perating **S**ystem) and why it is 💩 in DBMS (**D**ata**b**ase **M**anagement **S**ystem). I will **not** teach you how to use `mmap` in this blog or what virtual memory is, so it is **for people who know OS**.

Let's go to the business.

### `mmap` internals

`mmap` is somehow easy to implement. Take a look at [xv6: mmap](https://pdos.csail.mit.edu/6.S081/2022/labs/mmap.html), and you can even do it yourself (although a toy version). Anyway, let's start it.

Memory-mapped (mmap) file I/O is an OS-provided feature that maps the contents of a file on secondary storage into a program’s address space. We programmers can use `mmap` to do many things: garbage collection, shared memory, persistent storage, etc.

Behind all of those, it is the most important that `mmap` allows us to read/write files via a **direct pointer** to memory [in user space] (of course, illusion). There is **only one** data copy from disk to memory. But while using file I/O `read`/`write`, we need two times: disk to file system block cache [in kernel space] & block cache to memory [in user space]. Furthermore, `mmap` circumvents the cost of explicit `read`/`write` system calls.

Hence, you can regard `mmap` as a "bridge" between OS kernel and user space, while `read`/`write` as the "boat" transporting the data back and forth. The metaphor is not subtle, so I will tell you how it works in a real OS.

Assume we have a file "cidr.db", we will use `mmap` to read/write the file:

1. A user program calls `mmap` and gets a pointer to the VMA (**V**irtual **M**emory **A**rea)<br/>For the user program, the pointer is in its virtual memory and is the same as any other pointer. 
2.  In this segment, `mmap` only finds big enough virtual memory space and declares that VMA belongs to it.<br/>`mmap` is a system call, so the work is done in kernel space. OS initializes VMA struct for user process instead of allocating physical memory, or copying any data.
3. The user program attempts to write data to the file using the pointer.
4. CPU attempts to retrieve the page and write to physical memory.<br/>CPU tells MMU (**M**emory **M**anagement **U**nit) to find the *physical address* in the page table using a *virtual address*. But there is no such *physical address*, so it causes a *page fault* trap for OS.
5. OS knows it is a *page fault* and calls the *page fault handler* to deal with it.<br/>The handler copies the disk block to memory.
6. The handler also adds a mapping to the user page table.
7. CPU adds an entry in its TLB (**T**ranslation **L**ookaside **B**uffer) to accelerate future accesses.
8. Redo 4 to write the data to memory
9. The user program calls `munmap` to write the dirty page for persistence or just throw everything.

<img src="https://user-images.githubusercontent.com/70138429/194573687-33250251-c42e-4b7e-9db8-3639b724ca42.png" alt="how mmap accesses a file" style="zoom: 67%;" />

I have not covered many details, because it is very complex in real engineering.

In summary, `mmap` is almost the "best" tool to read/write files. But there are many shortcomings of `mmap`, which make it 💩 in DBMS.

### Why `mmap` sucks in DBMS?

> `mmap` and DBMS are like coffee and spicy food: an unfortunate combination that becomes obvious after the fact.

On the surface, `mmap` seems efficient and easy to handle. It reduces data copy, after all. On the other hand, the DBMS no longer needs to manage its own buffer pool. Therefore, DBMS developers are free to focus on other aspects of the system. Is this really the case?

After we dive into the water, the **dark side** of `mmap` is exposed. There are four problems with `mmap`: Transactional Safety, I/O Stalls, Error Handling, and Performance Issues. I will briefly introduce them.

* **Transactional Safety**: OS dominates when to flush a dirty page.<br/>When a flush occurs, OS never notifies anyone. It is a devastating blow to the DBMS transaction control. DBMSs who use `mmap` employ some complex protocols for this issue. (For further information, read the reference)
* **I/O Stalls**: OS dominates when to evict a page from memory.<br/>If OS evicts an upcoming page, the SQL query will encounter a blocking page fault. Furthermore, `mmap` does not support asynchronous reads, which also raises the I/O stall. `mmap` DBMS developers can use `mlock` or `madvise` partially mitigate its impact.
* **Error Handling**: OS would not validate the page.<br/>DBMS is responsible to ensure data integrity. When reading a page, buffer pool DBMS validates the page content with whatever approach. But `mmap` cannot do this.
* **Performance Issues**: Better performance, less money.<br/>`mmap` has serious bottlenecks that cannot be avoided without an OS-level redesign. Among the three issues below, TLB shootdowns can have a significant performance impact.<br/>(1) page table contention<br/>(2) single-threaded page eviction<br/>(3) TLB shootdowns: If OS evicts a page, it must also remove the page's mapping in **all cores**' TLBs. Whereas flushing the local TLB is inexpensive, issuing inter-processor interrupts to synchronize remote TLBs can take **thousands of cycles**.

Some of the problems can be overcome through careful implementation, but some cannot be solved without an OS-level redesign, especially **TLB shootdowns**.

Using `mmap` to manage database file I/O is a bad idea: it not only introduces much complexity but also has unsolvable performance limitations.

How to build a DBMS? Use lightweight buffer management techniques for file I/O.

### References

[1] [xv6 book: Chapter 4](https://pdos.csail.mit.edu/6.S081/2021/xv6/book-riscv-rev2.pdf) by MIT PDOS group

[2] [xv6 lab: mmap](https://pdos.csail.mit.edu/6.S081/2022/labs/mmap.html) by MIT PDOS group

[3] [Virtual Memory Primitives for User Programs](https://www.cs.princeton.edu/~appel/papers/vmpup.pdf) by Andrew W. Appel, Kai Li

[4] [`mmap` is 💩](https://www.cidrdb.org/cidr2022/papers/p13-crotty.pdf) by Andrew Crotty, Viktor Leis, Andy Pavlo

[5] [Buffer Pool](https://15445.courses.cs.cmu.edu/fall2022/project1/) by CMU 15-445