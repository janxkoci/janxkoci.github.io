**[Home](https://janxkoci.github.io) | [Tutorials](https://janxkoci.github.io/tutorials)**

# Jupyter and R for Scientists
This post describes the advantages of Jupyter Notebook and conda manager for scientists working with R.

> #### Motivation 
> I used to link to a blog post on anaconda.org to get my readers on the same page with Jupyter setup, but it was taken down. So I decided to write my own tutorial, which I can link to in my future posts. Everyone else is also free to share it.

## Why Jupyter & Conda?

![Jupyter notebook](https://janxkoci.github.io/img/jupyter_ape.png)

[Jupyter](http://jupyter.org) (formerly IPython) is commonly used by scientists to share results of their work in a manner that promotes reproducibility and open science. Jupyter notebook is a prototyping interface that allows to mix executable code with narrative text, equations, interactive figures, and tables, or convert the text into presentation with a few clicks. It supports over 50 programming languages, including popular data science languages like python, julia, ruby or R (notice ju-pyt-er), using special kernels.

[Conda](http://conda.io/) is an open source and cross-platform package manager, specialized in distributing science software written in various languages. The package manager can be obtained on it's own as [miniconda](https://conda.io/miniconda.html) or as part of [Anaconda distribution](https://www.anaconda.com/download/), along with over two hundreds of Python packages for science, math, engineering, and data analysis. It can then be used to install many scientific and bioinformatic packages, including jupyter and IRkernel, the R kernel for jupyter notebooks.

We can now use conda to install jupyter, R kernel, and few useful packages for phylogenetic data manipulation. Let's get to work!

## Setting up Jupyter with conda
I usually recommend installing [miniconda](https://conda.io/miniconda.html), unless your workflow is "Python-heavy", in which case you can install [Anaconda](https://www.anaconda.com/download/) instead. The difference is only with default packages: 
- **Miniconda** includes only conda manager with Python and a few dependencies. It then allows to install additional packages, including any or all from Anaconda distribution.
- **Anaconda** includes the above plus over two hundred additional python packages for scientific computing, such as `numpy` & `scipy`, Jupyter, Spyder IDE etc. If you mostly work with R, you don't need most of them.

So to install e.g. the *R Essentials* bundle, which includes many popular R packages, such as `dplyr`, `ggplot`, `shiny`, `jupyter`, and more, you can run this command:

```bash
conda install r-essentials
```

You may instead prefer to install just R with jupyter and package for phylogeny data manipulation:

```bash
conda install r-base r-irkernel jupyter r-ape
```

> See also my [cheat sheet for conda](https://janxkoci.github.io/tutorials/conda_cheatsheet.html)!

## Running R in Jupyter
To start Jupyter notebook session, just create folder for your new project and open terminal in it, then type:

```bash
jupyter-notebook
```

A browser window will open:

![Jupyter notebook start page](https://janxkoci.github.io/img/jupyter_0.png)

From there, you can create your first R notebook by clicking at the "New" button and selecting R from dropdown menu. Then your new notebook will open in a new tab:

![Jupyter notebook start page](https://janxkoci.github.io/img/jupyter_1.png)

Now you can type and execute R code directly in the cells. For example, load the phylogenetic package `ape`:


```R
library(ape)
```

Explore one of the datasets:


```R
data(woodmouse)
woodmouse
```


    15 DNA sequences in binary format stored in a matrix.
    
    All sequences of same length: 965 
    
    Labels:
    No305
    No304
    No306
    No0906S
    No0908S
    No0909S
    ...
    
    Base composition:
        a     c     g     t 
    0.307 0.261 0.126 0.306 


Check out the alignment as a character matrix - note also that Jupyter automatically truncates large tables & matrices:


```R
as.character(woodmouse)
```


<table>
<tbody>
	<tr><th scope=row>No305</th><td>n</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>g</td><td>a</td><td>c</td><td>c</td><td>c</td><td>t</td><td>a</td><td>t</td><td>a</td></tr>
	<tr><th scope=row>No304</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>t</td><td>g</td><td>t</td><td>n</td></tr>
	<tr><th scope=row>No306</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>t</td><td>a</td><td>t</td><td>a</td></tr>
	<tr><th scope=row>No0906S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>t</td><td>a</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No0908S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No0909S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No0910S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No0912S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No0913S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1103S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1007S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1114S</th><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>n</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1202S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1206S</th><td>a</td><td>t</td><td>t</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
	<tr><th scope=row>No1208S</th><td>n</td><td>n</td><td>n</td><td>c</td><td>g</td><td>a</td><td>a</td><td>a</td><td>a</td><td>a</td><td>⋯</td><td>a</td><td>a</td><td>a</td><td>c</td><td>c</td><td>c</td><td>n</td><td>n</td><td>n</td><td>n</td></tr>
</tbody>
</table>



Calculate a distance matrix from the alignment:


```R
dmat <- dist.dna(woodmouse)
```

Plot a NJ tree:


```R
plot(nj(dmat))
```


    
![png](conda_jupyteR_files/conda_jupyteR_12_0.png)
    


## Comments
If you'd like to leave a comment, you can **join the discussion on [Github][gist]**.

[gist]: https://gist.github.com/janxkoci/5ecd85dda9c8e4aa90c823ebddbe55bc
