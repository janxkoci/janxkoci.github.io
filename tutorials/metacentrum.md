---
layout: post
title: MetaCentrum
---

{% include toc.html %}

> MetaCentrum is an organisation managing a national grid of clusters for scientific computing. All academics and students at Czech institutions can use its resources for free.

## Useful links
To get you started, here are links to a few useful resources:

- [Get account](https://metavo.metacentrum.cz/en/application/index.html) (uses [eduID](https://www.eduid.cz/en/index) accounts)
- [Wiki with help guides](https://wiki.metacentrum.cz/wiki/Categorized_list_of_topics)
- [Beginner's guide](https://wiki.metacentrum.cz/wiki/Beginners_guide)
- [Hardware overview](https://metavo.metacentrum.cz/pbsmon2/hardware) (more [details here](https://metavo.metacentrum.cz/pbsmon2/props))
- [Long-term storage of data](https://wiki.metacentrum.cz/wiki/Working_with_data#Data_archiving_and_backup)

No need to read them right away, feel free to come back to them later 😉

## First steps
After you register an account using your university credentials, you can connect to the grid using `ssh` connection to several login nodes. You can also use the `scp` command or any SFTP client to copy your files over to the server. The `rsync` tool is also available.

> 💡 **Tip:** many modern text editors have plugins for SFTP protocol, allowing you to directly connect them to clusters and edit scripts and files remotely. Just watch out for the size of files you are editing. 😉

Authentication can be done using passwords or `ssh` keys - all the cool methods you know and love!

To work at the Linux system, you can check out [my previous guide](https://janxkoci.github.io/tutorials/linux_for_biologists.html) or the [Beginner's guide](https://wiki.metacentrum.cz/wiki/Beginners_guide) of MetaCentrum.

## Grid peculiarities
The grid infrastructure has a few peculiar traits that you may need to deal with. Below I'll provide quick solutions I used to deal with common problems.

For start, **the grid is not a homogeneous cluster**, but rather a network of clusters of various sizes, located in different cities all over the country. Different parts have different hardware specifications, and in some cases even different Linux OS versions (distributions).

For example, after you submit a job, the job may be running at a computer that does not see your data - so you have to specify where the data is (i.e. which city and cluster). Luckily, this is quite easy to do, so read on and don't panic! 😎

## Job scheduling with PBS Pro / OpenPBS
MetaCentrum uses PBS Pro / OpenPBS for scheduling jobs.

> ℹ️ **Note:** In 2020, PBS Pro was renamed to OpenPBS and [released](https://github.com/openpbs/openpbs) under an open-source license. However, most resources on the internet refer to the old name, including the documentation at MetaCentrum. The name PBS Pro is also used for commercial version (same code, but with paid support).

The syntax of PBS Pro differs from Torque PBS, which is another implementation used e.g. at our faculty cluster. You can have a look at the [quick cheat sheet](https://wiki.metacentrum.cz/w/images/9/9f/Quickstart-pbspro-small.pdf) or the [full docs](https://wiki.metacentrum.cz/wiki/About_scheduling_system) provided by MetaCentrum.

The basic command has the following structure:

```bash
qsub -A Project_ID -q queue -l select=x:ncpus=y:mem=z,walltime=[[hh:]mm:]ss[.ms] jobscript
```

In practice, you can omit a few parameters and mostly focus on specifying resources needed for your job (using the `-l` argument).

### Kerberos tickets
To be able to submit jobs, you also need something called a **kerberos ticket**. You will automatically obtain the Kerberos tickets while login with password. **The ticket is valid for 10 hours**. This limitation is especially important if you are using a [terminal multiplexer](https://janxkoci.github.io/tutorials/linux_for_biologists.html#terminal-multiplexers), such as GNU `screen` or `tmux`.

In case you need to obtain a new kerberos ticket (and cannot just logout and login back, such as when inside a `screen` or `tmux` session), you can issue the following command:

	kinit

The command will prompt you for a password to your account (by default).

### Queues
When submitting jobs, you should keep in mind a few details about the queues:

- Unless specified, the **default queue is only 1 hour**! You should explicitly specify longer queue or set a wall time for your job, if required.
- Queue and required job wall time affect **priority of the job** - longer jobs have lower priority. Very long jobs can wait a long time to even start!
- If not explicitly selected, the queue will be **automatically assigned based on wall time** set in the job script or as argument to the `qsub` command.

### Interactive jobs
The PBS scheduling system supports interactive jobs. These are useful to test software or (un)compress data, without clogging the login nodes.

You can start an interactive job by using the `qsub` command with the `-I` argument, e.g.:

```bash
qsub -N ijob -I -l select=1:ncpus=16:mem=4gb,walltime=24:00:00
```

## Storage
There are [three types](https://wiki.metacentrum.cz/wiki/Working_with_data#Data_storage) of storage at MetaCentrum:

- **Scratch storages** - fast, small capacity, use for computations (esp. I/O intensive)
- **Disk arrays** - regular drives to keep data between computations (includes your `/home` directory)
- **Hierarchical storages** - massive capacity for long-term storage of data

### Working directory
Not every computer in the grid will automatically share your `/home` directory, so its content will depend on where you login or where your job is being executed. For example, if you typically login to a cluster `skirit.ics.muni.cz` in Brno and copy all your files there, it does **not** mean that a job scheduled to run say in Prague will see the same files in the `/home` directory.

In practice, `/home` directories are specific to login nodes of clusters at each institution in each city (called [frontends](https://wiki.metacentrum.cz/wiki/Frontend)). They are also connected by network to other clusters in other cities, so the data are accessible within the entire grid. However, this means that your `/home` directory can actually be something like `/storage/brno2/home`.

For this reason, MetaCentrum provides a few **environment variables** to make accessing your files easier.

The most important is probably `PBS_O_WORKDIR` - it holds the full path to a directory from where you submit a job. This means that including the following line near the top of your script will make sure that your job has access to all the files it needs:

```bash
cd $PBS_O_WORKDIR || exit
```

On the other hand, the variable `HOME` corresponds to the `/home` directory at the particular node where your job started executing. This is arguably less useful, but worth knowing about.

If your job requires fast ([scratch](https://wiki.metacentrum.cz/wiki/Beginners_guide#Specify_scratch_directory)) disk space, you may need another useful variable - `SCRATCHDIR`.

## Software
There are several ways to get the software you need. First, MetaCentrum provides so-called **software modules**. These are loadable packages of software and libraries that you can load and use in your scripts. Many of the packages are provided in several versions to choose from, to cater to all your software needs.

But packages in **modules can be old**. Or the software you need is **not available**. In that case, you are free to install the software yourself. I highly recommend using a **package manager**, such as `conda` or `brew`, as I've [described before](https://janxkoci.github.io/tutorials/linux_for_biologists.html#scientific-package-managers).

> **Tip:** The [conda manager is already available](https://wiki.metacentrum.cz/wiki/Conda_-_modules) at MetaCentrum, as a module. It includes several preinstalled environments for specific tasks. If you choose to use it, it's probably a good idea to install your programs in an [isolated environment](https://gist.github.com/janxkoci/a3e446ee0f209e9593a2f2f87fca0058#environments) to not interfere with the rest of the conda-managed packages. On the other hand - nothing is stopping you from installing your own miniconda or anaconda in your home directory! 😉️

### Modules
MetaCentrum provides quite a lot of software modules for different areas of research. You can see a [list sorted by topics here](https://wiki.metacentrum.cz/wiki/MetaCentrum_Application_List).

A few quick commands to work with modules:

```bash
# check availability
module avail # see all available modules
module avail samtools # see all modules matching samtools
module avail r/ # see all R modules (without matching)
# load module
module load samtools # load default version
module load samtools-0.1.19 # load specific version
```

### Graphical programs
It is even possible to run [graphical programs](https://wiki.metacentrum.cz/wiki/Kategorie:Applications_with_GUI), such as RStudio, Tablet, or Geneious (MetaCentrum even provides a license, try `qsub -l geneious=1`). There are two main ways to run GUI programs:

- [X forwarding over ssh](https://wiki.metacentrum.cz/wiki/X-Window)
- [Remote desktop](https://wiki.metacentrum.cz/wiki/Remote_desktop)

#### X forwarding over ssh
To use graphical programs over `ssh`, you should include either the argument:

- `-X` (unencrypted X forwarding),
- `-Y` (encrypted X forwarding), or simply
- `-XY` (encrypted X forwarding, if available, else unencrypted fallback).

So to connect use e.g. `ssh -XY name@skirit.metacentrum.cz`.

#### Remote desktop
You can also use either a web browser or a VNC client to use GUI programs. See the [instructions from MetaCentrum](https://wiki.metacentrum.cz/wiki/Remote_desktop) for more details.

## Examples
### Example 1 - job script
Here is an example script that uses `pigz` (the parallel implementation of gzip) to uncompress data in current directory. The `pigz` package is already preinstalled at MetaCentrum (and available without loading any module, at least on the Debian systems), but we will use a version installed by `conda`, just so I can illustrate its usage.

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
Here is an example **series of commands** to launch an interactive job and the R computing environment in the current working directory:

```bash
## SUBMIT interactive job
qsub -N Rjob -I -l select=1:ncpus=16:mem=16gb,walltime=24:00:00

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

## GO TO WORKDIR
cd $PBS_O_WORKDIR || exit

## LOAD Tablet module
module add tablet-1.14

## START Tablet
tablet
```
