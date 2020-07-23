---
title: 'Likwid'
date: 2020-07-14
collection: blog
type: "blog"
permalink: /blog/:title
excerpt_separator: <!--more-->
tags:
  - Tech
  - Tool
---

> Likwid is a simple to install and use toolsuite of command line applications for performance oriented programmers.

<!--more-->

Follow the [guide](https://github.com/RRZE-HPC/likwid#download-build-and-install) to install Likwid, it has the following tools:
- likwid-topology
- likwid-perfctr
- likwid-pin
- likwid-bench
- likwid-powermeter
- likwid-memsweeper
- likwid-mpirun
- likwid-perfscope
- likwid-setFrequencies
- likwid-genTopoCfg
- likwid-features

### likwid-topology

- `likwid-topology -h `  #help msg

- `likwid-topology` #standard msg. Thread, Core, Socket, Caches, Memory

- `likwid-topology -c` #More about cache

- `likwid-topology -g` #Graphical output

  ```bash
  Example output:
  --------------------------------------------------------------------------------
  CPU name:	Intel(R) Core(TM) i7-4770 CPU @ 3.40GHz
  CPU type:	Intel Core Haswell processor
  CPU stepping:	3
  ********************************************************************************
  Hardware Thread Topology
  ********************************************************************************
  [...]
  ********************************************************************************
  Cache Topology
  ********************************************************************************
  [...]
  ********************************************************************************
  NUMA Topology
  ********************************************************************************
  [...]
  ********************************************************************************
  Graphical Topology
  ********************************************************************************
  Socket 0:
  +-----------------------------------------+
  | +-------+ +-------+ +-------+ +-------+ |
  | |  0 4  | |  1 5  | |  2 6  | |  3 7  | |
  | +-------+ +-------+ +-------+ +-------+ |
  | +-------+ +-------+ +-------+ +-------+ |
  | |  32kB | |  32kB | |  32kB | |  32kB | |
  | +-------+ +-------+ +-------+ +-------+ |
  | +-------+ +-------+ +-------+ +-------+ |
  | | 256kB | | 256kB | | 256kB | | 256kB | |
  | +-------+ +-------+ +-------+ +-------+ |
  | +-------------------------------------+ |
  | |                 8MB                 | |
  | +-------------------------------------+ |
  +-----------------------------------------+
  ```



### likwid-perfctr

CPU support can be found [here](https://github.com/RRZE-HPC/likwid/wiki/likwid-perfctr#supported-architectures)

Default install is access daemon mode, to change to direct access mode, follow instructions [here](https://github.com/RRZE-HPC/likwid/wiki/likwid-perfctr#prerequisites-for-direct-access-mode)

- `likwid-perfctr -h` #help msg

- `likwid-perfctr -a` #list all performance group

- `likwid-perfctr -e | less` #will show all available events and counter registers, 

- `likwid-perfctr -i` #CPU info, also has supported CPU list

- `likwid-perfctr -H -g MEM` #help info for group MEM, use `likwid-perfctr -a` to show all groups

- `likwid-perfctr -C S0:0 -g MEM ./a.out` #performance data for Socket-0 Core-0 and group MEM, will pin the application to S0:0. if not specified, will show performance data on for cores.

- `likwid-perfctr -C S0:0-3 -g L2 -g L3 -T 500ms ./a.out` #can specify more than one groups, but they will be switched and measured in a round-robin fashion, default switch time is 2s, -T can change the switching time, note that if switch time is longer than execution time, only only the first group has the chance to be measured.

- Use the Marker API

  ```c
  // This block enables to compile the code with and without the likwid header in place
  #ifdef LIKWID_PERFMON
  #include <likwid.h>
  #else
  #define LIKWID_MARKER_INIT
  #define LIKWID_MARKER_THREADINIT
  #define LIKWID_MARKER_SWITCH
  #define LIKWID_MARKER_REGISTER(regionTag)
  #define LIKWID_MARKER_START(regionTag)
  #define LIKWID_MARKER_STOP(regionTag)
  #define LIKWID_MARKER_CLOSE
  #define LIKWID_MARKER_GET(regionTag, nevents, events, time, count)
  #endif
  
  LIKWID_MARKER_INIT;
  LIKWID_MARKER_THREADINIT;
  
  LIKWID_MARKER_START("Compute"); // Compute can be any name you want
  								// You can have multiple regions
  // ---------------------------
  //	Your code to measure
  //	-------------------------- 
  LIKWID_MARKER_STOP("Compute");
  LIKWID_MARKER_CLOSE;
  ```

  ```bash
  #use the following command to compile
  gcc/g++ -O3 -fopenmp -pthread -o test test.c/cpp -DLIKWID_PERFMON -I<PATH_TO_LIKWID>/include -L<PATH_TO_LIKWID>/lib -llikwid -lm
  
  #in my case, the path to include/ and lib is slightly different, maybe due to version, the following worked for me:
  gcc/g++ -O3 -fopenmp -pthread -o test test.c/cpp -DLIKWID_PERFMON -I<PATH_TO_LIKWID>/src/includes -L<PATH_TO_LIKWID> -llikwid -lm
  
  #add -m to activate the Marker API
  likwid-perfctr -C S0:0 -g MEM ./test    # Same as not compiled with Marker API
  likwid-perfctr -C S0:0 -g MEM -m ./test # API activated
  ```

  The Marker API works for c/c++ Fortran, there are unofficial support for [python](https://github.com/RRZE-HPC/pylikwid), but I haven't tried that.



### likwid-pin

A tool for multi-threading analysis. Used to pin a thread to certain core. Can be used with the other tools, for example, `likwid-powermeter` does not provide options to pin thread to a core, but this is can be achieved using together with `likwid-pin`

From [likwid-pin wiki](https://github.com/RRZE-HPC/likwid/wiki/Likwid-Pin#introduction): For threaded applications on modern multi-core platforms **it is crucial to pin threads to dedicated cores**. While the Linux kernel offers an API to pin your threads, it is tedious and involves some coding to implement a flexible solution to address affinity. Intel includes an sophisticated pinning mechanism for their OpenMP implementation. While this already works quite well out of the box, it can be further controlled with environment variables.

I haven't try this tool out yet. May come back when I need this some day.



### likwid-bench

From [likwid-bench wiki](https://github.com/RRZE-HPC/likwid/wiki/Likwid-Bench#introduction): likwid-bench is a **benchmarking application** together with a framework to enable rapid prototyping of multi-threaded **assembly kernels**. Adding a new benchmark amounts to creating a simple text file and recompiling. The framework takes care of threaded execution and pinning, data allocation and placement, time measurement and result presentation.



### likwid-powermeter

Measure power consumed, and TDP info.

- `likwid-powermeter -h` #help msg

- `likwid-powermeter -i` #print MSR_PKG_POWER_INFO register

- `likwid-powermeter -t` #show temperature

- `likwid-powermeter -s 3s` #measure for 3 seconds

- `likwid-powermeter ./a.out` #measure power for all socket

- `likwid-powermeter -c 0` #measure socket 0 only. Note -c specify socket not core, power measurement is in the granularity of socket.

- `likwid-powermeter -c 0 likwid-pin S0:0-3 ./a,out` #work with likwid-pin, pin thread on Socket 0 core 0-3, and measure only power for socket 0

  ```bash
  # An output example
  --------------------------------------------------------------------------------
  CPU name:       Intel(R) Xeon(R) CPU E3-1241 v3 @ 3.50GHz
  CPU type:       Intel Core Haswell processor
  CPU clock:      3.49 GHz
  --------------------------------------------------------------------------------
  Sleeping longer as likwid_sleep() called without prior initialization
  Sleeping longer as likwid_sleep() called without prior initialization
  --------------------------------------------------------------------------------
  Runtime: 6.77409 s
  Measure for socket 0 on CPU 0
  Domain PKG:
  Energy consumed: 142.928 Joules
  Power consumed: 21.0993 Watt
  Domain PP0:
  Energy consumed: 74.5991 Joules
  Power consumed: 11.0124 Watt
  Domain PP1:
  Energy consumed: 0 Joules
  Power consumed: 0 Watt
  Domain DRAM:
  Energy consumed: 12.448 Joules
  Power consumed: 1.83759 Watt
  --------------------------------------------------------------------------------
  ```

  PP1 has some issue giving the correct results. I don't know why, in fact, I don't even know what's PP0 and PP1, according to [doc](https://rrze-hpc.github.io/likwid/Doxygen/group__PowerMon.html#ga0d0e3226055ad5266e487b58fd01cebd), it's not clear defined by intel

  ```bash  
  PKG 		PKG domain, mostly one CPU socket/package.
  PP0 		PP0 domain, not clearly defined by Intel.
  PP1 		PP1 domain, not clearly defined by Intel.
  DRAM 		DRAM domain, the memory modules.
  PLATFORM 	PLATFORM domain, the whole system (if powered through the main board)
  ```


### likwid-memsweeper 

A tool to cleanup ccNUMA domains


### likwid-mpirun 

Enable simple pinning for MPI and hybrid MPI/threaded applications. I have no experience with MPI, leave for now.


### likwid-perfscope

A tool to perform live plotting of performance data.


### likwid-setFrequencies

A tool to manipulate the frequency of CPU cores


### likwid-getTopoCfg

likwid-genTopoCfg is a command line application that stores the system's CPU, Cache and NUMA topology to a file. LIKWID applications use this file to read in the topology fast instead of re-gathering all values. The path to the topology file can be set in the global configuration file likwid.cfg. Furthermore, some default paths are checked:

- /etc/likwid_topo.cfg
- <INSTALLED_PREFIX>/etc/likwid_topo.cfg


### likwid-features

CPUs provide hardware features that perform their duty under the hood. The features are commonly prefetchers that load data in the caching hierarchy before it is needed by an application. Intel CPUs provide a register that allows to (de)activate these features during runtime. The likwid-features tool is a simple command line application to do that.

- It's pretty straight-forward, use `likwid-features -h` and just follow help msg


### Side Notes

When I compare the results from likwid-perfctr with perf stat, the number doesn't quite match. For example, 

```bash
perf stat -e instructions -- ls
# output:
# Performance counter stats for 'ls':
#
#         2,385,416      instructions
#
#       0.001127465 seconds time elapsed

likwid-perfctr -C S0:0 -g MEM ls
# output:
# --------------------------------------------------------------------------------
# Group 1: MEM
# +-----------------------+---------+--------+
# |         Event         | Counter | Core 0 |
# +-----------------------+---------+--------+
# |   INSTR_RETIRED_ANY   |  FIXC0  | 727250 |
# | CPU_CLK_UNHALTED_CORE |  FIXC1  | 910004 |
# |  CPU_CLK_UNHALTED_REF |  FIXC2  | 816480 |
# |       DRAM_READS      | MBOX0C1 |   8941 |
# |      DRAM_WRITES      | MBOX0C2 |    540 |
# +-----------------------+---------+--------+
#
# +-----------------------------------+--------------+
# |               Metric              |    Core 0    |
# +-----------------------------------+--------------+
# |        Runtime (RDTSC) [s]        |       0.0015 |
# |        Runtime unhalted [s]       |       0.0003 |
# |            Clock [MHz]            |    3891.9509 |
# |                CPI                |       1.2513 |
# |  Memory load bandwidth [MBytes/s] |     373.6813 |
# |  Memory load data volume [GBytes] |       0.0006 |
# | Memory evict bandwidth [MBytes/s] |      22.5688 |
# | Memory evict data volume [GBytes] | 3.456000e-05 |
# |    Memory bandwidth [MBytes/s]    |     396.2501 |
# |    Memory data volume [GBytes]    |       0.0006 |
# +-----------------------------------+--------------+
```

This is because likwid is in user space, when running perf stat with event in user level, the number is much closer:

```bash
perf stat -e instructions:u -- ls 
# output:
# Performance counter stats for 'ls':
#
#           714,112      instructions:u
#
#       0.001125405 seconds time elapsed
```

perf has event modifiers, as listed below: [link](https://perf.wiki.kernel.org/index.php/Tutorial)

```bash
Modifiers    Description                                                  Example
u            monitor at priv level 3, 2, 1 (user)                         event:u
k            monitor at priv level 0 (kernel)                             event:k
h            monitor hypervisor events on a virtualization environment    event:h
H            monitor host machine on a virtualization environment         event:H
G            monitor guest machine on a virtualization environment        event:G
```

<br>
References:<br>
[https://github.com/RRZE-HPC/likwid](https://github.com/RRZE-HPC/likwid) <br>
[https://perf.wiki.kernel.org/index.php/Tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)

