---
date: 2018-08-27
title: "Singularity Quick Start"
description: A Quick Start to using Singularity on the Sherlock Cluster
categories:
  - tutorial
type: Tutorial
set: clusters
set_order: 11
tags: [resources]
---

This is a quick start to using Singularity on the Sherlock cluster. You should have familiarity
with how to log in, and generally use a command line. If you run into trouble, please
<a href="https://www.github.com/vsoch/lessons/issues" target="_blank">ask us for help</a>.

## Login

<a href="https://www.sherlock.stanford.edu/docs/getting-started/connecting/" target="_blank">Here are</a> 
the official "How to Connect to Sherlock" docs. Generally you 
<a href="https://vsoch.github.io/lessons/kerberos/" target="_blank">set up kerberos</a> and do this:

```bash
ssh <username>@login.sherlock.stanford.edu
```

I'm lazy, so my preference is to define this setup in my `~/.ssh/config` file. By setting
the login node to be a specific one I can maintain a session over time. Let' say my
username is "tacos"

```bash

Host sherlock
    User tacos
    Hostname sh-ln05.stanford.edu
    GSSAPIDelegateCredentials yes
    GSSAPIAuthentication yes
    ControlMaster auto
    ControlPersist yes
    ControlPath ~/.ssh/%l%r@%h:%p
```

Then to login I don't need to type the longer string, I can just type:

```bash
ssh sherlock
```

## Interactive Node
You generally shouldn't run anything computationally intensive on the login nodes! It's not
just to be courteous, it's because the processes will be killed and you have to start over.
Don't waste your time doing this, grab an interactive node to start off:

```bash
# interactive node
sdev

# same thing, ask for different memory or time
srun --time 8:00:00 --mem 32000 pty bash
```

If your PI has <a href="https://srcc.stanford.edu/sherlock-high-performance-computing-cluster" target="_blank">purchased nodes</a> 
and you have a partition for your lab,
this means that your jobs will run a lot faster (you get priority as a member of the group!) and you
should use that advantage:

```
srun --time 8:00:00 --mem 32000 --partition mygroup --pty bash
```

If you don't have any specific partition, the implied one is `--partition normal`.

## Load Singularity
Let's get Singularity set up! You may want to add this to your bash profile (in HOME)
under `$HOME/.bashrc` (or similar depending on your shell) so that you don't need to manually
do it. 

```bash

module use system
module load singularity
```

A very import variable to export is the `SINGULARITY_CACHEDIR`. This is where Singularity stores
pulled images, built images, and image layers (e.g., when you pull a Docker image it first pulls
the `.tar.gz` layers before assembling into a container binary). What happens if you **don't** export
this variable? The cache defaults to your HOME, your HOME quickly gets filled up (containers
are large files!) and then you are locked out.

 >> don't do that.

```bash

export SINGULARITY_CACHEDIR=$SCRATCH/.singularity
mkdir -p $SINGULARITY_CACHEDIR
```

Also note for the above that we are creating the directory, which is something that needs to be
done once (and then never again!).

# Examples

Now let's jump into examples. I'm just going to show you, because many of the commands
speak for themselves, and I'll add further note if needed. What you should know that we
are going to reference a container with a "uri" (unique resource identifier) and it can
be in reference to (the most common forms):

 - **docker://**: a container on Docker Hub
 - **shub://**: a container on Singularity Hub
 - **container.simg**: a container image file

## Pull

You **could** just run or execute a command to a container reference, but I recommend
that you pull the container first.

```bash

singularity pull shub://vsoch/hello-world
singularity run $SCRATCH/.singularity/vsoch-hello-world-master-latest.simg
```

It's common that you might want to name a container based on it's Github commit (for Singularity Hub),
it's image file hash, or a custom name that you really like.

```bash

singularity pull --name meatballs.simg shub://vsoch/hello-world
singularity pull --hash shub://vsoch/hello-world
singularity pull --commit shub://vsoch/hello-world
```

## Exec

The "exec" command will execute a command to a container.

```bash

singularity pull docker://vanessa/salad
singularity exec $SCRATCH/.singularity/salad.simg /code/salad spoon
singularity exec $SCRATCH/.singularity/salad.simg /code/salad fork
```
```

# Other options for what you can do (without sudo)
singularity build container.simg docker://ubuntu
singularity exec container.simg echo "Custom commands"
```

## Run
The run command will run the container's runscript, which is the executable (or `ENTRYPOINT`/`CMD` 
in Docker speak) that the creator intended to be used.

```bash

singularity pull docker://vanessa/pokemon
singularity run $SCRATCH/.singularity/pokemon.simg run catch
```

## Options
And here is a quick listing of other useful commands! This is by no means
an exaustive list, but I've found it helpful to debug and understand a container.
First, inspect things!

```bash

singularity inspect -l $SCRATCH/.singularity/pokemon.simg  # labels
singularity inspect -e $SCRATCH/.singularity/pokemon.simg  # environment
singularity inspect -r $SCRATCH/.singularity/pokemon.simg  # runscript
singularity inspect -d $SCRATCH/.singularity/pokemon.simg  # Singularity recipe definition
```

Using `--pwd` below might be necessary if the working directory is important. 
Singularity doesn't respect Docker's `WORKDIR` Dockerfile
command, as we usually use the present working directory.

```bash

# Set the present working directory with --pwd when you run the container
singularity run --pwd /code container.simg
```

Singularity is great because most of the time, you don't need to think about
binds. The paths that you use most often (e.g., your home and scratch and tmp)
are "inside" the container. I hesitate to use that word because the
boundry really is seamless. Thus, if your host supports 
<a href="https://en.wikipedia.org/wiki/OverlayFS" target="_blank">overlayfs</a> and the configuration allows it, your container
will by default see all the bind mounts on the host. You can specify a custom mount 
(again if the administrative configuration allows it) with `-B` or `--bind`.

```bash

# scratch, home, tmp, are already mounted :) But use --bind/-B to do custom
singularity run --bind $HOME/pancakes:/opt/pancakes container.simg
```

I many times have conflicts with my PYTHONPATH and need to unset it,
or even just clean the environment entirely.

```bash

PYTHONPATH= singularity run docker://continuumio/miniconda3 python
singularity run --cleanenv docker://continuumio/miniconda3 python
```

What does writing a script look like? You are going to treat singularity as
you would any other executable! It's good practice to load it in your 
<a href="https://researchapps.github.io/job-maker/" target="_blank">SBATCH</a>
scripts too.

```bash

module use system
module load singularity
singularity run --cleanenv docker://continuumio/miniconda3 python $SCRATCH/analysis/script.py
```

## Build
If you build your own container, you have a few options!

 - Created an [automated build for Docker Hub](https://docs.docker.com/docker-hub/builds/) then pull the container via the `docker://` uri.
 - add a `Singularity` recipe file to a Github repository and build it automatically on [Singularity Hub](https://www.singularity-hub.org).
 - [Ask for @vsoch help](https://www.github.com/researchapps/sherlock) to build a custom container! 
 - Check out all the [example](https://github.com/researchapps/sherlock) containers to get you started.
 - [Browse the Containershare](https://vsoch.github.io/containershare/) and use the [Forward tool](https://www.github.com/vsoch/forward/) for interactive jupyter (and other) notebooks.
 - [Build your container](https://www.sylabs.io/guides/2.6/user-guide/quick_start.html#build-images-from-scratch) locally, and then use [scp](http://www.hypexr.org/linux_scp_help.php) to copy it to Sherlock:

```bash
scp container.simg <username>@login.sherlock.stanford.edu:/scratch/users/<username>/container.simg
```


Have a question? Or want to make these notes better? Please <a href="https://www.github.com/vsoch/lessons/issues">open an issue</a>.
