# JUQCS

## Purpose

JUQCS is a simulator for universal gate-based quantum computers. The state of a quantum computer is represented by a large tensor of complex numbers, the so-called statevector. Each step (called *gate*) in a quantum computer program corresponds to a linear tensor operation on the statevector. JUQCS simulates quantum computers by iteratively updating the statevector according to the computational model of gate-based quantum computers. The maximum number of qubits (i.e., the size of the largest quantum computer that can be simulated) is only limited by the total amount of memory available.

JUQCS is in-house software developed at JSC [[1]](#1). The program is written in CUDA Fortran and uses MPI and OpenMP for parallelization and has been run on many supercomputers in the world with a proven high level of scalability.

## Source

Archive name: `juqcs-bench.tar.gz`

A minimal version of JUQCS (_JUQCS Light_) that can be used to run the benchmark is available under the [`src`](src) directory as a submodule. It contains the source code, a Makefile, the input files for the benchmark, and submit script templates. To clone the repository including JUQCS Light, use `git clone --recurse-submodules`.

Please note the license requirements per `LICENSE` file. 

## Building

All commands necessary to use an MPI Fortran compiler to compile the minimal JUQCS source code are contained in the Makefile under the targets `qc22.gpu.exe` and `qc22.cpu.exe` for the GPU and the CPU version, respectively. Through the Makefile, the GPU version of JUQCS for example is compiled using
```
mpif90 -O2 -gpu=cc75,cc80 -mp -o qc22.gpu.exe qcpsigpu.cuf qcgpukernel.cuf modqc20.f90 qc22.f90 qc20libextra.f90 qcgpulib21.cuf qcgpuwrapper.cuf swapbits21gpu.cuf readinp20.f90 swapbits23.f90 generateevents20.f90
```

### Dependencies

JUQCS needs a CUDA Fortran compiler and an MPI infrastructure. On JUWELS, the necessary modules can be loaded with

```
module load Stages/2023
module load NVHPC/23.1 ParaStationMPI/5.7.0-1
```

### JUBE

JUBE scripts are provided to compile and run the JUQCS benchmarks in the [`benchmark/jube`](benchmark/jube) directory.

The scripts take care of loading the modules and compiling the source code automatically. Recompilation of the source code can be enforced using the optional tag `force_recompile` as in

```
jube run benchmark/jube/default.yaml [-t force_recompile]
```

Alternatively, run `make -C src/juqcs-light/src clean` before invoking any JUBE command (it is a good idea to always recompile, even though job submission will take longer, so that runs on different machines do not accidentally use the same executables).

_Note that for the MSA variant which uses the GPU and CPU executables at the same time (see below), the executables need to be manually built on the respective system module (for instance with the `makefile`)._

## Execution

The executable `qc22.gpu.exe` is the GPU version of JUQCS that uses CUDA and is meant to run on a GPU node.

The executable `qc22.cpu.exe` is the CPU version of JUQCS. It needs to be compiled under the right environment.

The default benchmark uses only the first executable. The modular system architecture (MSA) benchmark needs both executables.

All quantum circuit input files necessary for JUQCS to run the benchmarks are located in the directories [`src/juqcs-light/src/input`](src/juqcs-light/src/input) and [`src/juqcs-light/src/input-long`](src/juqcs-light/src/input-long).

### Modification

We use JUQCS with optimization level `-O2`. A higher level like `-O3` has generally not been found to produce run times that vary by more than 1-2 percent.
Although these optimizations are not extensively tested, we do not believe that errors due to fast math should affect the results (see the verification procedure below to check).

### Multi-Threading

The GPU version of the code uses MPI ranks for distribution and CUDA for parallelization. Each GPU is managed by one MPI rank. When the simulation work on the GPUs is done, the results are processed on the host using 12 OpenMP threads.

The CPU version of the code uses both MPI and OpenMP for parallelization (e.g. 4 MPI ranks and 12 OpenMP threads per node).

### Benchmark Variants

The JUQCS benchmark is to be executed in three variants.

1. **TCO Baseline**: This benchmark is the benchmark for the default evaluation. It simulates 36 qubits (`src/juqcs-light/src/input-long/h36.qcx`), so a total GPU memory of 1 TiB is needed. Note the usage of the `h36.qcx` file from the **`input-long`** directory.

2. **High-Scaling**: This benchmark simulates a larger amount of qubits, related to exploring scalability between a 50 PFLOP/s sub-partition of JUWELS Booster and a 1000 PFLOP/s Exascale system (with 20x the performance). As per general guidelines, two sub-variants exist: large memory and small memory. The large memory option simulates 42 qubits (64 TiB total memory), the small memory option simulates 41 qubits (32 TiB total memory). To submit a job to an Exascale system, use 46 or 45 qubits, depending on whether you choose the large or the small memory version.

3. **MSA**: This benchmark executes the CPU version of JUQCS on a CPU-equipped Cluster and a GPU version of JUQCS on a GPU-equipped Booster to collectively execute the algorithm. It simulates 34 qubits (256 GiB total memory) for this simplified case. Each MPI process uses either a single GPU or 12 CPUs on the corresponding nodes. The number of MPI processes (/nodes) for execution is a fixed value, see _Commitment_ for rules.

### Command Line

To execute a JUQCS benchmark from the command line, use the following template:

```
export I_MPI_HARD_FINALIZE=1 KMP_AFFINITY=compact
module load Stages/2023
module load NVHPC/23.1 ParaStationMPI/5.7.0-1
make -B -C src/juqcs-light/src qc22.gpu.exe  # build GPU executable
make -B -C src/juqcs-light/src qc22.cpu.exe  # build CPU executable (only for MSA; needs to be run on cluster)

srun ...  # see below; input file specifies the number of qubits: h36.qcx -> 36 qubits
```

Note that for all runs, both the total memory (= `16 * 2^numqubits`) and the total number of MPI processes always have to be a power of two. In the following, we give example command line strings to execute the respective variants, using certain numbers of nodes as examples.

1. **TCO Baseline**: This run uses 8 nodes with 4 GPUs per node (so 32 GPUs = 32 MPI ranks in total). Each GPU needs 32 GiB, so a total GPU memory of slightly more than 1 TiB is needed. Note that `input-long/` is used as the directory for the `h36.qcx` input file.

    ```
    srun -N 8 --ntasks-per-node 4 --cpus-per-task 12 --gres gpu:4 src/juqcs-light/src/qc22.gpu.exe src/juqcs-light/src/input-long/h36.qcx /*GPUS:4 /*SWAP:4 >h36.log
    ```
2. **High-Scaling**: The high-scaling runs use 512 nodes with 4 GPUs per node (so 2048 GPUs in total). The first run ("large") uses 32 GiB per GPU (so 64 TiB in total), the second uses 16 GiB per GPU (so 32 TiB in total).

    ```
    srun -N 512 --ntasks-per-node 4 --cpus-per-task 12 --gres gpu:4 --time 00:20:00 --partition largebooster src/juqcs-light/src/qc22.gpu.exe src/juqcs-light/src/input/h42.qcx /*GPUS:4 /*SWAP:4 >h42.log
    srun -N 512 --ntasks-per-node 4 --cpus-per-task 12 --gres gpu:4 --time 00:20:00 --partition largebooster src/juqcs-light/src/qc22.gpu.exe src/juqcs-light/src/input/h41.qcx /*GPUS:4 /*SWAP:4 >h41.log
    ```
3. **MSA**: This run uses 16 nodes; 8 GPU nodes with 4 GPUs per node (1 MPI process per GPU), and 8 CPU nodes with 4 MPI processes per node and 12 CPU tasks per process. The heterogeneous Slurm call on JUWELS Cluster is

    ```
    srun -N 8 --ntasks-per-node 4 --cpus-per-task 12 --partition booster --gres gpu:4 xenv -L NVHPC/23.1 -L ParaStationMPI/5.7.0-1 env OMP_NUM_THREADS=12 src/juqcs-light/src/qc22.gpu.exe src/juqcs-light/src/input/h34.qcx /GPUS:4 /*SWAP:4 : -N 8 --ntasks-per-node 4 --cpus-per-task 12 --partition batch xenv -L NVHPC/23.1 -L ParaStationMPI/5.7.0-1 env OMP_NUM_THREADS=12 src/juqcs-light/src/qc22.cpu.exe src/juqcs-light/src/input/h34.qcx /*SWAP:-1 >h34.log
    ```

`xenv` is a helper utility to load certain environment modules per sub-shell, here used in conjunction with the hetjob feature of Slurm.

### JUBE

Every variant of the benchmark can be executed with JUBE:

1. **TCO Baseline**

    ```
    jube run benchmark/jube/default.yaml
    ```
2. **High-Scaling**

    ```
    jube run benchmark/jube/high-scaling.yaml -t large  # run with 32 GiB per GPU
    jube run benchmark/jube/high-scaling.yaml           # run with 16 GiB per GPU
    ```
3. **MSA**

    ```
    jube run benchmark/jube/default-msa.yaml
    ```

The JUBE scripts are located at [`benchmark/jube/`](benchmark/jube/).

## Verification

Verifying the result of a JUQCS simulation is done by inspecting the `h*.log` files produced by JUQCS. In a successful run, the expectation values contained in this file have to be `<Qx(i)>=0.000E+00`, `<Qy(i)>=0.500E+00`, `<Qz(i)>=0.500E+00` for all qubits.

### JUBE

The verification process is done automatically by the JUBE script. The result of the verification process is shown under the `verified` column in the output of `jube result`. After a successful run and a subsequent analysis (see below), it can be viewed in `pretty` format using e.g.
```
jube result benchmark/jube/runs -s pretty
```

## Results

The metric of measurement for the Baseline and the MSA variant is the total simulation time, reported as `total_time[s]` by JUBE. If run from the command line, it can be found after `Elapsed time (rank = 0) = ` at the bottom of the `h*.log` file.

The metric of measurement for the High-Scaling variant is the total **normalized simulation time**, reported as `norm_total_time[s]` by JUBE. If run from the command line, it can be obtained by computing `total_time / norm`, where `total_time` is found after `Elapsed time (rank = 0) = ` at the bottom of the `h*.log` file, and `norm` is given by `(noperations - 1)/341` where `noperations` is found after `Total number of operations = ` in the top half of the `h*.log` file. The normalized simulation time can be divided into the normalized compute time (can be computed as `norm_total_time[s] - norm_mpi_time[s]`), the normalized time used for MPI communication over the network (reported as `norm_mpi_time[s]`). Ideally, the normalized compute time stays constant and the normalized MPI time increases with increasing number of qubits.

### JUBE

Using `jube result -s pretty` yields the following sample output for the three benchmark variants (the data is provided in the [`benchmark/data`](benchmark/data) directory):

1. **TCO Baseline**

    ```
    | qubits | largescale | verified | ngates | nodes | gpus | mem/gpu[GiB] | total_time[s] | init_time[s] | run_time[s] | mpi_time[s] | comp_time[s] |    systemname |   jobid |                           directory |
    |--------|------------|----------|--------|-------|------|--------------|---------------|--------------|-------------|-------------|--------------|---------------|---------|-------------------------------------|
    |     36 |         no |  correct |   7164 |     8 |   32 |         32.0 |       1083.79 |        10.35 |     1073.46 |      713.20 |       370.59 | juwelsbooster | 7536475 | runs/000060/000000_execute/work/h36 |
    ```
2. **High-Scaling**

    ```
    # large
    | qubits | large | verified | ngates | nodes | gpus | mem/gpu[GiB] | total_time[s] | init_time[s] | run_time[s] | mpi_time[s] | norm | norm_total_time[s] | norm_mpi_time[s] |    systemname |   jobid |                           directory |
    |--------|-------|----------|--------|-------|------|--------------|---------------|--------------|-------------|-------------|------|--------------------|------------------|---------------|---------|-------------------------------------|
    |     42 |   yes |  correct |    462 |   512 | 2048 |         32.0 |        201.03 |        27.46 |      174.47 |      159.60 | 1.35 |             148.38 |           117.80 | juwelsbooster | 7527323 | runs/000053/000000_execute/work/h42 |

    # small
    | qubits | large | verified | ngates | nodes | gpus | mem/gpu[GiB] | total_time[s] | init_time[s] | run_time[s] | mpi_time[s] | norm | norm_total_time[s] | norm_mpi_time[s] |    systemname |   jobid |                           directory |
    |--------|-------|----------|--------|-------|------|--------------|---------------|--------------|-------------|-------------|------|--------------------|------------------|---------------|---------|-------------------------------------|
    |     41 |    no |  correct |    451 |   512 | 2048 |         16.0 |        106.85 |        19.38 |       88.39 |       80.89 | 1.32 |              80.79 |            61.16 | juwelsbooster | 7527322 | runs/000052/000000_execute/work/h41 |
    ```
3. **MSA**

    ```
    | qubits | verified | ngates | nodes | gpus | mpi_size | mem/mpi[GiB] | total_time[s] | init_time[s] | run_time[s] | mpi_time[s] | comp_time[s] | systemname |   jobid |                           directory |
    |--------|----------|--------|-------|------|----------|--------------|---------------|--------------|-------------|-------------|--------------|------------|---------|-------------------------------------|
    |     34 |  correct |    374 |    16 |   32 |       64 |          4.0 |        158.33 |         2.82 |      155.60 |      120.10 |        38.23 |     juwels | 7527472 | runs/000055/000000_execute/work/h34 |
    ```

## References

<a name="1">[1]</a> D. Willsch, M. Willsch, F. Jin, K. Michielsen, and H. De Raedt, "GPU-accelerated simulations of quantum annealing and the quantum approximate optimization algorithm", [*Comput. Phys. Commun.* **278**, 108411](https://doi.org/10.1016/j.cpc.2022.108411) (2022)

## Commitment

### **TCO Baseline Variant**

The simulation time metric is `total_time[s]`.

The baseline simulation of `JUQCS` must be chosen such that the metric is below 1084 s. On the JUWELS Booster system at JSC, this value was achieved on 8 nodes, each running 4 MPI tasks (i.e., a total number of 32 GPUs).

### **High-Scaling Variant**

The simulation time metric is `norm_total_time[s]`. 

The "large" memory variant (tag "large") uses 
512 nodes with 4 tasks per node and 1 GPU per task (i.e. 2048 GPUs in total with 32 GiB per GPU = 42 qubits) and reaches a reference value of 148 s on JUWELS Booster. 
The "small" memory variant uses 
512 nodes with 4 tasks per node and 1 GPU per task (i.e. 2048 GPUs in total with 16 GiB per GPU = 41 qubits) and reaches a reference value of  81 s on JUWELS Booster. 

### **MSA Variant**:

The simulation time metric is always `total_time[s]`.
The variant is to be executed with 64 MPI processes, 32 processes using GPUs and 32 processes using CPUs. 
In the case of 4 GPUs per node, 8 GPU nodes are to be used with 1 MPI process per GPU; and 8 CPU nodes with 4 MPI processes per node. 
In the case of 2 GPUs per node, 16 GPU nodes are to be used (also 1 MPI process per GPU), and also 16 CPU nodes are to be used (with 2 MPI processes per node). 
In the case of another number of GPUs per node, 32 GPU MPI processes need to be distributed to the least possible amount of nodes (i.e. filling up the nodes), and the same number of nodes need to be used as well for the CPU branch of the execution. 
Always, 12 OpenMP threads should be launched per CPU MPI process.

The `total_time[s]` time metric should be smaller than a maximum of 1000 s.

On the modular JUWELS Cluster-Booster system at JSC, the reference value for 16 nodes (8 GPU nodes with 4 GPUs per node and 8 CPU nodes with 4 MPI Tasks per node, i.e 64 MPI ranks in total) is 158 s.
