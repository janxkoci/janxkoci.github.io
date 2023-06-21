---
layout: post
title: Conda cheat sheet
---

Simple cheatsheet for `conda` - a package manager designed for scientists and HPCs. **No sudo/root permissions needed**.

{% include toc.html %}

## Get conda (and/or mamba)
The `conda` manager is available for all major platforms in two main versions:

- [Anaconda](https://www.anaconda.com/products/individual#Downloads) - best for Python-heavy workflows, includes 200+ packages for scientific computing, currated by Continuum Inc. (`conda` dev company).
- [Miniconda](https://docs.conda.io/en/latest/miniconda.html) - ready for any workflow - only the `conda` manager, its dependencies, and channels at your command. It also allows installing any or all packages from the Anaconda distribution, and thousands more.

Another option is [Mamba](https://github.com/mamba-org/mamba) - a faster, independent replacement for `conda` written in `C++`, created by a non-profit [for the future of the ecosystem](https://medium.com/@QuantStack/open-software-packaging-for-science-61cecee7fc23). It now supports nearly all of `conda` features, with a few remaining limitations related to environments (see below). Install `mamba`:

- directly with [Mambaforge](https://mamba.readthedocs.io/en/latest/installation.html),
- or using `conda`.

Mamba includes extra features, like `query`, so make sure to check their [docs](https://mamba.readthedocs.io/en/latest/user_guide/mamba.html) too.

Below, you can use `conda` or `mamba` interchangeably, with a few [exceptions](#mamba-limitations).

## Conda basics

### Get help

```bash
conda -h # general help
conda install -h # command-specific help
```

### Packages

The `conda` manager can search the default channels as well as any additional channel you configure (see below). You can also search packages in all channels at [anaconda.org](https://anaconda.org/).

```bash
conda search pigz # search parallel gzip in available channels
conda search --info pigz # information about pigz package, including dependencies
conda search -c conda-forge pigz # search parallel gzip in conda-forge channel

conda install pigz # install pigz package from default channel
conda install -c conda-forge pigz # install newer version of pigz package from conda-forge channel
conda install pigz=2.4 # install specific version of pigz package

conda update conda # updates conda package
conda update --all # updates all packages

conda clean --all # cleans downloaded packages (already installed) to free up disk space
```

### Channels
Add channels to your conda configuration, so they are searched by default. Order of adding determines **priority**. The most useful channels are:

- [conda-forge](https://conda-forge.org/) - general science software, including packages for R, and more;
- [Bioconda channel](http://bioconda.github.io/) - for bioinformatics software, depends on the conda-forge channel (here comes the importance of priority).

```bash
# last channel = highest priority
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict # new since conda>=4.6
```

After adding channels into the config as above, they are searched by commands such as `conda search` or `conda install`, even without the `-c|--channel` parameter.

> ⚠️ **Note:** The Bioconda channel packs software only for macOS and Linux, but it also works on Windows 10 or higher using [WSL](linux_on_windows.html#windows-subsystem-for-linux-wsl).

### Environments
Create [isolated environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) to **avoid dependency conflicts** between installed packages or to **install multiple versions** of the same program side-by-side.

```bash
conda create -n py3.5 python=3.5 # create new environment with specific python version

conda create -n bioinfo fastqc trimmomatic bwa # create new environment with some bioinformatic tools
conda install -n bioinfo samtools gatk freebayes # install packages into existing environment

conda-env list # list available environments
conda info --envs # list available environments

conda activate py3.5 # activate new environment
conda deactivate # deactivate current environment

conda list # list packages installed in current env
conda list -n bioinfo # list packages installed in bioinfo env
conda list -n bioinfo | grep samtools # check if particular package is installed in given env
```

Conda allows [rolling back to previous version](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#restoring-an-environment) of an environment.

```bash
conda list --revisions # list revisions of current environment
conda install --rev 3 # roll back to rev 3 of current environment
```

### Mamba limitations
Mamba currently supports:

- creating environments,
- activating/deactivating environments.

But it does not support:

- rolling back revisions of environment's history

Well - yet. Give them time and [check again](https://github.com/mamba-org/mamba/issues/803) soon! 😉️

## Comments

