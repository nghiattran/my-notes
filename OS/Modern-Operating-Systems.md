# Modern Operating Systems

---
## Chapter 2


### 2.3 INTERPROCESS COMMUNICATION

Three issues with **InterProcess Communication** (**IPC**):

1. How one process can pass information to another.
2. Making sure two or more processes do not get in each other’s way.
3. Proper sequencing

#### 2.3.3 Mutual Exclusion with Busy Waiting

##### The Test and Set Lock (TSL) Instruction

```
TSL RX,LOCK
```

1. Read the content at the memory address of lock into register `RX`.
2. Store a non-zero value at the memory address of `LOCK`.

Locking the memory bus is very different from disabling interrupts.

**Disabling interrupts** on processor 1 has no effect at all on processor 2.

**Locking memory bus** does not allow on processors to access the memory.

Use TSL to solve race condition:

1. When lock = 0, any process may set lock = 1 by using TSL instruction and go to its critical section.
2. When the process finish its critical section, set lock = 0 using the original move instruction.

```
enter region:
	TSL REGISTER,LOCK 			// copy lock to register and set lock to 1
	CMP REGISTER,#0				// was lock zero?
	JNE enter region			// if it was not zero, lock was set, so loop
	RET 						// return to caller; critical region entered
leave region:
	MOVE LOCK,#0 				// store a 0 in lock
	RET 						// return to caller
```

#### Busy waiting problems

Both Peterson’s solution and the TSL solution can solve race condition but they use a tight loop waiting to get into critical section which is wasting CPU time and cause **Priority Inversion problem**. 

Priority Inversion problem with busy waiting method:

* A computer with two processes P0 with high priorities, and P1 with low priorities.
* The scheduling rules are such that P0 get CPU time whenever it is in ready state.
* At a time moment, P1 is in critical section.
* P0 become to ready state. P0 posse CPU time and start to run.
* Since P1 is in critical section, P0 runs busy waiting forever since P1 does not have a chance to get CPU time to finish its job.

#### 2.3.4 Sleep and Wakeup

Sleep and wakeup:

* Instead of busy waiting, it goes to sleeping state.
* Once a process finish its job in critical section, it calls wakeup function which allows one of sleeping process get into the critical section.

##### The Producer-Consumer Problem

The **producer-consumer problem** (**bounded-buffer problem**):

* Two processes share a common, fixed-sized buffer.
* Producer puts information into the buffer, and consumer takes it out.
* When the producer wants to put a new item in the buffer, but it is already full.
	* Go to sleep, awakened by customer when customer has removed one or more items
* When the consumer tries to take a item from the buffer, but buffer is already empty.
	* Go to sleep, awakened by producer when producer has puts one or more items into buffer.

```cpp
void producer(void) {
	int item;
	while (TRUE) { 													/* repeat forever */
		item = produce_item( ); 							/* generate next item */
		if (count == N) sleep( ); 						/* if buffer is full, go to sleep */
		insert_	item(item); 									/* put item in buffer */
		count = count + 1; 										/* increment count of items in buffer */
		if (count == 1) wakeup(consumer); 		/* was buffer empty? */
	}
}

void consumer(void) {
	int item;
	while (TRUE) { 													/* repeat forever */
		if (count == 0) sleep( ); 						/* if buffer is empty, got to sleep */
		item = remove_item( ); 								/* take item out of buffer */
		count = count − 1; 										/* decrement count of items in buffer */
		if (count == N − 1) wakeup(producer); /* was buffer full? */
		consume_item(item); 									/* print item */
	}
}
```

Problems:

1. Initially buffer is empty, `count = 0`.
2. The consumer just reads `count = 0`, since the consumer’s CPU time is over, scheduler assigns a CPU time to producer.
3. Producer produces item and checks count, `count = 0`. Insert item to buffer.Increase `count = count +1`. Now, since `count = 1`, it calls wakeup(consumer). Since the consumer is not sleeping yet, consumer misses the wakeup signal.
4. Consumer gets CPU time. Consumer already read `count = 0`, consumer goes to sleep.
5. Producer keep produce items and finally buffer become full. The producer go to sleep. Both sleep forever.

Quick fix by adding wakeup waiting bit:
* When a wakeup is sent to a consumer that is still awake, this bit is set.
* When the consumer tries to go to sleep:
	* If the wakeup waiting bit is on, it will be turned off, but the process will stay awake.

### 2.3.5 Semaphores

> **Semaphore** is integer variable type that counts the number of wakeups saved for future use. 

Two operations on semaphores (generalizations of **sleep** and **wakeup**):

1. Down
	* Checks to see if the value is greater than 0:
		* If so, it decrements the value (i.e., uses up one stored wakeup) and just continues.
		* If the value is 0, the process is put to `sleep` without completing the `down` for the moment.
2. Up 
	* Increments the value of the semaphore addressed.
	* If one or more processes were sleeping on that semaphore (unable to complete an earlier down operation) one of them is chosen by the system and is allowed to complete its down.

Checking the value, changing it, and possibly going to sleep, are all done as a single, indivisible **atomic action**.

```cpp
#define N 100 							/* number of slots in the buffer */
typedef int semaphore;					/* semaphores are a special kind of int */
semaphore mutex = 1; 					/* controls access to critical region */
semaphore empty = N; 					/* counts empty buffer slots */
semaphore full = 0; 					/* counts full buffer slots */

void producer(void) {
	int item;
	while (TRUE) { 								/* TRUE is the constant 1 */
		item = produce_item( ); 		/* generate something to put in buffer */
		down(&empty); 							/* decrement empty count */
		down(&mutex); 							/* enter critical region */
		insert_item(item); 					/* put new item in buffer */
		up(&mutex); 								/* leave critical region */
		up(&full); 									/* increment count of full slots */
	}	
}
void consumer(void) {
	int item;
	while (TRUE) {								/* infinite loop */
		down(&full); 								/* decrement full count */
		down(&mutex);								/* enter critical region */
		item = remove_item( ); 			/* take item from buffer */
		up(&mutex); 								/* leave critical region */
		up(&empty); 								/* increment count of empty slots */
		consume_item(item); 				/* do something with the item */
	}
}
```

> Semaphores that are initialized to 1 and used by two or more processes to ensure that only one of them can enter its critical region at the same time are called **binary semaphores** (semaphore that only has 2 values).

### 2.3.6 Mutexes

> A **mutex** is a shared variable that can be in one of two states: unlocked or locked.

When a thread (or process) needs access to a critical region, it calls **mutex lock**:
* If the mutex is currently unlocked (meaning that the critical region is available), the call succeeds and the calling thread is free to enter the critical region.
* If the mutex is already locked, the calling thread is blocked until the thread in the critical region is finished and calls mutex unlock.
* If multiple threads are blocked on the mutex, one of them is chosen at random and allowed to acquire the lock.

### 2.3.7 Monitors

> A **monitor** is a collection of procedures, variables, and data structures that are all grouped together in a special kind of module or package.

In monitor, only one process can be active in a monitor at any instant. Monitor uses **conditional variables** along with two operations on them, wait and signal.

When a monitor procedure discovers that it cannot continue (e.g., the producer finds the buffer full), it does a *wait* on some condition variable. This action causes the calling process to block.

On the other hand, the consumer can wake up its sleeping partner by doing a *signal* on the condition variable that its partner is waiting on.

This can cause two active processes at the same time so Hoare and Brinch Hansen proposed two solutions:

* Hoare: Letting the newly awakened process run, suspending the other one.
* Hansen: Requiring that a process doing a signal must exit the monitor immediately. If a signal is done on a condition variable on which several processes are waiting, only one of them, determined by the system scheduler, is revived.

## 2.4 SCHEDULING

When a computer is multiprogrammed, it frequently has multiple processes or threads competing for a CPU at the same time. If only one CPU is available, *scheduler* has to be made which process to run next by using **scheduling algorithm**. 

Three types of schedulers:
* Long-Term Scheduler – selects a process from the pool of job and load into memory for execution
* Short-term scheduler – selects a process from the ready queue and allocates the CPU.
* Memory Scheduler – schedule which process is in memory and in disk

### 2.4.1 Introduction to Scheduling

In addition to picking the right process to run, the scheduler also has to worry about making efficient use of the CPU because process switching is expensive:
* First a switch from user mode to kernel mode must occur
* The state of the current process must be saved, including storing its registers in the process table so they can be reloaded later.
* A new process must be selected by running the scheduling algorithm.
* Then the memory management unit must be reloaded with the memory map of the new process.
* Finally, the new process must be started.
* In addition, the process switch may invalidate the memory cache and related tables, forcing it to be dynamically reloaded from the main memory twice.

##### Process Behavior

Process that spend most of their time computing are called **compute-bound** or **CPU-bound**, and processes that spend most of their time waiting for I/O are called **I/O bound**. 

##### When to Schedule

Situations in which scheduling is needed:

1. When a new process is created, a decision needs to be made whether to run the parent process or the child process.
2. When a process exits, some other process must be chosen from the set of ready processes
3. When a process blocks on I/O, on a semaphore, or for some other reasons.
4. When an I/O interrupt occurs.

Two categories of scheduling algorithms with respect to how they deal with clock interrupts:
1. A nonpreemptive scheduling algorithm picks a process to run and then just lets it run until it blocks or voluntarily releases the CPU.
2. A preemptive scheduling algorithm picks a process and lets it run for a maximum of some fixed time.

##### Categories of Scheduling Algorithms

Different scheduling algorithms are needed in different environments. Three environments worth distinguishing are:

1. Batch:
	* Nonpreemptive algorithms, or preemptive algorithms with long time periods for each process, are often acceptable
	* Reduces process switches and thus improves performance
2. Interactive:
	* With interactive users, preemption is essential to keep one process from hogging the CPU and denying service to the others
3. Real time:
	* Preemption is, oddly enough, sometimes not needed because the processes know that they may not run for long periods of time and usually do their work and block quickly

##### Scheduling Algorithm Goals

Some goals for a scheduling algorithm:
* All systems:
	* Fair ness - giving each process a fair share of the CPU
	* Policy enforcement - seeing that stated policy is carried out
	* Balance - keeping all parts of the system busy
* Batch systems:
	* Throughput - maximize jobs per hour
	* Turnaround time - minimize time between submission and termination
	* CPU utilization - keep the CPU busy all the time
* Interactive systems:
	* Response time - respond to requests quickly
	* Propor tionality - meet users’ expectations
* Real-time systems:
	* Meeting deadlines - avoid losing data
	* Predictability - avoid quality degradation in multimedia systems

#### 2.4.2 Scheduling in Batch Systems

##### First-Come, First-Served

Basically, there is a single queue of ready processes. All incomming processes will be push into the queue and executed in order.

Strengths:
* Easy to understand and equally easy to program
* Fair in the same sense that allocating scarce resources

Weaknesses:
* If some biggest requests come first the average delay will be very high

##### Shortest Job First

The scheduler picks the **shortest job first** (the process that can be done in the shortest time)

Strengths:
* Optimal for scheduling.

Weaknesses:
* Very difficult to know the length of CPU request.

It is worth pointing out that shortest job first is optimal only when all the jobs are available simultaneously. Jobs arrive at different time, the scheduler only picks the shortest job available at a time which might not be optimal.

##### Shortest Remaining Time Next

A preemptive version of shortest job first is **shortest remaining time next**. With this algorithm, the scheduler always chooses the process whose remaining run time is the shortest. This scheme allows scheduler to pick the optimal solution.

#### 2.4.3 Scheduling in Interactive Systems

##### Round-Robin Scheduling

**Round-Robin**: each process is assigned a time interval, called its **quantum**, during which it is allowed to run. If the process is still running at the end of the quantum, the CPU is preempted and given to another process. If the process has been blocked or finished before the quantum has elapsed, the CPU switching is done when the process blocks, of course. 

One issue with Round-Robin is the length of quantum:
* If the length of quantum is too short, scheduler has to switch from process to process in a very short period of time and switching operation is both unuseful and resource-consumming.
* If the length of quantum is too long, the whole system will experience delays since other processes have to wait for the running process to finish.

Setting the quantum too short causes too many process switches and lowers the CPU efficiency, but setting it too long may cause poor response to short interactive requests. 

##### Priority Scheduling

The basic idea is straightforward: each process is assigned a priority, and the runnable process with the highest priority is allowed to run.

To prevent high-priority processes from running indefinitely, the scheduler may decrease the priority of the currently running process at each clock tick which causes its priority to drop below the next highest process's priority, so a process switch will occur.

Ways to define process priority:
* Internal way: based on the measurable quantity or quantities to compute the priority of a process.
* External way: based on the importance of the process

One problems with priority scheduling is the **starvation** of lower priority process. Processes with lowest priority might never be executed if higher priority processes keep comming.

But it can be solved using **aging**. A aging is a technique of gradually increasing the priorities of processes that wait in the system for a long time.

### 2.4.4 Scheduling in Real-Time Systems

A real-time system is one in which time plays an essential role. Typically, one or more physical devices external to the computer generate stimuli, and the computer must react appropriately to them within a fixed amount of time.

Real-time systems are generally categorized as **hard real time**, meaning there are absolute deadlines that must be met and **soft real time**, meaning that missing an occasional deadline is undesirable, but nevertheless tolerable. 

The events that a real-time system may have to respond to can be further categorized as **periodic** (meaning they occur at regular intervals) or **aperiodic** (meaning they occur unpredictably).

A real-time system is said to be **schedulable** if  

![alt text](schedulable.png "Schedulable formula")

Where:

**m**: number of periodic events 
Event **i** occurs with period **Pi** and requires **Ci** sec of CPU time

Real-time scheduling algorithms can be:
* **Static**: make their scheduling decisions before the system starts running
	* Works only when there is perfect information available in advance about the work
* **Dynamic**: make scheduling decisions at run time, after execution has started

## Chapter 3: MEMORY MANAGEMENT

**Memory hierarchy**:
* Small, fast, very expensive registers,
* Volatile, expensive cache memory,
* Megabyte medium-speed, medium-speed RAM
* Huge size of slow cheap, non-volatile disk storage (Hard Disks)

**Memory management** is a part of operating system which manage the memory hierarchy. Its job is to efficiently manage memory: keep track of which parts of memory are in use, allocate memory to processes when they need it, and deallocate it when they are done.

### 3.1 NO MEMORY ABSTRACTION

Three options to organize memory:

* Operating system at the bottom of memory in RAM.
* OS in ROM (Read-Only Memory) at the top of memory.
* The device drivers may be at the top of memory in a ROM and the rest of the system in RAM down below.

![alt text](memory-organization.png "Three options to organize memory")

Models (a) and (c) have the disadvantage that a bug in the user program can wipe out the operating system, possibly with disastrous results.

##### Running Multiple Programs Without a Memory Abstraction

However, even with no memory abstraction, it is possible to do multiprogramming if the operating system saves the entire contents of memory to a disk file, then bring in and run the next program. This concept is called **swapping**.

With the addition of some special hardware, it is possible to run multiple programs concurrently, even without swapping. The early models of the IBM 360 solved the problem as follows. Memory was divided into 2-KB blocks and each was assigned a 4-bit protection key held in special registers inside the CPU. A machine with a 1-MB memory needed only 512 of these 4-bit registers for a total of 256 bytes of key storage. The PSW (Program Status Word) also contained a 4-bit key. The 360 hardware trapped any attempt by a running process to access memory with a protection code different from the PSW key. Since only the operating system could change the protection keys, user processes were prevented from interfering with one another and with the operating system itself.

The core problem here is that the two programs reference absolute physical
memory but what they actually mean is relative memory of the program as the following.

![alt text](memory-multiprocess-problem.png "Problem with Running Multiple Programs Without a Memory Abstraction")

In this case, the program 2 (figure b), JMP to 28 which does CMP. But when loaded to memory, JMP to 28 means doing MOV instruction of program 1 (figure 1).

To solve this problem, the OS will automatically was loaded at address 16,384, 28 will be added to every program address during the load process so the instruction JMP 28 would be JMP 16,412 instead.

### 3.2 A MEMORY ABSTRACTION: ADDRESS SPACES

Exposing physical memory to processes has several major drawbacks:
* If user programs can address every byte of memory, they can easily trash the operating system, intentionally or by accident, bringing the system to a grinding halt
* It is difficult to have multiple programs running at once

#### 3.2.1 The Notion of an Address Space

> An **address space** is the set of addresses that a process can use to address memory. Each process has its own address space, independent of those belonging to other processes

##### Base and Limit Registers

The classical solution is to equip each CPU with two special hardware registers, usually called the **base** and **limit** registers.

When these registers are used, programs are loaded into consecutive memory locations wherever there is room and without relocation during loading.

When a process is running, the base register is loaded with the physical address where its program begins in memory and the limit register is loaded with the length of the program.

So in the last example, program 1 will have base and limit as 0 and 16,384, respectively, program 2 will have base and limit as 16,384 and 32,768, respectively. So that when the process run `JMP 28` it will be treated as `JMP 16412`.

In practice, the total amount of RAM needed by all the processes is often much more than can fit in memory. Keeping all processes in memory all the time requires a huge amount of memory and cannot be done if there is insufficient memory.

Two general approaches to dealing with memory overload:
* **Swapping**: bringing in each process in its entirety, running it for a while, then putting it back on the disk. Idle processes are mostly stored on disk, so they do not take up any memory when they are not running.
* **Virtual memory**: allows programs to run even when they are only partially in main memory

#### 3.2.2 Swapping

There are two ways to allocate memory:
* **Fixed partition**: For each process, a fixed sized partition will be created in memory.
	* Fixed location, size, and number of processes in memory.
	* Simple to manage.
* **Variable partition**: 
	* The number, location, and size of the partitions vary dynamically
	* Need to keep track of partition information dynamically for memory allocation and deallocation
	* Might create multiple memory holes.

Since processes usually change theirs size variable partition is needed but it introduces another problem as shown below.

![alt text](memory-swapping.png "Memmory swapping")

When swapping creates multiple holes in memory, it is possible to combine them all into one big one by moving all the processes downward as far as possible, which is known as **memory compaction**. But it is usually not done because it requires a lot of CPU time.

Solution for glowing space:
* Allocate extra memory for each process
* Use adjacent hole for managing the growing size
* If there is not enough room for a process to grow, it will be swapped out for killed.

#### 3.2.3 Managing Free Memory

When memory is assigned dynamically, the operating system must manage it and there are two ways to keep track of memory usage:
* Bitmaps
* Free lists

##### Memory Management with Bitmaps

With a bitmap, memory is divided into allocation units as small as a few words and as large as several kilobytes. Corresponding to each allocation unit is a bit in the bitmap, which is 0 if the unit is free and 1 if it is occupied. 

Issues with bitmap design:
* The smaller the allocation unit, the larger the bitmap.
* The larger the allocation unit, the smaller bitmap – But memory may be wasted in the last unit of the process if the process size is not an exact multiple of the allocation unit.

Advantage:
* Simple way to keep track of memory words in a fixed amount of memory

The main problem: 
* To allocate memory for the process with k unit size, the memory manager must search the bitmap to find a run of k consecutive 0 bits in the map and this is a slow operation.

##### Memory Management with Linked Lists

Another way of keeping track of memory is to maintain a linked list of allocated and free memory segments, where a segment either contains a process or is an empty hole between two processes. 

Each entry in the list specifies:
* A hole (H) or process (P)
* The address at which it starts
* The length
* A pointer to the next item.

Advantage: 
* When a process terminates or is swapped out, updating the list is straightforward

When the processes and holes are kept on a list sorted by address, several algorithms can be used to allocate memory for a created:
* **First fit**: 
	* The memory manager scans along the list of segments until it finds a hole that is big enough.
	* The hole is then broken up into two pieces, one for the process and one for the unused memory.
	* Fast because it searches as little as possible.
* **Next fit**:
	* Like first fit but it starts searching the list from the place where it left off last time instead of always at the beginning, as first fit does.
* **Best fit**:
	* Searches the entire list, from beginning to end, and takes the smallest hole that is adequate.
	* Slower than the previous two because it has to search the whole list.
	* Waste more memory because it tends to create tiny, useless hole, while holes in the previous two can be used for short process.
* **Worst fit**:
	* Contrast to best fit, it take the largest available hole, so that the new hole will be big enough to be useful.
	* Not a very good idea either.

Four algorithms can be speed up by maintaining two lists: 
1. A list for processes: as discussed above.
2. A list for holes: can be maintained with sorted by size. Best fit does not need search entire list.

With a hole list sorted by size, first fit and best fit are equally fast, and next fit is pointless.

Another allocation algorithm is **quick fit**, which maintains separate lists for some of the more common sizes requested. Quick fit is quick to finding a hole of required size but it also has the a disadvantage, maintain the separate lists for hole is expensive.

#### 3.3.1 Paging

On computers with virtual memory, when a program generate **virtual addresses**, the virtual addresses will be sent to a **MMU (Memory Management Unit)**  that maps the virtual addresses onto the physical memory address.

Example: A computer generates 16-bit addresses (0 - 64KB). However, this computer only has 32KB of physical memory, so although it can handle 64KB but only 32KB of them can be loaded into memory at a time. Therefore, 64KB has to be written on the disk and devided into small pieces and those pieces can be brought in as needed.

The virtual address space consists of fixed-size units called **pages** and corresponding units in the physical memory are called **page frames**. The two are generally the same size. Assume that every page and page frame is 4KB so that we will have 64/4 = 16 pages and 32/4 = 8 page frames

**Page fault** is a sittuation in which a program references to an unmapped address. In this case the OS will pick a page with little useage and replace it with the unmapped address.

With a 16 pages and 4KB per page memory, we need 16-bit virtual address consists of 4-bit page number (2^4 = 16 pages) and 12-bit offset (2^12 = 4KB)

The page number is used as an index into the **page table**.  If the **Present/absent bit** is 0, the OS will remap page frame. If it is 1, the OS will find the corressponding address and put onto the memory bus.

#### 3.3.2 Page Tables

The mapping of virtual addresses onto physical address:
1. The virtual address is split into a virtual page number and
2. An offset

The page number is used to find corresponding page frame and offset is to find physical address.

##### Structure of a Page Table Entry

A typical page table entry:

1. Page frame number
2. Present/absent bit
3. Protection bits: what kinds of access are permitted
4. Modified bit. If the page has been modified, it will be written to the disk before is removed from page frame. If not, it can just be abandoned.
5. Referenced bit  is set whenever a page is referenced, either for reading or for writing. This bit is used to determine which page is evicted when a page fault occurs.
6. Cache disable bit

#### 3.3.3 Speeding Up Paging

Two issues which paging:

1. The mapping from virtual address to physical address must be fast.
2. If the virtual address space is large, the page table will be large.

##### Translation Lookaside Buffers

Using paging reduces the performance of computer since it requires additional reference to access page table. To tackle this perform, computer designers use a device called **TLB (Translation Lookaside Buffer)** or sometimes an **associative memory**.

How TLB works:

* When a virtual address is presented to the MMU for translation, the hardware check if its virtual page number is in TLB.
* If a match is found and the protection bit is not violated, the page frame is taken from TLB without going to the page table.
* If a match is found but the protection bit is violated, a protection fault is generated.
* If no match is found, MMU detects the miss and does an ordinary page table lookup. Then it wil replace one entry with the entry it looked up earlier.

#### 3.3.4 Page Tables for Large Memories

##### Multilevel Page Tables

Example: We have 32-bit virtual address which is devided into 3 partitions: a 10-bit PT1 field, a 10-bit PT2 field, and a 12-bit Offset. Since there are 12 bits offset, each page is 4KB and there are 2^10*2^10 = 2^20 of them

The point for having mutiple page table is to avoid keeping all the page table in memory all the time. 

![alt text](multiple-page-table.png "Multiple page table")

In the figure above, the top-level contains 1024 entries. Assume a process needs 12 megabytes: the bottom
4 megabytes of memory for program text, the next 4 megabytes for data, and the top 4 megabytes for the stack. Therefore, we only need 3 entries from the top level page table. These entries reference to 3 second-level page tables which will be used as indexes for page frame. 

Example: Address space: 0x00403004 (0000000001 0000000011 000000000100) contains PT1: 1 (0000000001), PT2: 3 (0000000011), and offset 4 (000000000100).

It is possible for PT1 and PT2 to have different sizes and a page-table system can also be expanded to three or four level

### 3.4 PAGE REPLACEMENT ALGORITHMS

When a page fault occurs, the operating system has to choose a page to evict to make room for a new one. If the page is modified in the memory, it must be written to the disk before removed. If it is not modified, a new page will overwrite it.

#### 3.4.1 The Optimal Page Replacement Algorithm

The **optimal page replacement algorithm**: the page that will not be used for the longest period of time should be removed

This approach guarantee the optimal solution but impossible most the time since there is no way to OS can know about future usage.

#### 3.4.2 The Not Recently Used Page Replacement Algorithm

To collect useful page usage statistic, we can have two status bits:
* R is set whenever the page is referenced
* M is set when the page is written to

When a page fault occurs, the operating system inspects all the pages and divides them into four categories based on the current values of their R and M bits:
* Class 0: not referenced, not modified.
* Class 1: not referenced, modified.
* Class 2: referenced, not modified.
* Class 3: referenced, modified.

The NRU algorithm removes a page at random from the lowest numbered nonempty class.

#### 3.4.3 The First-In, First-Out (FIFO) Page Replacement Algorithm

The operating system maintains a list of all pages currently in memory, with the most recent arrival at the tail and the least recent arrival at the head. On a page fault, the page at the head is removed and the new page added to the tail of the list. 

This approach is rarely used since the oldest page may still be useful.

#### 3.4.4 The Second-Chance Page Replacement Algorithm

A simple modification to FIFO that avoids the problem of throwing out a heavily used page is to inspect the R bit of the oldest page. If it is 0, the page is both old and unused, so it is replaced immediately. If the R bit is 1, the bit is cleared, the page is put onto the end of the list of pages, and its load time is updated as though it had just arrived in memory. Then the search continues.

If every page is referenced the oldest will be evicted since the R bit is cleared when a page is moved.

#### 3.4.5 The Clock Page Replacement Algorithm

A modification of the second chance. Clock page replacement algorithm keeps all pages in a circular list so that a page does not need to be moved around like in the second chance.

#### 3.4.6 The Least Recently Used (LRU) Page Replacement Algorithm

**LRU (Least Recently Used)** paging: when a page fault occurs, throw out the page that has been unused for the longest time.

This approach yield almost optimal solution but create a lot overhead since maintaining a linked list of page, updating it, removing element from it every memory reference is time-consuming.

### 3.5 Modeling Page Replacement Algorithm

#### 3.5.1 Belady’s Anomaly

Intuitively, it might seem that the more page frames the memory has, the fewer page faults a program will get. This is not always the case – **Belady’s Anomaly**

For instance:

Job sequence:

0 1 2 3 0 1 4 0 1 2 3 4

3 page frames with 9 page faults (bold ones):

| 0 | 1 | 2 | 3 | 0 | 1 | 4 | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **0** | 0 | 0 | **3** | 3 | 3 | **4** |   |   | 4 | 4 |   |
|   | **1** | 1 | 1 | **0** | 0 | 0 |   |   | **2** | 2 |   |
|   |   | **2** | 2 | 2 | **1** | 1 |   |   | 1 | **3** |   |

4 page frames with 10 page faults (bold ones):

| 0 | 1 | 2 | 3 | 0 | 1 | 4 | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **0** | 0 | 0 | 0 |   |   | **4** | 4 | 4 | 4 | **3** | 3 |
|   | **1** | 1 | 1 |   |   | 1 | **0** | 0 | 0 | 0 | **4** |
|   |   | **2** | 2 |   |   | 2 | 2 | **1** | 1 | 1 | 1 |
|   |   |   | **3** |   |   | 3 | 3 | 3 | **2** | 2 | 2 |

#### 3.5.2 Stack Algorithm

A paging system can be characterized by three items:
1. The reference string of the executing process
2. The page replacement algorithm
3. The number of page frames available in memory

How **Stack Algorithm** works:
* Maintains an internal array M that keep track of the state of memory.
* M has as many as virtual memory pages n.
* Top m entries contain all the pages currently in the memory (page frames).
* Bottom n - m entries contains all the pages that have been referenced once but have been page out and are not currently in memory

**Stack Algorithm** in action:

1. When a page is referenced, it is always moved to the top entry in M.
2. If the page referenced was already in M, all pages above it move down one position.
3. A transition from within the box to outside of it corresponds to a page being evicted from the memory
4. The pages that were below the referenced page are not moved.

Basically, stack algorithm increases page frame (actual page frame in memory + additional virtual page frame on disk). 

Stack algorithm do not suffer from Belady’s Anormaly because instead of leaving the list unchanged in the referenced page is already in the list, like in normal page replacement algorithm, it move every page above down one and move it on top (step 2, 3).

### 3.5 DESIGN ISSUES FOR PAGING SYSTEMS

#### 3.5.1 Local versus Global Allocation Policies

**Local page replacement**: only consider replacement for pages that are used by the process that request referenced page:
* Fixed amount of page for a process

**Global page replacement**: consider replacement for all pages in memory.
* Number of page for each process varies from time to time
* In general, better solution

#### 3.5.2 Load Control

Even though we use best replacement algorithm with global allocation policies, the system might still thrash, if the combined working sets of all processes exceed the capacity of memory.

Possible concern is reducing the degree of multiprogramming – swap out some of processes into the disk.

#### 3.5.3 Page Size

The **page size** is a parameter that can be chosen by the operating system

**Internal fragmentation** is the wasted space within each allocated block because of rounding up from the actual requested allocation to the allocation granularity.  

**External fragmentation** is the various free spaced holes that are generated in either your memory or disk space. External fragmented blocks are available for allocation, but may be too small to be of any use.

If the page size is too big:
* Waste since data may not use up the whole page size

If the page size is too small:
* Requires more page, more memory to keep track those pages
* Too many pages means more time for seeking and rotating
* Longer loading time

Overhead and optimal page size:

```
	Overhead = s * e / p + p/2

	Overhead' = - s * e / p^2 + 1/2 = 0

	Optimal page size = sqrt(2 * s * e)

```

Average process size: **s**

Page size: **p**

Page entry size: **e**

Number of pages needed per process: **s**/**p**

Page table space: **s****e**/**p**

Internal fragmentation: **p**/2

### 3.7 SEGMENTATION

Segment is a logical entity, which the programmer knows and uses as a logical entity.

But divided memory into segments, computer can handle more jobs seen segment can grow and shrink accordingly to jobs.

#### 3.7.1 Implementation of Pure Segmentation

OS maintain a segment table to keep track of all segments

|   | limit | Base |
|---|-------|------|
| 0 | 1000  | 1400 |
| 1 | 400   | 6300 |
| 2 | 400   | 4300 |
| 3 | 1100  | 3200 |
| 4 | 1000  | 4700 |

Example:

* A reference to segment 3 to byte 852: 3200 + 852 = 4052
* A reference to segment 0 to byte 1222: Error since byte 1222 is out of range of segment 0

When using segment, memory will be divided up into a number of chunks, some containing segments and some containing holes. Those holes in memory is called **external fragmentation**, wasted memory in holes.

#### 3.7.2 Segmentation with Paging

If the segments are large, it may be inconvenient, or even impossible, to keep them in main memory in their entirety. Fortunately, we can use a combination of segmentation and paging to tackle that problem.

To keep track of all segment OS uses segment table which contains **segment descriptor**. Segment descriptor contains address to the page table, segment length (in pages), and several other infomation.

So a virtual address for this requires 3 things:

1. Segment number
2. Page number  
3. Offset within page

How it works:
* The segment number was used to find the segment descriptor.
* A check was made to see if the segment’s page table was in memory. If it was, it was located. If it was not, a segment fault occurred. If there was a protection violation, a fault (trap) occurred.
* The page table entry for the requested virtual page was examined. If the page itself was not in memory, a page fault was triggered. If it was in memory, the main-memory address of the start of the page was extracted from the page table entry.
* The offset was added to the page origin to give the main memory address where the word was located.
* The read or store finally took place.


## Chapter 4: FILE SYSTEMS

Three essential requirements for long-term information storage:

1. It must be possible to store a very large amount of information.
2. The information must survive the termination of the process using it
3. Multiple processes must be able to access the information at once

### 4.1 FILES

#### 4.1.1 File System

#### 4.1.2 File Structure

Three kinds of file structure:
* Byte sequence
* Record sequence
* Tree
	* File consists of a tree of records

#### 4.1.3 File Types

File types:
* Regular files:
	Contain user information
* Directories:
	System files for maintaining the structure of the file system
* Character special files:
	Related to input/output and used to model serial I/O devices, such as terminals, printers, and networks
* Block special files:
	Are used to model disks

##### Regular files

Two types of regular files:
* ASCII files:
* Binary files:
	* Executable files:
		* Has 5 sections: header, text, data, relocation bits, and symbol table
		* Header: contains **magic number**, identifying the file as an executable file
	* Archive files
		* A collection of library procedures (modules) compiled but not linked

#### 4.1.4 File Access

**Sequential access** :
* A process can read all the bytes in a file in order, starting from the beginning. 
* Cannot read out of order

**Random Access** :
* A process can read all the bytes in a file in any order. 

### 4.2 DIRECTORIES

Three ways to structure directory:
* Single-level directory System 
	* One directory containing all files
* Two-level directory System
	* Giving each user a private directory
* Hierarchical directory System
	* User can create a directory to group their files in logical way

### 4.3 FILE-SYSTEM IMPLEMENTATION

#### 4.3.1 File-System Layout

Most disk can be divided into partitions with different file system. Sector 0 of the disk is called the **MBR** (**Master Boot Record**) and is used to boot the computer.

The end of the MBR contains the **partition table**. This table gives the starting and ending addresses of each partition. Only one partition can be active at a time.

When the computer is booted:

1. The BIOS reads in and executes the MBR
2. MBR locates the active partition, read in its first block, which is called the **boot block**, and execute it.
3. The program in the boot block loads the operating system contained in that partition

![alt text](fs-layout.png "fs-layout")

#### 4.3.2 Implementing Files

##### Contiguous Allocation

**Contiguous allocation**: store each file as a contiguous run of disk blocks

Advantages:
* Simple to implement: only need the disk address of the first block and the number of blocks in the file to keep track of where a file’s blocks
* The read performance is excellent: only one seek for a read

Drawback:
* Fragmentation: create a lots of holes which can be too small for a program ti fit in
* Only works well for fixed size files

##### Linked-List Allocation

**Linked-List allocation**: the first word of each block is used as a pointer to the next one. The rest of the block is for data.

Advantages:
* No fragmentation
* Only need to store address of the first block

Drawback:
* Random access is extremely slow because OS has to start at beginning of the list every single time

##### Linked-List Allocation Using a Table in Memory

All link information for each of file is written into the File Allocation Table (FAT) which is located in main memory

Advantages:
* No fragmentation
* Random access is much easier comparing to Linked-List Allocation

Drawback:
* Need to maintain FAT
* Entire table must be in memory all the time to make it work

##### I-nodes

**I-node** (**index-node**): lists the attributes and disk addresses of the file’s blocks. Therefore, with a given the i-node, it is possible to find all the blocks of the file. The last block can be used to link to another i-node, hence files can be as big as needed.

Advantages:
* No fragmentation
* Only need one i-node in memory if a file is opened
* Far smaller size than FAT
* Disproportional with disk size

#### 4.3.3 Implementing Directories

**Design 1**: a directory consists of a list of fixed-size entries, one per file, containing a (fixed-length) file name, a structure of the file attributes, and one or more disk addresses (up to some maximum) telling where the disk blocks are.

**Design 2**: using i-node, a directory consists of a name and a pointer to corresponding i-node.

How to handle long file name? There are three approaches:

1. Set a limit on file name: Wasting memory since not all file will have that length
2. All directory entries are start with length of the entry followed by data with fixed format: Leads to fragmentation because deleting a file means creating a hole (same as contiguous disk files)
3. Make the directory entries all fixed length and keep the files names together in a heap at the end of the directory

#### 4.3.4 Shared Files

When several users are working together on a project, they often need to share files. As a result, it is often convenient for a shared file to appear simultaneously in different directories belonging to different users.

A problem with shared file:
* Assume a shared file shared with user B and user C.
* Then file information must be saved in both directory B and directory C.
* If user B modifies shared file, only B can see changes, thus defeating the purpose of sharing.

Solutions:

1. Disk blocks information for a file are not saved in the directory but in i-node.
2. A system create a symbolic link to a shared file. The new file contains the path name of the shared file

Both have some drawbacks

##### With i-node:

1. Problem 1:
	* At the moment that B links to the shared file, the i-node records the file’s owner as C
	* If C then tries to remove the file, it removes the file and clears the i-node. Hence B point to an invalid i-node
	* If the i-node is later reassigned to another file, B’s link will point to the wrong file
2. Problem 2: A solution for the previous problem is to have the system check the number of user for a file in i-node. If the number of user is = 0, just delete it. If it > 0, remove link for C but keep link for B. Still have a problem
	* Since the file is created by C, C is the owner.
	* C will continue to be billed for the file even if he doesn't need it and C can't stop that except have B delete it for him

##### Symbolic links: 

The previous problem does not occur with symbolic links since only real owner has pointer to the i-node. But it requires extra overhead for getting a path, parsing it, and maintaining an extra block for i-node for each link.

#### 4.3.5 Log-Structured File Systems

**Log-Structured File Systems** is a kind of file system that is designed to improve access time to files.

Log-structured file systems are based on the assumption that files are cached in main memory.

As a result, disk traffic will become dominated by writes. 

How LSF works:
* All writes are initially buffered in memory
* Periodically, all the buffered writes are written to the disk in a single segment, at the end of the log.
* A single segment contains i-nodes, directory blocks, and data blocks, all mixed together so LSF uses those information to write to disk

LFS uses a data structure called an i-node map to maintain the current location of each i-node.

Once a existing file is open for update, a segment of the file is load to the memory 

#### 4.3.6 Journaling File Systems

In JFS, the OS keeps track of what the FS is going to do before it does which means that the OS have a set of FS's planned works. Hence if the system crashes before the FS finish it job, the OS will look at the journal and finish planned job in the next reboot.


### 4.4 FILE-SYSTEM MANAGEMENT AND OPTIMIZATION

#### 4.4.1 Disk-Space Management

##### Block size

As discussed above, storing files in segments gives the OS some flexibilities but just as in page table, determining block size is also important

If block size is too small, wasting time for seeking and rotation delay. If block size is too big, internal fragmentation or wasting space.

Formula for calculating time to read a file of `k` bytes, `B` block size, rotation time `R` msec, and avg seek time `S`:

```
	S + R/2 + round( k/B ) * R
```

##### Keeping Track of Free Blocks

Two ways to manage free blocks:
* As linked list
* As bit map

###### Linked list

How it works:
* When the free list method is used, only one block of pointers need be kept in main memory.
* When a file is created, the needed block blocks are taken from the block of pointers.
* When it runs out a new block of pointer is read in from the disk.
* When a file is deleted, its blocks are freed and added to the block of pointer in the main memory.

With a 1-KB block, a 32-bit disk block number, and 16 GB disk:
* A block can hold 8 * 2^10 / 32 = 256 -1 = 255 free blocks
* There are 2^34/2^10 = 2^24 KB disk blocks
* 2^24/255 = 65794 blocks  to hold free block numbers
* Need 65794 KB 

###### Bitmap

A disk with n block requires a bitmap with n bits. A free blocks are represented by 1s in the map, allocated blocks by 0 (or vice versa).

With a 1-KB block, and 16 GB disk:
* There are 2^34/2^10 = 2^24 KB disk blocks
* Need 2^24 (bits) = 2^24 / 2^3 (bytes) = 2^11 * 2^10 (bytes) = 2^11 KB
* Need 2^11 KB

#### 4.4.2 File-System Backups

File Backup System design issues:
1. Should the entire file system be backed up or only part of it?
2. it is wasteful to back up files that have not changed since the last backup
3. Since big amount of data are typically dumped, it may be desirable to compress the data before writing them to tape.
4. It is difficult to perform a backup on an active file system – making rapid snapshots of the file system.
5. Backup introduces many nontechnical problems into an organization – security problem

There are two strategies for dumping a disk to tape or disk: 
* Physical and 
* Logical Dump

##### Physical Dump

A **physical dump** starts at block 0 of the disk, writes all the disk blocks onto the output disk in order, and stops when it has copied the last one.

Advantages: easy to implement

Disadvantages: inability to skip selected directories, make incremental dumps, and restore individual files upon request

##### Logical Dump

A **logical dump** starts at one or more specified directories and recursively dumps all files and directories found there that have changed since some given base date.

The dump algorithm maintains a bitmap indexed by i-node number with several bits per i-node The algorithm operates in four phases

* Phase 1
	Begins at the starting directory and examines all the entries in it. For each modified file, its i-node is marked in the bitmap. Each directory is also marked and recursively inspected.
* Phase 2
	Unmarking any directories that have no modified files or directories in them or under them.
* Phase 3
	All marked directory is dumped
* Phase 4
	All marked files is dumped

## Chapter 5

### 5.1 PRINCIPLES OF I/O HARDWARE

#### 5.1.1 I/O Devices

I/O devices can be divided into two categories:

1. Block devices:

	* Stores information in fixed-size blocks, each one with its own address.
	* All transfers are in units of one or more entire (consecutive) blocks.
	* It is possible to read or write each block independently of all the other ones.
	* Need seek operation before read or write operation

2. Character devices:

	* Delivers or accepts a stream of characters, without regard to any block structure.
	* It is not addressable and does not have any seek operation

Some devices do not fit in two group listed above.

#### 5.1.2 Device Controllers

I/O units often consist of a mechanical component and an electronic component (**device controller** or **adapter**).

A device's controller plays an important role in the operation of that device; it functions as a bridge between the device and the operating system.

Each device controller has a local buffer and a command register. It communicates with the CPU by interrupts.

How a device controller works:
* The Device Controller receives the data from a connected device and stores it in local buffer inside the controller. 
* Then it communicates the data with a Device Driver.
* For each device controller there is an equivalent device driver which is the standard interface through which the device controller communicates with the Operating Systems through interrupts.

#### 5.1.3 Memory-Mapped I/O

Each controller has a few registers used for communicating with CPU.

By writing into these registers, OS can command the device to:
* To deliver data (write)
* To accept data (read)
* To switch itself on or off

By reading from these registers, OS can learn:
* What is the device state
* Whether it is ready to accept new command or not

In addition to the control registers, many devices have a data buffer that the operating system can read and write.

The issue thus arises of how the CPU communicates with the control registers and also with the device data buffers. There are two approaches:

1. Non Memory-Mapped I/O
2. Memory-Mapped I/O

##### Non Memory-Mapped I/O

Each control register is assigned an I/O port number (8 bit or 16 bit number)

The set of all the I/O ports form the I/O port space, only the OS can access it.

By using special I/O instruction, the CPU can read in control register and can write the content of a register

In this scheme, the address spaces for memory and I/O are different

##### Memory-Mapped I/O

Map all the control registers into the memory space. Each control register is assigned a unique memory address to which no memory is assigned

The assigned addresses are usually at the top of the address space.

When the CPU wants to read a word, either from memory or from an I/O port:
* The CPU puts the address on the bus’s address line
* Every memory module and every I/O device compares the address lines to the range of addresses that it services
* If the address falls in its range, it responds to the request. Since no address is ever assigned to both memory and an I/O device, there is no ambiguity and no conflict.


###### Hybrid

Using memory-mapped I/O data buffers and separate I/O ports for the control registers

When the CPU wants to read a word, either from memory or from an I/O port:
* The CPU puts the address on the bus’s address line
* Then, asserts a read signal on the bus’s control line to tell whether I/O space or memory space is needed.
* Either I/O device or memory responds to the request based on the second signal line.

![alt text](io-schemes.png "io-schemes")

Advantages:
* With memory-mapped I/O high level language can be used to write an I/O device driver, otherwise,  some assembly code is needed.
* No special protection mechanism is needed to keep user processes from performing I/O.
* Every instruction that can reference memory can also reference control registers. Hence, it is faster in execution.

Disadvantages:
* Caching in nowaday computers can create problems.
* The trend in modern PC is to have a dedicated high-speed memory bus between memory but I/O devices have no way of seeing memory addresses as they go by on the memory bus. Hence, an extra mechanism is needed.


#### 5.1.4 Direct Memory Access

No matter whether a CPU does or does not have memory-mapped I/O, it needs to address the device controllers to exchange data with them. The CPU can request data from an I/O controller one byte at a time, but doing so wastes the CPU’s time, so a different scheme, called **DMA (Direct Memory Access)** is often used.

Disk read from Disk to Memory without DMA controller:

1. The disk controller reads data bit by bit from the disk until an entire block is in the controller’s buffer.
2. It checks checksum to verify any error.
3. If there is no error, controller causes an interrupt to get a service from operating system.
4. The service transfer data from controller’s buffer to main memory through bus line


When DMA is used:

1. The CPU programs the DMA controller by setting its registers so it knows what to transfer and where to transfer to. It also issues a command to the disk controller telling it to read data from the disk into its internal buffer and verify the checksum.
2. The DMA controller initiates the transfer by issuing a read request over the bus to the disk controller.
3. Transfer data.
4. When it completes transfering, the disk controller sends an acknowledgement signal to the DMA controller. It also increments memory address to use and decrements the byte count. If byte count still > 0, repeat from step 2 to step 4.

![alt text](DMA.png "AMD")

I/O device interupts:
* When a I/O device finishes its job, it sends a signal for interrupt service through bus line.
* This signal is detected by the interrupt controller chip on the parentboard.
* If no other interrupt are pending, the interrupt controller processes the interrupt immediately.
* If interrupt controller is busy handling another interrupt, the device is just ignored for the moment and continues to assert an interrupt signal on the bus until it is serviced by the CPU.
* Interrupt controller sends a signal to the CPU causes it to stop what it is doing and start to handle the interrupt. The number on the address lines is used to index the **interrupt vector**.
* Interrupt vector has pointers where interrupt service procedures are located.
* Program counter saves the location where the interrupt service procedure is located.
* Shortly after it starts running, the interrupt service procedure acknowledges the interrupt by writing a certain value to one of the interrupt controller’s I/O ports which tells the controller that it is free to issue another interrupt.

### 5.2 PRINCIPLES OF I/O SOFTWARE

#### 5.2.1 Goals of the I/O Software

Goals:
* Device independence: write programs that can access any I/O device without having to specify the device in advance
* Uniform naming: the name of a file or a device should simply be a string or an integer and not depend on
the device in any way
* Error handling: 
	* Error must be handled as close to the hardware possible.
	* That means I/O error must by detected by controller or device itself.
* Synchronous (blocking) vs. asynchronous (interrupt-driven) transfers
* Buffering: For some devices, data cannot be stored directly in its final destination. It must be
saved in a buffer and check error or decoded proper form.
* Sharable vs. Dedicated Device I/O Software: 

There are three fundamentally different ways that I/O can be performed: 
1. Programmed I/O
2. Interrupt-Driven I/O
3. I/O using DMA

#### 5.2.2 Programmed I/O

How it works, considering you want to print the eight-character string "ABCDEFGH" on the printer via a serial interface:

1. The software assembles the string in a buffer in user space
2. The user process then acquires the printer for writing by making a system call to open it.
	* If printer is currently in use by another process, this call will fail and return an error code or will block until the printer is available
3. Once it has the printer, the user process makes a system call telling the operating system to print the string on the printer.
4. The operating system then (usually) copies the buffer with the string to an array, say, `p`, in kernel space, where it is more easily accessed
5. It then checks to see if the printer is currently available. If not, it waits until it is.
6. As soon as the printer is available, the operating system copies the first character to the printer’s data register (the character might not appear yet because some printer buffers a whole line or page before printing)
7. Operating system waits for the printer to become ready again to print another one and repeat steps 6,7 until the end of the string.

Programmable view of the procedure:

```c++
copy_from_user(buffer, p, count); 					/* p is the ker nel buffer */
for (i = 0; i < count; i++) { 							/* loop on every character */
	while (*printer_status_reg != READY) ; 		/* loop until ready */
	*printer_data_register = p[i]; 						/* output one character */
}
return_to_user( );
```

Programmed I/O is simple but has the disadvantage of tying up the CPU full time until all the I/O is done.

#### 5.2.3 Interrupt-Driven I/O

Print on a printer that does not buffer characters but prints each one as it arrives.

**Interrupt-Driven I/O** lets the CPU to do something else while waiting for the I/O device to becomes ready.

Programmable view of the procedure:

Code executed at the time the print system call is made:
```c++
copy_from_user(buffer, p, count);
enable_interrupts( );
while (*printer_status+reg != READY);
*printer_data_register = p[0];
scheduler( );
```

Interrupt service procedure for the printer.
```c++
if (count == 0) {
	unblock user( );
} else {
	*printer_data_register = p[i];
	count = count − 1;
	i = i + 1;
}
acknowledge_interrupt( );
return_from_interrupt( );
```

#### 5.2.4 I/O Using DMA

An obvious disadvantage of interrupt-driven I/O is that an interrupt occurs on every character. Interrupts take time, so this scheme wastes a certain amount of CPU time. A solution is to use DMA.

In essence, **DMA** is **programmed I/O**, only with the DMA controller doing all the work, instead of the main CPU so it needs requires special hardware (the DMA controller) but frees up the CPU during the I/O to do other work.

Code executed at the time the print system call is made:
```c++
copy_from_userbuffer, p, count);
set_up_DMA_controller( );
scheduler( );
```

Interrupt service procedure for the printer.
```c++
acknowledge_interrupt( );
unblock_user( );
return_from_interrupt( );
```

The big win with DMA is reducing the number of interrupts from one per character to one per buffer printed.

On the other hand, the DMA controller is usually much slower than the main CPU.

### 5.3 I/O SOFTWARE LAYERS

I/O software is typically organized in four layers:

![alt text](io-layers.png "io-layers")

#### 5.3.1 Interrupt Handlers

Steps for Interrupt handler:

1. Save any registers (including the PSW) that have not already been saved by the interrupt hardware.
2. Set up a context for the interrupt-service procedure. Doing this may involve setting up the TLB, MMU and a page table.
3. Set up a stack for the interrupt service-procedure.
4. Acknowledge the interrupt controller. If there is no centralized interrupt controller, reenable interrupts.
5. Copy the registers from where they were saved (possibly some stack) to the process table.
6. Run the interrupt-service procedure. It will extract information from the interrupting device controller’s registers.
7. Choose which process to run next. If the interrupt has caused some high-priority process that was blocked to become ready, it may be chosen to run now.
8. Set up the MMU context for the process to run next. Some TLB setup may also be needed.
9. Load the new process’ registers, including its PSW.
10. Start running the new process.

#### 5.3.2 Device Drivers

Each I/O device attached to a computer needs some device-specific code, called **device driver**, for controlling it. 

Device driver: 
* Is generally written by the device’s manufacturer and delivered along with the device
* Is the interface between OS and a I/O controller.
* In order to access the device’s hardware, each of device deriver has to be part of the operating system kernel. But, it is still possible to construct a driver which run in user’s space, with system calls for reading and writing the device registers.

**Standard interfaces**: a number of procedures that the rest of the OS can call to get the deriver to do work for it.

An OS with a single binary program (UNIX) that contain all of device driver need to be recompile if a new driver is added.

Operating systems usually classify drivers into one of a small number of categories:
* Block devices
* Character devices

### 5.4 DISKS

Disks come in a variety of types:
* Magnetic hard disks: reads and writes are equally fast, which makes them suitable as secondary memory
* Optical disks
* Solid-state disks

#### 5.4.1 Disk Hardware

##### Magnetic Disks

Magnetic disks are organized into cylinders, each one containing as many tracks as there are heads stacked vertically. The tracks are divided into sectors, with the number of sectors around the circumference typically being 8 to 32 on floppy disks, and up to several hundred on hard disks. The number of heads varies
from 1 to about 16.

We can convert a logical block number into an disk address that consists of a cylinder number, a track number and a sector number

#### 5.4.2 Disk Formatting

The format consists of a series of concentric tracks, each tack containing sectors, with short gaps between sectors

The format of a sector: 

1. Preamble – start with a bit pattern which contains the cylinder number, sector number, and other information
2. Data – size of data portion is determined by formatting program (usually 512 bytes).
3. ECC (error correcting code) – information used for error correction.

#### 5.4.3 Disk Arm Scheduling Algorithms

The time required is determined by three factors:

1. Seek time (the time to move the arm to the proper cylinder).
2. Rotational delay (how long for the proper sector to appear under the reading head).
3. Actual data transfer time

For most disks, the seek time dominates the other two times, so reducing the mean seek time can improve system performance substantially.

##### FCFS –First Come First Serve

Requests come in order:  98, 183, 37, 122, 14, 124, 65, and 67. The disk head is initially at cylinder 53

![alt text](diskscheduling-fcfs.png "fcfs")

FCFS : 98, 183, 37, 122, 14, 124, 65, 67

Total Head movement = |98 - 53| + |183 - 98| + |37 - 183| + |122 - 37| + |14 - 122| + |124 - 14| + |65 -124| + |67 - 65| = 640

#### Shortest-Seek-Time-First (SSTF) 

Serve closest to the current location.

![alt text](diskscheduling-sstf.png "sstf")

SSTF : 65, 67, 37, 14, 98, 122, 124, 183

Total Head movement = |65-53| + |67-65| + |37-67| + |14-37| + |98-14| + |122-98| + |124-122| + |183-124| = 236

#### Elevator Scheduling (SCAN) 

The disk arm starts at one end of the disk, and moves toward the other end.

![alt text](diskscheduling-scan.png "diskscheduling-scan")

SCAN: 37, 14, 0, 65, 67, 98, 122, 124, 183
Total Head movement = |37-53| + |14-37| + |0-14| + |65-0| + |67-65| + |98-67| + |122-98| + |124-122| + |183-124|

#### C-SCAN

Move the head from one end of disk to the other. When the head reaches the other end, it immediately return to the beginning of the disk without servicing any request on the return trip.

![alt text](diskscheduling-cscan.png "diskscheduling-cscan")

CSCAN: 65, 67, 98, 122, 124, 183, 199, 0, 14, 37

#### LOOK

Both Elevator and C-SCAN move the disk arm across the full width of the disk. More commonly, the arm goes only as far as the final request in each direction, then it reverses direction immediately without going all the way to the end of the disk.

The versions of SCAN and C-SCAN algorithm are called LOOK and C-LOOK

### 5.4.4 Error Handling

**Bad sectors**: sectors that do not correctly read back the value just written to them.

## Chapter 6

### 6.1 RESOURCES

#### 6.1.1 Preemptable and Nonpreemptable Resources

A **preemptable resource** is one that can be taken away from the process owning it with no ill effects.

A **nonpreemptable resource**, in contrast, is one that cannot be taken away from its current owner without potentially causing failure. 

In general, deadlocks involve nonpreemptable resources because deadlocks caused preemptable resource can be solved by reallocating resources from a process to another process.

The abstract sequence of events required to use a resource:
1. Request the resource.
2. Use the resource.
3. Release the resource.

### 6.2 INTRODUCTION TO DEADLOCKS

**Deadlock**: A set of processes is deadlocked if each process in the set is waiting for an
event that only another process in the set can cause.

#### 6.2.1 Conditions for Resource Deadlocks

Four conditions for deadlocks:

1. Mutual exclusion condition. 

	Each resource is either currently assigned to exactly one process or is available.

2. Hold-and-wait condition. 

	Processes currently holding resources that were granted earlier can request new resources.

3. No-preemption condition. 

	Resources previously granted cannot be forcibly taken away from a process. They must be explicitly released by the process holding them.

4. Circular wait condition. 

	There must be a circular list of two or more processes, each of which is waiting for a resource held by the next member of the chain. 

All four of these conditions must be present for a resource deadlock to occur. If one of them is absent, no resource deadlock is possible.

In general, four strategies are used for dealing with deadlocks:

1. Just ignore the problem. Maybe if you ignore it, it will ignore you.
2. Detection and recovery. Let them occur, detect them, and take action.
3. Dynamic avoidance by careful resource allocation.
4. Prevention, by structurally negating one of the four conditions.

### 6.3 THE OSTRICH ALGORITHM

Stick your head in the sand and pretend there is no problem. (hilarious)

### 6.4 DEADLOCK DETECTION AND RECOVERY

#### 6.4.1 Deadlock Detection with One Resource of Each Type

Deadlock detection with one resource of each type is simple by using a resource allocation graph which can be used to identify circle (deadlock)

Example:

1. Process A holds R and wants S.
2. Process B holds nothing but wants T.
3. Process C holds nothing but wants S.
4. Process D holds U and wants S and T.
5. Process E holds T and wants V.
6. Process F holds W and wants S.
7. Process G holds V and wants U.

We will have a graph like this

![alt text](deadlock-one.png "Deadlock graph")

From the directed graph on the left we can see that there is a cycle which means a deadlock.

#### 6.4.2 Deadlock Detection with Multiple Resources of Each Type

When there are multiple resources, directed graph will be not enough, so a resource matrix is used in this situation.

![alt text](deadlock-matrix.png "Deadlock matrix")

Each process is initially said to be unmarked.As the algorithm progresses, processes will be marked, indicating that they are able to complete and are thus not deadlocked. When the algorithm terminates, any unmarked processes are known to be deadlocked. 

The deadlock detection algorithm:
1. Look for an unmarked process, Pi, for which the ith row of R is less than or equal to A.
2. If such a process is found, add the ith row of C to A, mark the process, and go back to step 1.
3. If no such process exists, the algorithm terminates.

#### 6.4.3 Recovery from Deadlock

##### Recovery through Preemption

In some cases it may be possible to temporarily take a resource away from its current owner and give it to another process. In many cases, manual intervention may be required, especially in batch-processing operating systems running on mainframes.

##### Recovery through Rollback

If the system designers and machine operators know that deadlocks are likely, they can arrange to have processes **checkpointed** periodically.

To be most effective, new checkpoints should not overwrite old ones but should be written to new files, so as the process executes, a whole sequence accumulates.

##### Recovery through Killing Processes

### 6.5 DEADLOCK AVOIDANCE

The system must be able to decide whether granting a resource is safe or not and make the allocation only when it is safe. Thus, the question arises: Is there an algorithm that can always avoid deadlock by making the right choice all the time?

#### 6.5.1 Resource Trajectories

#### 6.5.2 Safe and Unsafe States

A state is said to be **safe** if there is some scheduling order in which every process can run to completion even if all of them suddenly request their maximum number of resources immediately.

Safe state:

![alt text](safe-state.png "Safe State")

Unsafe state (b):

![alt text](unsafe-state.png "Unsafe State")

#### 6.5.3 The Banker’s Algorithm for a Single Resource

Like above. The idea is that OS can reserve less resource than needed by all processes and carefully allocate them to prevent deadlock

#### 6.5.4 The Banker’s Algorithm for Multiple Resources

Instead of having one matrix, in this case, we need two matrixes just like in deadlock detection with multiple resources.

The algorithm for checking to see if a state is safe:

1. Look for a row, R, whose unmet resource needs are all smaller than or equal to A. If no such row exists, the system will eventually deadlock since no process can run to completion (assuming processes keep all resources until they exit).
2. Assume the process of the chosen row requests all the resources it needs (which is guaranteed to be possible) and finishes. Mark that process as terminated and add all of its resources to the A vector.
3. Repeat steps 1 and 2 until either all processes are marked terminated (in which case the initial state was safe) or no process is left whose resource needs can be met (in which case the system was not safe).

### 6.6 DEADLOCK PREVENTION

#### 6.6.1 Attacking the Mutual-Exclusion Condition

#### 6.6.2 Attacking the Hold-and-Wait Condition

##### Approach 1

* Each process requests all resources, before starting execution.
* If everything is available, all resources (requested by the process) are located and finish its job.
* If some resources are not available, no resources are located to the process.

Problems:
* Processes do not know what and how much they need.
* Resources cannot be used optimally.

##### Approach 2

* Allows a process to request resources only when the process has none
* To get a new resource, first, release all the resources currently holds and request all at time same time

Problems:
* Stavation is possible

#### 6.6.3 Attacking the No-Preemption Condition

##### Approach 1

* A process that is holding resources that is needed by another process and that process cannot be allocated, it will be preempted to release holding resources.
* A process will be restart only if there is enough resources.

#### 6.6.4 Attacking the Circular Wait Condition

##### Approach 1

* A process can only hold one resource
* If it needs another one, it must release the first one first.

##### Approach 1

* Having an ordered list of resources
* A process can request resources whenever they want, but all requests must made in numerical
order.
