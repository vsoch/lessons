---
date: 2018-08-05
title: "Using Pytorch and Singularity on Sherlock"
description: A custom built pytorch and Singularity image on the Sherlock cluster
categories:
  - tutorial
type: Tutorial
set: clusters
tags: [resources]
---

Here is a quick getting started for using pytorch on the Sherlock cluster! We have pre-built two containers, Docker containers, then we have pulled onto the cluster as Singularity containers that can help you out:

 - <a href="https://github.com/researchapps/sherlock/tree/master/pytorch-dev" target="_blank">README</a> with instructions for using one of several pytorch containers provided.
 - Singularity container with python 2.7, and pytorch [<a href="https://github.com/researchapps/sherlock/blob/master/pytorch-dev/Dockerfile.py2" target="_blank">Dockerfile</a>]
 - Singularity container with python 3.5, and pytorch [<a href="https://github.com/researchapps/sherlock/blob/master/pytorch-dev/Dockerfile" target="_blank">Dockerfile</a>]
 - Both containers on <a href="https://hub.docker.com/r/vanessa/pytorch-dev/" target="_blank">Docker Hub</a>

For tutorials using the containers in detail, see the first link with the README. In the following
example, we will show using the Python 2.7 container.

<hr>


## Getting Started

In the getting started snippet, we will show you how to grab an interactive gpu node using
`srun`, load the needed libraries and software, and then interact with torch (the module import name
for pytorch) to verify that we have gpu.

**Copy the container that you need from @vsoch shared folder**

```bash

# Pytorch with python 2.7 (this is used in tutorial below)
cp /scratch/users/vsochat/share/pytorch-dev-py2.7.simg $SCRATCH

# Pytorch with python 3
cp /scratch/users/vsochat/share/pytorch-dev.simg $SCRATCH

# Pytorch with python 3 (provided by pytorch/pytorch on Docker Hub)
cp /scratch/users/vsochat/share/pytorch-0.4.1-cuda9-cudnn7-devel.simg $SCRATCH
```

**Grab an interactive node with gpu**

```bash
$ srun -p gpu --gres=gpu:1 --pty bash
```

**Load modules Singularity and cuda library**

```bash

$ module use system
$ module load singularity
$ module load cuda
```

Don't forget to copy the container, if you didn't above!

```bash

cp /scratch/users/vsochat/share/pytorch-dev-py2.7.simg $SCRATCH
cd $SCRATCH
```

**Shell into the container**

Now we are going to shell into the Singularity container! Note the `--nv` flag here, 
it means "nvidia" and will expose the libraries on the host for the container to see.
You can use this flag with "exec" and "run" too!

```bash

$ singularity shell --nv pytorch-dev-py2.7.simg 
Singularity: Invoking an interactive shell within container...

Singularity pytorch-dev-py2.7.simg:~> python
```

We just launched python! Let's now import torch, and generate a variable called
"device" that will be "cuda:0" if we are using the gpu, or "cpu" if not. 

```python

Python 2.7.14 |Intel Corporation| (default, May  4 2018, 04:27:35) 
[GCC 4.8.2 20140120 (Red Hat 4.8.2-15)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
Intel(R) Distribution for Python is brought to you by Intel Corporation.
Please check out: https://software.intel.com/en-us/python-distribution
>>> import torch
>>> import torch.nn as nn
>>> from torch.autograd import Variable
>>> device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu") 
>>> print(device)
cuda:0
```

Tada! And from the above we see that we have successfully set up the container to work
with Python 2.7 and GPU. You can now read more about pytorch to get started with machine
learning.

Thanks to one of our awesome users in the <a href="https://cocolab.stanford.edu/" target="_blank">CoCoLab</a> for helping to develop this container and tutorial!

If you need a refresher with job submission, check out <a href="https://vsoch.github.io/lessons/sherlock-jobs/" target="_blank">our post on SLURM</a>.
Do you have questions or want to see another tutorial? Please <a href="https://www.github.com/vsoch/lessons/issues">reach out</a>!
