# Modern way of optimizing garbage collection in Minecraft: Java Edition

## Introduction
Garbage collection is the process by which programs try to free up allocated memory that is no longer used by objects. An object is considered ‘in-use’ or ‘referenced’ if some part of our program still maintains a pointer to it. Conversely, an ‘unused’ or ‘unreferenced’ object is no longer accessed by any part of our program, allowing the memory it occupies to be reclaimed. Garbage collection is an essential part of Java’s memory management system and is by default managed by the Java Virtual Machine (JVM). In the JVM, the garbage collectors are responsible for freeing up the heap memory, which is where Java objects are stored. This helps to ensure efficient resource usage. It also frees developers from having to manually manage the program’s memory.

There are many different approaches to memory management, and there is no “best” way to do it. Even in a single language/runtime, there can be more than one approach to garbage collection, and the JVM is a great example of that.

Instead of a singular garbage collector, modern JVM has 5 to choose from:<br>
•	Garbage-First (G1) Garbage Collector (the default since Java 9)<br>
•	Serial Garbage Collector<br>
•	Parallel Garbage Collector<br>
•	Shenandoah Garbage Collector (Java 12+)<br>
•	Z Garbage Collector (available for production since Java 15)<br>


## Z Garbage Collector (ZGC)
The Z Garbage Collector, also known as ZGC, is a scalable, low-latency garbage collector. It was first introduced in Java 11 as an experimental feature and became production-ready in Java 15. Its focus is ultra-low latency and scalability. This makes it ideal for web applications or applications that must handle a large amount of data, like Minecraft. It is almost completely concurrent and has low pause times of under 1ms. However, by default it takes a non-generational approach to garbage collection. This means it stores all objects together, regardless of their age, so each garbage collection cycle it collects all objects, which is demanding on CPU resources and is not recommended for smaller systems.

## Generational Garbage Collection
In the context of memory management, a generation refers to a categorization of objects based on the time of their allocation. Let’s shift our focus to generational garbage collection. This represents a memory management strategy that works by dividing the objects into different generations, based on the time of allocation, and applying different approaches based on their generation.

In the context of Java, the memory is partitioned into two main generations: young and old. Newly created objects find their place in the young generation, where frequent garbage collection takes place. Objects that persist beyond multiple garbage collection cycles are promoted to the older generation. This division optimizes efficiency by acknowledging the short lifespan of most of the objects.

Garbage-First Garbage Collector (G1GC) is a generational garbage collector, and until recently was the only generational JVM garbage collector. This is why it has been the best choice for Minecraft for over a decade. But since the release of Java 21 there is a new alternative.

## Generational Z Garbage Collector (ZGC)
The biggest changes in Java 21 have come to the Z garbage collector. Prior to Java 21, ZGC was only a non-generational garbage collector. This meant that it did not divide the heap into generations so when performing a scan, it analyzed the whole heap, which in return made the resources needed much greater than G1GC. It also ran into an issue where the application could allocate memory faster than the GC could reclaim from dead objects, which in return made Java threads stuck waiting for memory.  This was offset by two solutions, setting a larger heap size (which meant the application was spending more time doing the garbage collection, further reducing throughput) and increasing the number of threads for the garbage collector to make it run faster (which in return is taking threads of the application to use). Neither of these were perfect so in Java 21 work was done to allow ZGC to use generational garbage collection.

## Using Generational ZGC in Minecraft
As previously said, Generational ZGC was only added in Java 21. Minecraft requires Java 21 since version 1.20.5, which came out in April 23, 2024. To use Generational ZGC in older Minecraft versions you have to download Java 21 yourself and set it as the default Java runtime for each of your Minecraft client instances you want to use it on, or if you are running a Minecraft server – make it the Java runtime ran in your start-up script. Keep in mind that older Minecraft versions, notably ones before 1.17, may not work as intended with Java 21.

Enabling Generational ZGC is very straightforward. Just add `-XX:+UseZGC -XX:+ZGenerational` to your Java arguments either in your Minecraft client or your server’s start-up script in between `java` and `-jar`. If you are running Java 23 or above the `XX:+ZGenerational` is not needed anymore because it is on by default.

## Tuning Generational ZGC
ZGC has been designed to be adaptive and to require minimal manual configuration. During the execution of the Java program, ZGC dynamically adapts to the workload by resizing generations, scaling the number of GC threads, and adjusting tenuring thresholds. Because of this many arguments used to tune G1GC, like `-XX:ConcGCThreads=`, either do not work with ZGC or do not provide any benefits.  But there are still things you need to consider when using it.

###	Setting the maximum heap size
The most important tuning option for ZGC is setting the maximum heap size, essentially a limit of how much memory the JVM can use. This is done with the `-Xmx={memory}M` where `{memory}` is the amount of RAM you want to allocate to your Minecraft instance in megabytes. Because ZGC is a concurrent collector, you must select a maximum heap size such that the heap can accommodate the live-set of your application and there is enough headroom in the heap to allow allocations to be serviced while the GC is running. This means you should not set the maximum heap size to entire amount of memory available on your system.

### Returning Unused Memory to the OS
By default, ZGC uncommits unused memory, returning it to the operating system. This, however, may be undesirable for Minecraft servers because it can have a negative impact on the latency of Java threads. Best way to go about it is by setting your minimum heap size `-Xms` to the same value as your maximum heap size `-Xmx`, which effectively disables this. Some Minecraft server hosting providers make this simply impossible, however. In which case the only other way to achieve similar results is by adding `-XX:-ZUncommit` to your start-up arguments.

### Reducing latency further
Not only uncommitting but also committing memory has a negative impact on the latency of Java threads. This is why we should also use the `-XX:+AlwaysPreTouch` argument which will cause the JVM to page in memory before the application starts, which can further reduce latency.

### NUMA support
ZGC has NUMA support, which means it will try its best to direct Java heap allocations to NUMA-local memory. NUMA stands for Non-Uniform Memory Access and refers to the architecture design used in multi-socket systems. In NUMA systems, memory is divided into multiple memory nodes, with each node associated with a specific processor or socket. Each processor has faster access to its own local memory node compared to accessing remote memory nodes.

By default, ZGC enables NUMA support, allowing it to leverage the benefits of NUMA architectures. When running on a NUMA machine (e.g. a multi-socket x86 machine), having NUMA support enabled will often give a noticeable performance boost. However, if the JVM detects that it is bound to use memory on a single NUMA node, NUMA support will be disabled. Even though explicitly enabling NUMA support is possible it will not provide any benefits as some might suggest.

## String Deduplication
String deduplication is a JVM feature that has been around for quite a while now. It helps reduce Java heap memory usage by automatically deduplicating identical character arrays that are backing String objects. In the case of Minecraft, it can slightly help reduce the heap memory usage. It used to only work with G1GC, but in Java 18 a huge chunk of it was rewritten to also support ZGC. It is also more beneficial to use it with ZGC, because it does the string deduplication concurrently. To enable this feature simply add `-XX:+UseStringDeduplication` to your start-up arguments.

## Enabling Transparent Huge Pages (THP) on Linux
Large pages, or sometimes huge pages, is a technique to reduce the pressure on the processors TLB caches. These caches are used to speed up the time to translate virtual addresses to physical memory addresses. 

Configuring ZGC to use large pages will generally yield better performance (in terms of throughput, latency and start up time) and comes with no real disadvantage, except that it is slightly more complicated to setup. Note that not every Linux machine provided by hosting providers will allow the use of Transparent Huge Pages when using ZGC, in which case the process below will not give any benefits.

The setup process requires you to have complete root access to your Linux machine. For convenient we will be using Transparent Huge Pages (THP). First thing you must do is run these commands in your terminal:<br>
`echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled`<br>
`echo advise | sudo tee /sys/kernel/mm/transparent_hugepage/shmem_enabled`<br>
`echo 1 | sudo tee /sys/kernel/mm/transparent_hugepage/khugepaged/defrag`<br>
`echo defer | sudo tee /sys/kernel/mm/transparent_hugepage/defrag`

Now let’s go over what each of those do:<br>
`echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled` enables madvice mode.<br>
`echo advise > /sys/kernel/mm/transparent_hugepage/shmem_enabled` enables shmem huge pages for the heap.<br>
`echo 1 | sudo tee /sys/kernel/mm/transparent_hugepage/khugepaged/defrag` enables defragmentation of huge pages.<br>
`echo defer | sudo tee /sys/kernel/mm/transparent_hugepage/defrag` configures THP to defer defragmentation of huge pages.<br>

After you are done the only thing left is adding `-XX:+UseTransparentHugePages` to your start-up arguments. You should also make sure your `-Xms` equals your `-Xmx` value, adding `-XX:-ZUncommit` is not an option.

## Conclusion
The ZGC has an excellent “out-of-the-box” experience. The addition of generations makes it even more versatile and a great option for running Minecraft. 

Simply starting the game with Java 21 or above and adding `-Xms{memory}M -Xmx{memory}M -XX:+UseZGC -XX:+ZGenerational -XX:+AlwaysPreTouch -XX:+UseStringDeduplication` to the start-up arguments, where `{memory}` is the amount of RAM in megabytes you would like to allocate, should be the most optimal way of optimizing garbage collection. 

If setting -Xms is not possible then adding `-XX:-ZUncommit` is the next best option. For people running a Linux machine enabling Transparent Huge Pages can also be beneficial.

## Sources
https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector.html<br>
https://malloc.se/blog/zgc-jdk18<br>
https://dataintellect.com/blog/low-latency-java-optimisation-through-garbage-collector-tuning/<br>
https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b<br>
https://belief-driven-design.com/looking-at-java-21-generational-zgc-e5c1c/<br>
https://www.baeldung.com/java-21-generational-z-garbage-collector<br>
