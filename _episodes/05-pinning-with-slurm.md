---
geometry: margin=2cm
pagestyle: empty
colorlinks: true
---

# Thread pinning and distribution in Slurm

## Intended learning outcomes

By the end of this lesson, you should be able to:

- Describe thread pinning and distribution and why they're useful in modern HPC systems
- Understand how pinning and distribution are used to enhance performance
- Analyse and understand the hardware hierarchy of a particular system
- Use Slurm to pin MPI processes to particular CPU sockets or cores

## What is thread pinning?

*Pinning*, also known as *affinity* or *binding*[^definitions] is a feature of multicore systems where the active threads or processes of an application are *pinned* to certain physical processors or cores for the entire runtime of the application. This is useful because each physical CPU core in a HPC system does not have equal access to all resources. Some systems have two or more CPUs in a single node, so some regions of main memory are better connected to one CPU and have faster access times as a result. It's common for systems to have one CPU but multiple NUMA zones: regions of main memory with Non-Uniform Memory Access. In these systems, groups of cores have faster access to certain parts of main memory. The memory is still technically accessible by all cores, but a core accessing memory in a region not associated with it will incur some extra access time.

[^definitions]: Confusingly, there is not a consistent term to describe thread pinning. You may see any combination of thread, process and CPU along with binding, affinity and pinning.

To illustrate the performance impact of pinning, imagine a node with two sockets, i.e. two distinct physical CPUs. Each CPU has fast access to a local bank of main memory but slower access to the memory local to the other CPU. If a thread on this system isn't pinned, that is the system is allowed to migrate threads between CPUs during runtime, it could initially allocate and use some space on the fast memory associated with its initial CPU, then be migrated across to the other CPU, where accessing the same block of memory will be significantly slower. By pinning this thread to a single CPU, or better a single core, we avoid this potentially slow memory access.

Of course, there will still be circumstances where a thread must access data in the memory bank that *isn't* closest to its pinned CPU. Pinning is not a guarantee of fast memory access; it merely stops threads being unexpectedly migrated away from their local, fast memory.

::: Challenge
Can you convince yourself that a system with one CPU but multiple NUMA zones could suffer from the same performance issues if threads were not pinned and allowed to migrate between NUMA zones?
:::

## Other uses of thread pinning

Memory is not the only resource in a HPC system that might be accessed in an unequal way by different cores. PCIe slots can also offer faster access to certain cores or processors. Since these slots can host performance-critical components like GPUs or network cards, pinning can ensure threads access the same resources consistently and optimally over the entire runtime of the application.

Again, to illustrate the performance impact of pinning, imagine the same two-CPU system as we considered previously but now each CPU has a GPU associated with it. That is, each CPU has fast access to its closest GPU, but slower access to GPUs associated with the other CPU. A thread might initialise on, say, CPU 0[^cpu0], allocate some space on its local memory bank, and transfer some data from there to the associated GPU 0. This memory transfer will be as fast as possible. Without pinning, this same thread could be migrated to CPU 1. It would then slowly access the same memory space on CPU 0's memory bank, but even worse, it may move data to GPU 0. Some systems may be able to notice that the memory transfer can take a faster path without going through CPU 1, but the worst-case scenario is the data is transferred between CPUs twice: once into CPU 1 then back again on its way to GPU 0.

[^cpu0]: On HPC systems, CPUs, cores, GPUs and most other pieces of hardware are usually indexed from 0. So a two-CPU system would likely report CPU 0 and CPU 1.

## Understanding system details with `hwloc-ls`

It's helpful to understand the details of the specific system we want to target, particularly to understand the number of cores available and how they are distributed between sockets and NUMA zones. The command `hwloc-ls` gives us this information. It should be available on all DiRAC systems but if the specific system or node you're using doesn't offer it, you may be able to use `nproc` and `numactl -s` as alternatives.

Here's a Slurm script that we can use to print out the details of the CPU(s) on our worker node with `hwloc-ls`:

```bash
#!/usr/bin/env bash

#SBATCH --nodes=1

#SBATCH -o hwloc-ls.out
#SBATCH --exclusive
#SBATCH -p cosma8
#SBATCH -t 00:10:00
#SBATCH -A <PROJECT_CODE>

hwloc-ls
```

You should copy this into a script named something like `hwloc-ls.sh` and submit it with `sbatch hwloc-ls.sh`[^project_code]. Once this runs, you should have an output file `hwloc-ls.out` with contents similar to our example output below. The output can seem overwhelming so let's go through the example in pieces.

[^project_code]: In the Slurm submission script you will need to replace the text `<PROJECT_CODE>` with a real project code, which you require to gain access to any DiRAC system, and can usually find through the SAFE system.

The tool organises its output to represent the hardware hierarchy of the entire node. It's slightly easier to understand this if we have an expectation of what the hardware actually is so I'll spoil the surprise and summarise the hardware on this specific node. This node has two CPUs, each with 64 cores, grouped into NUMA zones of 16 cores each, for a total of 8 NUMA zones. Within each group of 16 cores, two subgroups of 8 cores share a single L3 cache, so there are two L3 caches per NUMA zone. Some groups of cores are directly attached to PCIe slots, with some of those connected to video or network cards.

Here, the output has been annotated with comments to describe each level, and some longer sections have been edited out and replaced with `...`.

```bash
Machine (1007GB total)  # The entire node
  Package L#0  # A single socket or CPU. Notice there is a Package L#1 listed lower down, because this system has two CPU sockets.
    Group0 L#0  # A group of cores within one CPU that has a number of resources directly linked to it.
      NUMANode L#0 (P#0 125GB)  # The region of memory directly linked to this collection of cores
      L3 L#0 (32MB)  # The L3 cache available to just 8 cores in this group.
        L2 L#0 (512KB) + L1d L#0 (32KB) + L1i L#0 (32KB) + Core L#0  # The collection of caches associated with a single *physical* core
          PU L#0 (P#0)  # Logical core number 1
          PU L#1 (P#128)  # And logical core number 2
        L2 L#1 (512KB) + L1d L#1 (32KB) + L1i L#1 (32KB) + Core L#1  # 2nd physical core
          PU L#2 (P#1)
          PU L#3 (P#129)

        ... # Similar details of cores 2-6

        L2 L#7 (512KB) + L1d L#7 (32KB) + L1i L#7 (32KB) + Core L#7  # The 8th core in this L3 cache group
          PU L#14 (P#7)
          PU L#15 (P#135)
      L3 L#1 (32MB)  # The other L3 cache in this group, serving the other 8 cores
        ... # The other 8 cores in the entire group of 16
      HostBridge  # The video card plugged into the PCI slot directly linked to this core group.
        PCIBridge
          PCIBridge
            PCI 62:00.0 (VGA)
    Group0 L#1  # Another group of 16 nodes
      NUMANode L#1 (P#1 126GB)  # With another NUMA zone attached
        ...  
      HostBridge  # This group is linked to a local storage drive: "sda"
        PCIBridge
          PCI 43:00.0 (SATA)
            Block(Disk) "sda"
    Group0 L#2
        ...
      HostBridge  # This group is connected to the high-bandwidth InfiniBand network card
        PCIBridge
          PCI 21:00.0 (InfiniBand)
            Net "ibp33s0"
            OpenFabrics "mlx5_0"
    Group0 L#3
        ...
  Package L#1  # This details the hardware associated with the 2nd CPU socket
    Group0 L#
        ...
      HostBridge  # This group is connected to the ethernet network card
        PCIBridge
          PCI e1:00.0 (Ethernet)
            Net "em1"
    Group0 L#5
        ...
    Group0 L#6
        ...
    Group0 L#7
        ...
```


## Exploring Slurm pinning options with `taskset`

The tool `taskset` provides information on the pinning settings currently applied to a running process.

Here's a Slurm script that we can edit and submit to explore the effects of different Slurm options on pinning:

```bash
#!/usr/bin/env bash

#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8

#SBATCH --exclusive
#SBATCH -p cosma8
#SBATCH -t 00:10:00
#SBATCH -A <PROJECT_CODE>

srun bash -c 'echo -n "task ${SLURM_PROCID} (node ${SLURM_NODEID}): "; taskset --cpu-list --pid ${BASHPID}' | sort
```

Again, copying this into a script called something like `print_pinning_info.sh` and submitting it with `sbatch print_pinning_info.sh` will produce an output file called something like `slurm-808080.out` with the contents:

```
task 0 (node 0): pid 2752726's current affinity list: 0-255
task 1 (node 0): pid 2752727's current affinity list: 0-255
task 2 (node 0): pid 2752728's current affinity list: 0-255
task 3 (node 0): pid 2752729's current affinity list: 0-255
task 4 (node 0): pid 2752730's current affinity list: 0-255
task 5 (node 0): pid 2752731's current affinity list: 0-255
task 6 (node 0): pid 2752732's current affinity list: 0-255
task 7 (node 0): pid 2752733's current affinity list: 0-255
```

In the submission script we requested 8 processes[^process_tasks], which have been assigned, however `taskset` reports that the current affinity is set to `0-255` on every process, that is these processes are allowed on any logical core with an index between 0 and 255. Since the entire system has a total of 256 logical cores, this means pinning is *not* enabled and the system is free to migrate these processes between any of its available cores.

[^process_tasks]: For simplicity, we will use tasks and processes interchangeably. 

### Using `--cpu-bind=cores`

**Edit the submission script to add the flag `--cpu-bind=cores` to the `srun` command.** The line containing `srun` should now look like:

```bash
srun --cpu-bind=cores bash ...
```

Submitting again should produce output like:

```
task 0 (node 0): pid 2753061's current affinity list: 0,128
task 1 (node 0): pid 2753062's current affinity list: 64,192
task 2 (node 0): pid 2753063's current affinity list: 1,129
task 3 (node 0): pid 2753064's current affinity list: 65,193
task 4 (node 0): pid 2753065's current affinity list: 2,130
task 5 (node 0): pid 2753066's current affinity list: 66,194
task 6 (node 0): pid 2753067's current affinity list: 3,131
task 7 (node 0): pid 2753068's current affinity list: 67,195
```

Notice that each process now has a very constrained affinity list, for example process 0 can only be assigned to core 0 or 128, i.e. the two logical cores associated with the physical core 0. It might seem odd that process 0 is assigned to core 0, and then process 1 is assigned to core 64. This is a consequence of the way Slurm initially distributes processes across cores. On this system, Slurm is trying to distribute processes so that they are balanced between the two CPU sockets. We'll explore process or thread distribution in a later section.

### Other pinning options

[The Slurm documentation](https://slurm.schedmd.com/mc_support.html) contains detailed information about more flags to control pinning, including further options for the `--cpu-bind` flag.

::: challenge
Edit your script and in the `--cpu-bind` flag, change `cores` to `sockets`, then resubmit the script. Consider what the output might look like.

Is the output what you expected? Why?
:::

## Pinning threads with OpenMP

Previously, we've used `srun` to pin MPI processes, but in hybrid MPI-OpenMP codes it can improve performance to also pin OpenMP threads. We can ensure these threads are pinned to OpenMP *places* by setting the environment variable `OMP_PROC_BIND`[^OMP_PROC_BIND] to `true`. The possible places in the OpenMP standard[^omp_standard] are `threads`, `cores` and `sockets`, all set in the environment variable `OMP_PLACES`. As a starting point, we recommend putting the following settings in your Slurm submission scripts, and then experimenting with the other options:

```bash
export OMP_PROC_BIND=true
export OMP_PLACES=cores
```

[^OMP_PROC_BIND]: See the [OMP_PROC_BIND documentation](https://www.openmp.org/spec-html/5.0/openmpse52.html#x291-20580006.4) for information on the more advanced pinning settings: `master`, `close`, and `spread`, all options that control the distribution of threads within places.

[^omp_standard]: Some implementations of OpenMP offer more sophisticated options for `OMP_PLACES`, like pinning to NUMA zones. You may wish to experiment with these options for extra performance gains.

## Thread distribution

While pinning threads ensures threads stick to the cores they're initially assigned, we can also control the way in which threads are initially distributed with *thread distribution*. By combining these techniques, we can have extremely precise control over which threads run on which cores in a system, and potentially unlock performance gains as a result.

*Thread distribution* is the way in which threads or processes are initially distributed amongst cores and sockets. This is generally only important when threads access resources, like the filesystem, GPUs, or memory, in an unbalanced way. For example, a multi-GPU simulation may sensibly assign one thread per GPU to process data transfers. It's more performant to ensure these specific threads are spread across cores in such a way that any data transfers are also spread across different transfer buses. This helps maximise the available bandwidth.

Let's look at some specific hardware. In our earlier walk-through of the `hwloc-ls` output, we found the InfiniBand network card attached to group L#2. Any cores within this group will have the fastest access to this card. If a piece of software is written in such a way that thread or process 0 is the process that communicates most with other nodes, it may improve performance to ensure that thread is initially distributed (and pinned) to group L#2, i.e. cores between 32-48.

A more advanced example involving GPUs is given in [LUMI's documentation](https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/distribution-binding/#gpu-binding), where users are advised to assign MPI ranks associated with certain GPUs to the NUMA zones that are directly linked to each GPU. In this case, MPI processes must be assigned to very specific CPU cores with the Slurm option `--cpu-bind=map_cpu:<map>`.

::: challenge

Edit the Slurm script from earlier and modify the `--cpu-bind` flag to use `map_cpu:...` to map MPI processes to specific cores. Your goal is to write an explicit mapping that results in a process distribution *matching* the output from `--cpu-bind=cores`.

:::: solution

`map_cpu` requires us to write a list of core IDs, where the first ID in the list will host thread 0, the second ID, thread 1, and so on. To match the distribution produced by `--cpu-bind=cores`, we need thread 0 to be assigned to core 0, thread 1 to core 64, thread 2 to core 1, thread 3 to core 65, and then alternating upwards through the cores. We'll ignore the affinity associated with the logical cores, since it doesn't affect performance. Since we have 8 processes to distribute, we should have 8 IDs in our list.

The actual flag is given to `srun` as:

```bash
srun --cpu-bind=map_cpu:0,64,1,65,2,66,3,67 ...
```

And the output matches our earlier use of `--cpu-bind=cores`:

```
task 0 (node 0): pid 2179244's current affinity list: 0
task 1 (node 0): pid 2179245's current affinity list: 64
task 2 (node 0): pid 2179246's current affinity list: 1
task 3 (node 0): pid 2179247's current affinity list: 65
task 4 (node 0): pid 2179248's current affinity list: 2
task 5 (node 0): pid 2179249's current affinity list: 66
task 6 (node 0): pid 2179250's current affinity list: 3
task 7 (node 0): pid 2179251's current affinity list: 67
```
::::
:::

## Other resources

- [OpenMP CPU affinity for GCC's libgomp](https://gcc.gnu.org/onlinedocs/libgomp/GOMP_005fCPU_005fAFFINITY.html)
- [LUMI resources on Process and Thread Distribution and Binding](https://lumi-supercomputer.github.io/LUMI-training-materials/2day-20240502/07_Binding/)
- [LUMI documentation on distribution and binding](https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/distribution-binding/)
- [ULHPC documentation on affinity and pinning](https://hpc-docs.uni.lu/jobs/affinity_and_pinning/)
