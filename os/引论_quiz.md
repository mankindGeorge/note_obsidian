# OS Quiz

## Question 1
Which of the following goals of an operating system specifically focuses on ensuring that system resources like the CPU and memory are used as much as possible?

- [ ] Convenience
- [x] Effectiveness
- [ ] Expandability
- [ ] Openness

**Hint:** Think about the metric used to measure how 'busy' the system remains while processing tasks.

## Question 2
What was the primary bottleneck in 'Manual Operating Systems' that led to the development of Off-line I/O?

- [ ] Lack of user interface
- [x] CPU idleness during human operation
- [ ] Frequent system crashes
- [ ] High memory costs

**Hint:** Consider the disparity in speed between a human hand and an electronic processor.

## Question 3
In a Multi-programmed Batch Processing System, what is the main purpose of keeping multiple jobs in memory simultaneously?

- [ ] To allow users to interact with their programs in real-time
- [ ] To reduce the physical size of the computer
- [x] To enable the CPU to switch to another task when the current one performs I/O
- [ ] To prevent unauthorized access to system files

**Hint:** Consider what happens to the CPU when a single-program system needs to read from a disk.

## Question 4
Which characteristic of a Time-Sharing System ensures that each user feels as though they have exclusive use of the computer?

- [x] Independence
- [ ] Multi-channeling
- [ ] Timeliness
- [ ] Interactivity

**Hint:** Focus on the psychological effect of the system's design on the individual user's experience.

## Question 5
How does a Hard Real-Time System differ from a Soft Real-Time System?

- [ ] Hard real-time systems only support one task at a time
- [x] Soft real-time systems allow for missed deadlines without total system failure
- [ ] Hard real-time systems are used primarily for office automation
- [ ] Soft real-time systems use a slower CPU

**Hint:** Think about the consequences of a task completing slightly after its intended 'deadline'.

## Question 6
In operating systems, what is the fundamental difference between 'Parallelism' and 'Concurrency'?

- [ ] Concurrency requires a multi-core processor, while parallelism does not
- [x] Parallelism occurs at the same instant, whereas concurrency occurs over a time interval
- [ ] Concurrency refers to multiple users, while parallelism refers to multiple tasks
- [ ] There is no difference between the two terms

**Hint:** Consider the difference between doing two things at the exact same moment versus switching between them very quickly.

## Question 7
Which of the following resources is typically accessed through 'Mutual Exclusion' sharing?

- [ ] A Read-Only data file
- [x] A Printer
- [ ] Physical RAM
- [ ] A Hard Drive seeking a file

**Hint:** Think of a device where interleaving the data from two different users would result in a useless physical product.

## Question 8
What is 'Asynchrony' in the context of multi-programmed systems?

- [ ] The ability of the OS to run without a clock signal
- [x] The unpredictable speed and progress of concurrently running processes
- [ ] A technique to make the CPU run faster than its rated speed
- [ ] The requirement that all programs must finish at the same time

**Hint:** Consider how the limited number of resources causes processes to pause and resume at irregular intervals.

## Question 9
In Memory Management, what is the primary purpose of 'Address Mapping'?

- [ ] To find the physical location of a file on the disk
- [x] To convert logical addresses used by a program into actual physical RAM addresses
- [ ] To increase the clock speed of the memory bus
- [ ] To ensure that every process uses the exact same physical memory address

**Hint:** Think about the translation needed when a program's 'internal' view of memory doesn't match the 'real' location in hardware.

## Question 10
Which OS structural design principle focuses on 'Mechanism and Policy Separation' to keep the kernel as small as possible?

- [ ] Monolithic Structure
- [x] Microkernel Structure
- [ ] Layered Structure
- [ ] Modular Structure

**Hint:** Identify the structure that aims to move services like file systems and device drivers out of the core kernel.
