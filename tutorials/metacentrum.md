---
layout: post
title: MetaCentrum
---

> **MetaCentrum is an organisation managing a national grid of clusters for scientific computing. All academics and students at Czech institutions can use its resources for free.**

{% include toc.html %}

## Useful links
To get you started, here are links to a few useful resources:

- [Get account](https://metavo.metacentrum.cz/en/application/index.html) (uses [eduID](https://www.eduid.cz/en/index) accounts)
- [MetaCentrum Documentation](https://docs.metacentrum.cz/en/docs/welcome)
- [Beginner's guide](https://docs.metacentrum.cz/en/docs/computing/run-basic-job)
- [Software modules](https://docs.metacentrum.cz/en/docs/software/alphabet)
- [Hardware overview](https://metavo.metacentrum.cz/pbsmon2/hardware) (more [details here](https://metavo.metacentrum.cz/pbsmon2/props))
- [Long-term storage of data](https://docs.metacentrum.cz/en/docs/data/types-of-storage)

No need to read them right away, feel free to come back to them later 😉

## First steps
After you register an account using your university credentials, you can connect to the grid using `ssh` connection to several login nodes. You can also use the `scp` command or any SFTP client to copy your files over to the server. The `rsync` tool is also available.

> 💡 **Tip:** many modern text editors have plugins for SFTP protocol, allowing you to directly connect them to clusters and edit scripts and files remotely. Just watch out for the size of files you are editing. 😉

Authentication can be done using passwords or `ssh` keys - all the cool methods you know and love!

To work at the Linux system, you can check out [my previous guide](https://janxkoci.github.io/tutorials/linux_for_biologists.html) or the [introductory guide](https://docs.metacentrum.cz/en/docs/computing/run-basic-job) of MetaCentrum.

## Grid peculiarities
The grid infrastructure has a few peculiar traits that you may need to deal with. Below I'll provide quick solutions I used to deal with common problems.

For start, **the grid is not a homogeneous cluster**, but rather a network of clusters of various sizes, located in different cities all over the country. Different parts have different hardware specifications, and in some cases even different Linux OS versions (distributions).

For example, after you submit a job, the job may be running at a computer that does not see your data - so you have to specify where the data is (i.e. which city and cluster). Luckily, this is quite easy to do, so don't panic and read on! 😎

## Job scheduling with OpenPBS (ex PBS Pro)
MetaCentrum uses OpenPBS for scheduling jobs.

> ℹ️ **Note:** In 2020, PBS Pro was renamed to OpenPBS and [released](https://github.com/openpbs/openpbs) under an open-source license. However, many resources on the internet refer to the old name, including the documentation at MetaCentrum. The name PBS Pro is also used for commercial version (same code, but with paid support).

The syntax of OpenPBS differs from Torque PBS, which is another implementation used e.g. at our faculty cluster. You can have a look at the [quick overview](https://docs.metacentrum.cz/en/docs/computing/resources/pbs-commands) or the [detailed docs](https://docs.metacentrum.cz/en/docs/computing/advanced) provided by MetaCentrum.

The basic command has the following structure:

```bash
qsub -A Project_ID -q queue -l select=x:ncpus=y:mem=z,walltime=[[hh:]mm:]ss[.ms] jobscript
```

In practice, you can omit a few parameters and mostly focus on specifying resources needed for your job (using the `-l` argument). You can also check the online [qsub assembler](https://docs.metacentrum.cz/en/docs/computing/resources/qsub-compiler).

You can check your jobs in the queue with `qstat`:

```bash
qstat # show all jobs (will be very long)
qstat -u my_name # show jobs for given user only
qstat -xu my_name # show also recently finished jobs
```

### Kerberos tickets
To submit jobs, you also need something called a [**kerberos ticket**](https://docs.metacentrum.cz/en/docs/computing/advanced#kerberos-authentication). You will automatically get a fresh Kerberos ticket after login with a password. **The ticket is valid for 10 hours**. This limitation is especially important if you are using a [terminal multiplexer](https://janxkoci.github.io/tutorials/linux_for_biologists.html#terminal-multiplexers), such as GNU `screen` or `tmux`.

In case you need a new kerberos ticket (and cannot just logout and login back, such as when inside a `screen` or `tmux` session), you can use the following command:

	kinit

The command will ask you for a password to your account. After authentication, a new ticket will be issued for your account and you will be able to submit jobs again.

### Queues
When submitting jobs, you should keep in mind a few details about the [queues](https://docs.metacentrum.cz/en/docs/computing/resources/queues):

- Specify the resources you need - **the defaults are 1 core, 400 MB of RAM, and 24 hours of walltime**. You should explicitly specify longer queue or set a wall time for your job, if required.
- **Shorter jobs have higher priority**. Very long jobs can wait a long time to even start! You can also **increase your priority** by [acknowledging MetaCentrum in your publications](https://docs.metacentrum.cz/en/docs/computing/resources/fairshare).
- If not explicitly selected, the queue will be **automatically assigned based on wall time** set in the job script or as argument to the `qsub` command.

### Interactive jobs
The PBS scheduling system supports [interactive jobs](https://docs.metacentrum.cz/en/docs/computing/advanced#interactive-jobs). These are useful to test software or (un)compress data, without clogging the login nodes.

You can start an interactive job by using the `qsub` command with the `-I` argument, e.g.:

```bash
# select 1 node, 16 cores, 4 GB of memory, and 24 h walltime
qsub -I -N ijob -l select=1:ncpus=16:mem=4gb,walltime=24:00:00
```

## Storage
There are [three types](https://docs.metacentrum.cz/en/docs/data/types-of-storage) of storage at MetaCentrum:

- **Scratch storages** - fast, small capacity, use for computations (esp. I/O intensive)
- **Disk arrays** - regular drives to keep data between computations (includes your `/home` directory)
- **Hierarchical storages** - massive capacity for [long-term storage of data](https://docs.metacentrum.cz/en/docs/data/types-of-storage)

### Working directory
Not every computer in the grid will automatically share your `/home` directory, so its content will depend on where you login or where your job is being executed. For example, if you typically login to a cluster `skirit.ics.muni.cz` in Brno and copy all your files there, it does **not** mean that a job scheduled to run say in Prague will see the same files in the `/home` directory.

In practice, `/home` directories are specific to login nodes of clusters at each institution in each city (called [frontends](https://docs.metacentrum.cz/en/docs/computing/infrastructure/frontends)). They are also connected by network to other clusters in other cities, so the data are accessible within the entire grid. However, this means that your `/home` directory can actually be something like `/storage/brno2/home`.

For this reason, MetaCentrum provides a few **environment variables** to make accessing your files easier.

The most important is probably `PBS_O_WORKDIR` - it holds the full path to a directory from where you submit a job. This means that including the following line near the top of your script will make sure that your job has access to all the files it needs:

```bash
cd $PBS_O_WORKDIR || exit
```

On the other hand, the variable `HOME` corresponds to the `/home` directory at the particular node where your job started executing. This is arguably less useful, but worth knowing about.

If your job requires fast ([scratch](https://docs.metacentrum.cz/en/docs/computing/infrastructure/scratch-storages)) disk space, you may need another useful variable - `SCRATCHDIR`.

## Software
There are several ways to get the software you need. First, MetaCentrum provides so-called **software modules**. These are loadable packages of software and libraries that you can load and use in your scripts. Many of the packages are provided in several versions to choose from, to cater to all your software needs.

But packages in **modules can be old**. Or the software you need is **not available**. In that case, you are [free to install the software yourself](https://docs.metacentrum.cz/en/docs/software/install-software). I highly recommend using a **package manager**, such as `conda` or `brew`, as I've [described before](https://janxkoci.github.io/tutorials/linux_for_biologists.html#scientific-package-managers).

> 💡 **Tip:** The `conda` manager is already [available at MetaCentrum](https://docs.metacentrum.cz/en/docs/software/sw-list/conda-modules), as a module. It includes several preinstalled environments for specific tasks. If you choose to use it, it's probably a good idea to install your programs in an [isolated environment](https://janxkoci.github.io/tutorials/conda_cheatsheet.html#environments), to not interfere with the rest of the conda-managed packages. On the other hand - nothing is stopping you from installing your own miniconda or anaconda in your home directory! 😉️

### Modules
MetaCentrum provides quite a lot of software modules for different areas of research. You can see a [list here](https://docs.metacentrum.cz/en/docs/software/alphabet).

A few quick commands to work with modules:

```bash
# check availability
module avail # see all available modules
module avail samtools # see all modules matching samtools
module avail r/ # see all R modules (without matching)
# load module
module load samtools # load default version
module load samtools-0.1.19 # load specific version
module add samtools-0.1.19 # same as above
# list loaded modules
module list # see list of loaded modules
```

You can find more details in the [MetaCentrum docs](https://docs.metacentrum.cz/en/docs/software/modules).

### Graphical programs
It is even possible to run [graphical programs](https://docs.metacentrum.cz/en/docs/software/graphical-access), such as RStudio, Tablet, or Geneious (MetaCentrum even provides a license, try `qsub -l geneious=1`).

## Examples
### Example 1 - job script
Here is an example script that uses `pigz` (the parallel implementation of `gzip`) to uncompress data in current directory. The `pigz` package is already preinstalled at MetaCentrum (and available without loading any module, at least on the Debian systems), but we will use a version installed by `conda`, just so I can illustrate its usage.

```bash
#PBS -N pigz
#PBS -l select=1:ncpus=60:mem=4gb,walltime=24:00:00

## GO TO WORKDIR
cd $PBS_O_WORKDIR || exit

## EXPORT CONDA PATH
export PATH="/storage/brno2/home/jena/miniconda3/bin:$PATH"

## DECOMPRESS DATA
pigz -d data/*.fq.gz
```

### Example 2 - interactive R session
Here is an example **series of commands** to launch an interactive job and the `R` computing environment in the current working directory:

```bash
## SUBMIT interactive job
qsub -I -N Rjob -l select=1:ncpus=16:mem=16gb,walltime=24:00:00
```

After the interactive session starts, you can start working:

```bash
## GO TO WORKDIR
cd $PBS_O_WORKDIR || exit

## LOAD R module
module add r/r-4.1.1-intel-19.0.4-xrup2b3

## START R CLI session
R
```

### Example 3 - GUI app & interactive session
Here is an example **series of commands** to launch an interactive job and the Tablet graphical software in the current working directory (don't forget to use `ssh -XY`):

```bash
## SUBMIT interactive job
qsub -N tablet -I -l select=1:ncpus=16:mem=16gb,walltime=24:00:00
```

After the interactive session starts, you can start working:

```bash
## GO TO WORKDIR
cd $PBS_O_WORKDIR || exit

## LOAD Tablet module
module add tablet-1.14

## START Tablet
tablet
```

## Comments
