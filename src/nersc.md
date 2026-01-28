# Submitting NERSC jobs

## NERSC Jobs

NERSC maintains a fairly detailed set of documentation pages on how to submit jobs, monitor jobs, cluster resources, etc. which can be found here: https://docs.nersc.gov/jobs/

However it is a lot of information and can get fairly technical, so this page is designed to distill down some of that information to get a quick start for submitting jobs on Perlmutter. This page is aimed at relatively small workflows and for developing code, it is not designed to run large scale production. Additionally here is a [link to some slides](https://www.nersc.gov/assets/Uploads/05-Running-Jobs-Feb2024.pdf) on job submission from the NERSC New Users training

NERSC uses Slurm for cluster/resource management and job scheduling. [Slurm](https://slurm.schedmd.com/documentation.html) is responsible for allocating resources to users, providing a framework for starting, executing and monitoring work on allocated resources and scheduling work for future execution [copied from NERSC docs]. So if you are familiar with Slurm, then a lot of this will feel familiar.

Jobs are submitted to the queue using a job script that defines all the resources needed and the commands to run. Additionally, a compute node can be requested and used interactively with a shell similar to the login nodes.

When running jobs on NERSC (interactively or submitting to the queue) an account to charge the computing hours needs to be specified. The account is different for CPU-only and GPU jobs, and for DUNE they are the following: CPU: `dune` and GPU: `dune_g`

Jobs are submitted to different queues depending on the queue constraints and the user's desired outcomes. Each queue corresponds to a "Quality of Service" (QOS): Each queue has a different service level in terms of priority, run and submit limits, walltime limits, node-count limits, and cost. At NERSC, the terms "queue" and "QOS" are often used interchangeably [copied from NERSC docs].

Furthermore, it is suggested to also take a look at the [NERSC best practices](https://docs.nersc.gov/jobs/best-practices/) page for how to effectively use NERSC and not slow down the system for everyone.

### Which Queue to Use

NERSC lists twelve different available queues in the documentation, but for nearly all expected ND development/work there are only a few that are relevant. These are the interactive and regular queues and their shared equivalents.

If you need an interactive terminal/shell, then the correct queue is to submit to `shared_interactive` or `interactive` (specifying in the constraint if the job is CPU-only or needs GPUs).

For jobs submitted to the batch system, the `shared` queue should be used rather than `regular` for most applications (this kind of contradicts the NERSC docs). If you think your job needs the resources offered by entire nodes please ask for advice from the production team.

Jobs are charged based on the amount of nodes requested regardless of if all the compuational resources are used, and typically most jobs do not need the resources of an entire node.

The other queues are for more specialized purposes or requirements, and are unlikely to be needed for typical ND uses. If your NERSC allocation is exhausted, **do not** use the `overrun` queue, contact Matt or Callum.

### Interactive Jobs

An interactive job gives you a shell on a compute node for the time requested and allows you to use the compute node as you would any login node. This is quite useful for debugging the environment and seeing if code will run or development when running small jobs (or infrequent jobs).

Request an interactive GPU session like so:
```bash
salloc --nodes 1 --qos shared_interactive --time XX:00:00 --constraint gpu --gpus 1 --account=dune_g
```
Be sure to specify the time, which is requested as HH:MM:SS. In this example it requests a node with a GPU and charges the DUNE allocation for the hours. Additionally see the [larnd-sim example](https://github.com/lbl-neutrino/larnd-sim-example) for running interactively.

There are a mix of nodes with 40 GB and 80 GB GPUs available on Perlmutter and you may find the 40 GB GPU is not enough when running larnd-sim. To specifically request an 80 GB GPU node use the following constraint: `-C 'gpu&hbm80g'` (or as a long option `--constraint 'gpu&hbm80g'`).

Request a shared interactive CPU-only session like so:
```bash
salloc --nodes 1 -n 1 -c 8 --qos shared_interactive --time XX:00:00 --constraint cpu --account=dune
```
which will request one node with one task (`-n`) with eight CPUs (`-c`). Note that the amount of RAM available scales with the number of CPUs requested.

See the [NERSC page on interactive jobs](https://docs.nersc.gov/jobs/interactive/) for more info.

### Job Scripts

The top-level [NERSC page on running jobs](https://docs.nersc.gov/jobs/) gives a good overview, but the following hopefully expands upon it and gives a more complete example. A job script starts out with a shell invocation (the [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))) and a number of Slurm directives `#SBATCH` that define the resources for the job. The NERSC docs list some of the most common options and the [Slurm documentation](https://slurm.schedmd.com/documentation.html) contains the full list, with a (maybe) [helpful summary page here](https://slurm.schedmd.com/pdfs/summary.pdf).

NERSC has a [page that can generate a basic job script](https://docs.nersc.gov/jobs/jobscript-generator/) for you that will simply submit another script (or executable) to the batch system. This script declares what resources the job needs and what commands or other scripts/executables to run. However the script generator assumes you need the entire node regardless of how many threads or GPUs you are actually using, and thus may need some minor editing to request accurate resources.

Generally NERSC recommends writing a separate script for all the actual commands, code, simulation, etc. that you want to run compared to the script that submits the job. This way both scripts can be modified separately and is more flexible.

One thing to note is that your script must be set to be executable (have executable permissions) for this to work, otherwise you will get a `Permission denied` error from Slurm.

#### Shared Queue
For tasks that need a small amount of resources, e.g. 32 or fewer threads or 1-2 GPUs, the shared queue should be used rather than requesting an entire 256 core machine to run a single-threaded program.

Job scripts are submitted to the queue using the `sbatch` command. Here is an example of a job that only needs a single CPU thread and no GPU that uses the shared QOS:
```bash
#!/bin/bash
#Submit to shared queue
#SBATCH --qos=shared

#Only need CPU resources (no GPU)
#SBATCH --constraint=cpu

#Name of the job
#SBATCH --job-name PHYSICS_TEST

#Charge to DUNE account
#SBATCH --account=dune

#Request a max time of 4 hours
#SBATCH --time=4:00:00

#Need only a single node
#SBATCH --nodes=1

#Only running a single task
#SBATCH --ntasks=1

#Request 4 GB of memory
#SBATCH --mem=4GB

#Store job output information in a file
#SBATCH --output=/path/to/physics_job.log

#Script or commands to run
srun /path/to/physics.sh --flag 999
```

Once the job script is ready, it can be sent to the queue using the `sbatch` command. It can be used with no command line options/flags which uses the directives in the script or default values.
```
sbatch physics_job_script.sh
```

You can also supply flags to the `sbatch` command to define/request resources. Values specified on the command line will take precedence or overwrite values specified in the script.
```
sbatch --time=6:00:00 physics_job_script.sh
```

The same shared queue can be used to request GPU nodes, and should be used when only 1 or 2 GPUs are needed (e.g. a single larnd-sim run). The changes would be to set the constraint to GPU, define the number of GPUs needed, and set the account for the GPU allocation. For example:
```bash
#Need a GPU node
#SBATCH --constraint=gpu

#Request a single GPU
#SBATCH --gpus-per-task=1

#Change account to charge for GPUs
#SBATCH --account=dune_g
```

#### Regular Queue
The shared queue will work for most development tasks, but sometimes more power is needed (e.g. production) and the regular queue will allow for entire nodes to be exclusively requested. The following is an example pulled from the NERSC job script generator (which is annotated):
```bash
#!/bin/bash

#Ask for one node
#SBATCH -N 1

#Need only cpu resources (no gpu)
#SBATCH -C cpu

#Submit to regular queue
#SBATCH -q regular

#Name of the job
#SBATCH -J PHYSICS_TEST

#Charge to DUNE account
#SBATCH -A dune

#Ask for 2 hours of walltime
#SBATCH -t 2:00:0

# Only needed if code uses OpenMP
# but also harmless to export
# OpenMP settings:
export OMP_NUM_THREADS=1
export OMP_PLACES=threads
export OMP_PROC_BIND=spread

#run a multi-threaded application in a single process using all 256 cores:
srun -n 1 -c 256 --cpu_bind=cores /path/to/physics.sh --flags 123

# OR:

# run a single-threaded application in 256 parallel processes (tasks):
srun -n 256 -c 1 /path/to/physics_single_threaded.sh --flags 123
```
This script will submit a job for one node (with all its resources) that runs the `physics.sh` script with some flags to the regular queue and using the DUNE allocation.

If you want to have each task process a separate input file, you can use (for a single-node job) the `SLURM_LOCALID` environment variable, which runs from 0 to `tasks_per_node - 1`. This can be used as an index into an array of input paths. A generalization for multi-node jobs is `$((SLURM_NODEID*SLURM_NTASKS_PER_NODE + SLURM_LOCALID))`.

Another example is below using the long form of the options but is otherwise very similar. See the top-level [NERSC page on running jobs](https://docs.nersc.gov/jobs/) for the list of options and their descriptions.
```bash
#!/bin/bash

#Need to request gpu resources
#SBATCH --constraint=gpu

#Submit to regular queue (also can try debug queue)
#SBATCH --qos=regular

#Name of the job
#SBATCH --job-name=ABC_TEST

#Charge to DUNE account
#SBATCH --account=dune

#Ask for 2 hours of walltime
#SBATCH --time=2:00:0

#Ask for one node (commonly passed to srun directly)
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=256
#SBATCH --gpus=4
#Not sure why the NERSC generator asks for all 256 even if
#only one thread is specified. Might be able to specify fewer.

#The above options are passed to the srun command

# Only needed if code uses OpenMP
# but also harmless to export
# OpenMP settings:
export OMP_NUM_THREADS=1
export OMP_PLACES=threads
export OMP_PROC_BIND=spread

#run the application
#the -o flag specifies a file to store the job output/log from SLURM
srun -o /path/to/job_log_output1.txt /path/to/physics.sh --flags 123

#can also run another job with different inputs
#these jobs will run in serial (make sure you ask for enough time)
srun -o /path/to/job_log_output2.txt /path/to/physics.sh --flags 456
```

#### Debug Queue

For developing and testing your code and checking if the job scripts will run on the compute nodes, there is a specific queue (QOS) called `debug` that has a max runtime of 30 minutes. This can be useful if the regular queue has a long wait time, and to specify the debug QOS: `--qos=debug`.

The `debug` queue is for testing jobs that will eventually run using the `regular` queue. If your job is going to run using the `shared` queue, use that as well for testing if the job will run (maybe with a small(er) walltime to reduce the wait time).

### File transfer / storage / scratch

Each user on Perlmutter has a large, temporary storage space called "scratch" that is optimized for file transfer and I/O. Your scratch space can be accessed with the environment variable `$SCRATCH` (or `$PSCRATCH`). Files in scratch not accessed for 8 weeks will be automatically purged, with a record left behind of the purged filenames (the file itself is lost); you can see these records using `ls $SCRATCH/.purged*`.

As a general best practice, users should do work from `$SCRATCH` instead of `$HOME`. `$HOME` is meant for permanent and relatively small storage, and it is not tuned to perform well for parallel jobs. `$HOME` is perfect for storing files such as source codes and shell scripts, etc. Additionally as part of the DUNE project there is a DUNE project space available to store files, code, etc. You can create a directory at `/global/cfs/projectdirs/dune/users/`.

`$SCRATCH` is meant for large and temporary storage. It is optimized for read and write operations, and is accessible from all Perlmutter compute nodes. `$SCRATCH` is perfect for staging data and performing parallel computations.

### Job monitoring (and cancelling)

The [NERSC page on monitoring](https://docs.nersc.gov/jobs/monitoring/) is quite clear with all the options to monitor your jobs, and even how to login to the nodes where your jobs are running if needed. A few of the most common bits from the page are copied/paraphrased here.

Since NERSC uses SLURM, `squeue` is the base command for job monitoring and can be used as-is if you are already familiar. NERSC also has a wrapper around `squeue` called `sqs` that by default shows the status (and some additional info) about your jobs, and it accepts all the same flags as `squeue`. As a quick example, to see your own jobs in the queue you can run:
```
squeue --me
```
Currently queued and recently finished jobs can be queried by `squeue`, but for past history and other accounting data, `sacct` must be used. Avoid using either command in loops or high-volume automated tasks as large amounts of requests to the SLRUM controller or database can cause performance issues.

As an optional directive when submitting jobs, an email alert can be setup to be sent when the job starts, ends, or fails. The `#SBATCH` options are the following:
```
#SBATCH --mail-type=begin,end,fail
#SBATCH --mail-user=user@domain.com
```

Jobs can be cancelled using the `scancel` command which kills job(s) based on the $JobID. The $JobID can be found using `sqs` or `squeue` (and some other methods) and then used to kill a job (or multiple) like so: `scancel $JobID1 $JobID2`

### Containers

Jobs can be run in containers to provide an environment or other necessary programs. The container needs to be part of the NERSC container registry, and NERSC only supports Shifter and Podman-HPC as container technologies (note that Singularity/Apptainer is not supported). To use a container you need to request it as part of the `SBATCH` options in the job submission script and then add the container runtime to the commands in the script.

```bash
#!/bin/bash
#Same options as before
#SBATCH --qos=shared
#SBATCH --constraint=cpu
#SBATCH --job-name PHYSICS_TEST
#SBATCH --account=dune
#SBATCH --time=4:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --mem=4GB
#SBATCH --output=/path/to/physics_job.log

#Request container image from the NERSC registry
#SBATCH --image=registry/image:tag

#Script or commands to run
#Note the addition of shifter as part of the command preceding the python invocation
srun shifter python script.py
```
Containers can be used in interaction jobs by adding the `--image` flag as part of the `salloc` request. For example:
```
salloc -N 1 -t 60 -C cpu -q interactive --image=registry/image:tag
```

Juypter Notebooks at NERSC can be run via containers but it requires a couple steps to configure properly, and these instructions are copied [from NERSC documentation](https://docs.nersc.gov/services/jupyter/how-to-guides/). First the Jupyter kernel needs to be installed with the desired Shifter image like so:

```
shifter --image=<imgname> python3 -m ipykernel install --prefix $HOME/.local --name env --display-name MyEnvironment
```
where the image name, environment name, and display name need to be specified. Next, the kernel specification needs to be told to run Shifter when starting so it loads the container environment. Edit the corresponding kernel spec JSON file (should be located in `.local/share/jupyter/kernels/<env_name>`) as `kernel.json` by modifying `argv` to prepend the following lines:
```bash
"shifter",
"--image=<imgname>",
```
The kernel spec should look something like this:
```json
{
  "argv": [
    "shifter",
    "--image=myimage:v1.2.3",
    "</path/to/your/image/python>",
    "-m",
    "ipykernel_launcher",
    "-f",
    "{connection_file}"
  ],
  "display_name": "MyEnvironment",
  "language": "python",
  "metadata": {
    "debugger": true
  }
}
```
