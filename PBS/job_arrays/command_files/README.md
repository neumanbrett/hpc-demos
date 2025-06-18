# Using Job Arrays to Launch a Command File

Multiple Program, Multiple Data (MPMD) jobs run multiple independent, typically serial executables simultaneously. Such jobs can easily be dispatched with PBS job arrays, even on machines like Derecho where compute nodes are exclusively and entirely assigned to a users' job.

The `launch_cf` tool reads in a command file that contains applications to run within the job array.  Each line of the command file will be assigned to an index of the job array when using default arguments.  Multiple CPUs can be assigned per line using the `nthreads|threads-per-step` argument and will assign an index to each chunk of CPUs used for each step.

# Command File

Consider a command file as follows.

`cmdfile`:
   ```   
# this is a comment line for demonstration
./cmd1.exe < input1 # comments are allowed for steps too, but not required
./cmd2.exe < input2
./cmd3.exe < input3
...
./cmdN.exe < inputN
   ```

The `cmdfile` should be in your working directory, and all commands should be either in your path or referenced from the working directory.

Then,

`launch_cf <PBS Submission Args>`

For example,

`launch_cf -A <account> -l walltime=1:00:00`

## Examples:

### Launches the commands listed in ./cmdfile:
```
launch_cf -A PBS_ACCOUNT -l walltime=1:00:00
```

### Launches the OpenMP-threaded commands listed in ./omp_cmdfile:
```#  (requires ppn=128 = (32 steps/node) * (4 threads/step)
launch_cf -A PBS_ACCOUNT -l walltime=1:00:00 --nthreads 4 --steps-per-node 32 ./omp_cmdfile
```

The command will assume reasonable defaults on Derecho and Casper for the number of job "steps" from the cmdfile to run per node, memory per node, PBS queues, etc... Each of these parameters can be controlled via launch_cf optional command line arguments described in the [Optional Parameters](#optional-parameters) section.

# Environment Configuration

The jobs listed in the cmdfile will be launched from a bash shell in the users default environment. The optional file `config_env.sh` will be "sourced" from the run directory in case environment modification is required, for example to load a particular module environment, to set file paths, etc.


# Structure for a launch_cf workflow

The command file, executables, and input files should all reside in the directory from which the job is submitted. If they don't, you need to specify adequate relative or full paths in both your command file and job scripts. The sub-jobs will produce output files that reside in the directory in which the job was submitted. The command file can then be executed with the launch_cf command.  An example structure would look like:

```
> Run_Directory/
   - cmdfile
   - config_env.sh (optional)
   - application.exe
   - data.nc
``` 


# Optional Parameters

The output of the `launch_cf` help:
```
launch_cf -h
     <-q|--queue PBS_QUEUE>
     <--ppn|--processors-per-node #CPUS>
     <--steps-per-node #Steps/node>
     <--nthreads|--threads-per-step #Threads/step>
     <--mem|--memory RAM/node>
     <--cpu_type TYPE>
     <--random_start_delay MAX_DELAY (sec)>
     -A PBS_ACCOUNT -l walltime=01:00:00
     ... other PBS args ...
     <command file>
```

Where arguments within brackets `< >` are optional.  The default values of optional parameters assign:
- A single CPU per line in the command file
- A job array with the total number of indices equal to the number of lines in the command file
- A PBS select statement that calculates the number of nodes required based on lines in the command file divided by the processors per node

## Queue

`-q` or `--queue`: Direct translation from Casper or Derecho queues.  Queue types:

	Derecho:
		- main
		- develop

	Casper:
		- casper

	Cross-submission:
		- casper@casper-pbs
		- main@desched1
		- develop@desched1


## Processors per node

`--ppn` or `--processors`

Equivalent to submitting a PBS resource select statement using `ncpus=X`.

This can be thought of as the number of CPUs available per node that can be used to distribute lines from the command file.

| System  | Default | Max | Min |
| ------- | ------- | --- | --- |
| Derecho | 128     | 128 | 1   |
| Casper  | 36      | 62* | 1   |

**The most available and prioritized Casper HTC Nodes have 62 available processors.    Since Casper has many node types with different CPU amounts, be careful with allocating too many CPUs which will result in jobs never starting.*

This value works with `--steps-per-node` and `--nthreads` to determine the total node and CPU requirements for your job array.


## Threads per step

`--nthreads` or `--threads-per-step`

The number of threads per line in your command file.  Since each line of the command file is an executable, this is equivalent to assigning the number of threads or CPUs per executable.

| System  | Default | Max    | Min |
| ------- | ------- | ------ | --- |
| Derecho | 1       | ${ppn} | 1   |
| Casper  | 1       | ${ppn} | 1   |


## Steps per node

`--steps-per-node`

Assigns the number of lines in your command file that will run on a single node.  This is most often left as the default to allow the tool to assign the steps per node based on the **processors per node (ppn)** divided by the **threads per step (nthreads)**.

If manually entering values for steps per node, careful consideration of the interaction between `--steps-per-node` and `--nthreads` is required to avoid over allocating resources per node.  For example, the Derecho settings of `--ppn`=128, `--steps-per-node`=64, and `--nthreads`=4 will fail since it will attempt to fill 256 (64\*4) onto a single node which exceeds the parts-per-node amount.

| System  | Default                        | Max      | Min |
| ------- | ------------------------------ | -------- | --- |
| Derecho | `${ppn} / ${threads_per_step}` | `${ppn}` | 1   |
| Casper  | `${ppn} / ${threads_per_step}`  | `${ppn}` | 1   |

## Memory

`--mem`

Assigns the amount of memory available per node.

| System  | Default (GB) | Max (GB) | Min (GB) |
| ------- | ------------ | -------- | -------- |
| Derecho | 235          | 235      | 4        |
| Casper  | 10           | 733*     | 4        |


**The HTC nodes have a maximum of 733GB of memory per node.  Large memory nodes are available on Casper if needed.*

## CPU Type 

Used on Casper to request specific resources.  Check the [Casper Node Types](https://ncar-hpc-docs.readthedocs.io/en/latest/compute-systems/casper/starting-casper-jobs/casper-node-types/) page for available resources and how to request them.


# Querying a Job Array

To view the progress and details of each index in the job array while running, add the `-t` flag to `qstat` and include the `[]` brackets to indicate that the job is a job array:

```
qstat -t <jobid>[]
```

