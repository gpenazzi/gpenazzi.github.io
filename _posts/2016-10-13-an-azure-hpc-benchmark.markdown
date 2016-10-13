---
layout: post
title:  "An Azure HPC benchmark"
date:   2016-10-13
categories: [hpc]
tags: [hpc, dftb, azure, cloud computing]
---

This is a porting from an old post in a now deprecated (and dismissed) blog.

I've spent some time setting up a small HPC cluster on the Microsoft Azure
service. It was fun, it actually worked and did not take too much time.
I won't report the whole procedure here because the administration tools
are changing quickly and I believe that my procedure is already outdated,
I am just publishing the results.

I have tried to run a DFTB+ benchmark. DFTB+ is an academic Fortran
implementation of the Density Functional Tight Binding, an approximate quantum
mechanics method widely used in solid state physics and quantum chemistry.
Find it on <www.dftb-plus.info>.
The number crunching kernel of the code is a a complex generalized eigenvalue
problem solved through the SCALAPACK parallel library.
Note that I could not ensure full consistency in the compilation on different
platform, not the same compiler were available and I did not perform all the
calculations myself. The code on the Azure cloud was compiled using gfortran 4.8
with default optimization and the scalapack from the Intel mkl 11.2.

The benchmark is a self consistent calculation of a large supercell of
Silicon Carbide (SiC). For comparison, I got some data run on the Hannover
supercomputer (details on <http://www,hlrn.de>). At 2015 the Cray system at HLRN
 was ranked 83 in the world, according to http://www.top500.org. This benchmark
  was kindly provided by Balint Aradi.

![DFTB on HLRN scaling]({{ site.github.url }}/resources/images/dftbscaling1.png)

The XC30 is the Cray system. The ICE1 is an old cluster and it is significantly
slower. I could not retrieve the exact input file used for the figures above nor
 I have access to the hlrn to run the benchmark from scratch; I set up a SiC
 supercell with the same number of atoms but in my case the SCC iteration
 terminates in a smaller number of steps. I did not want to modify the input
 file to match the number of iterations because in that way I could have not
 stay in my planned cost.

I ran the same calculation on our institute (BCCMS) cluster as well, a now 5
years old Intel Xeon system with 40 Gb/s Infiniband interconnects, the cluster
performance is comparable with the ICE1 in Hannover. In the figure below the
results for Azure (using the Infiniband instances) and for our cluster.

![DFTB on Azure scaling]({{ site.github.url }}/resources/images/azure_bench.png)

As you see I could benchmark up to 128 cores. The scaling is not bad, from the
Cray calculation we do not expect ideal scaling beyond 96 cores for the 4000
atom system. The scaling on the Azure system is somewhat less regular, I would
not wonder too much because the Cray of course is a dedicated system.
Nevertheless the Azure Cloud manage the 4096 atom calculation in less than 1000
seconds starting from 32 cores and above. Considering the difference in the
number of SCC cycles (5 vs 11), the results are overall comparable.

When I started I would not expect a behavior close to a dedicated HPC system:
the overhead due to the virtualization is small on paper but I was somewhat
skeptic. After seeing those numbers my idea is that a virtualized Cloud system,
provided the right interconnect, is competitive with a dedicated HPC solution!

The price is still somewhat high, though. The red line in the benchmark above
costs a few euro per point. This is interesting if the cpu/hour cost can be
compensated by a smart dynamic resource allocation, but it stays the fact that
a “realistic” calculation can easily take some cpu days, and it may have a price
 in the range between 100 and 1000 Euro. I have not a clear estimate of the cost
of a dedicated HPC system, the point is that in academy the access to HPC is not
 regulated in the same way access to direct funding (read employs, travelling
funds etc.) is. To some extent, getting computational time is fairly easier
than getting a PhD grant.
