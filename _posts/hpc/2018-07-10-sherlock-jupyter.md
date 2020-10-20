---
date: 2018-07-9
title: "Jupyter Notebooks on Sherlock"
description: Use Jupyter Notebooks on Sherlock with Port Forwarding
categories:
  - tutorial
type: Tutorial
set: clusters
set_order: 7
tags:
 - tutorial
 - resources
---

Today we are going to walk through using a [tool](https://github.com/vsoch/forward) provided by one of our users to set up port forwarding on Sherlock. What does this mean? It means you...

 - will run a password protected jupyter notebook on a cluster node
 - can access the notebook in a browsr on your local machine

It also means that *any tool that is served on a port* that you can run on an interactive node via an sbatch job could
be accessed on your computer. Very cool! 

## Why should I use this?

Likely some of you are doing this on your own with basic bash commands, and this repository nicely packages those commands.

### Collaborative Scripts

The repository has a folder of general sbatch scripts that expose ports that can be forwarded. If you add a new script that you find useful and want to share with others, we encourage you to <a href="https://github.com/vsoch/forward" target="_blank">tell us about it</a>! If it's not perfect or finished, don't be afraid to share! We will provide any additional help needed to finish up your script.

### Community Support

This tool is brought to you by one of our community <a href="https://github.com/raphtown" target="_blank">talented users</a> that reached out to the user list. It's an amazing example of the goodness that can come out of sharing your code because others can use it too.


## 1. On Your Computer
The repository contains a set of scripts for setting up automatic port forwarding on sherlock with jupyter notebooks.
There are a set of commands you will need to run just once to configure the tool, and then general "start", "end" and "resume" operations for interacting with notebook jobs. The repository README has instructions for setup and usage, and we will also walk through them here.

### Step 1: Download the Repository
Make sure you put the folder somewhere meaningful. A subfolder of `$HOME` is a suggestion. This will be your working location for future use of the tool, as it holds scripts and a parameter file, `params.sh`

```bash
git clone https://github.com/vsoch/forward
cd forward
```

Let's take a look at what we have here!

```bash
├── end.sh    (-- end a session
├── resume.sh (-- resume a started session
├── sbatches  (-- batch scripts you can run to forward a port to!
  ├── jupyter.sbatch
  └── tensorboard.sbatch
├── setup.sh  (-- run once to set up the tool
└── start.sh  (-- how you start a connection
```

### Step 2: Generate your Parameters
This is a one time generation script that will create a parameter text file with a port, username, and cluster resource. You can generate it by running the script [setup.sh](https://github.com/vsoch/forward/blob/master/setup.sh):

```bash
$ bash setup.sh 

Sherlock username > tacocat

Next, pick a port to use.  If someone else is port forwarding using that
port already, this script will not work.  If you pick a random number in the
range 49152-65335, you should be good.

Port to use > 56143

Next, pick the sherlock partition on which you will be running your
notebooks.  If your PI has purchased dedicated hardware on sherlock, you can use
that partition.  Otherwise, leave blank to use the default partition (normal).

Sherlock partition (default: normal) > 

Next, pick the path to the browser you wish to use.  Will default to Safari.

Browser to use (default: /Applications/Safari.app/) > /usr/bin/firefox
```

Notice that I've changed the default browser (Safari on a Mac) to firefox on Ubuntu. Also note that I pressed enter
to use the default queue. The resulting file is a simple text file:

```bash
$ cat params.sh 
USERNAME="tacocat"
PORT="56143"
PARTITION="normal"
BROWSER="/usr/bin/firefox"
MEM="20G"
TIME="8:00:00"
```

### Step 3: SSH Credentials
It wouldn't be so great if anyone with your username and a port could access your notebook! For this reason we need to create ssh credentials. You might have this already set up if you've followed the <a href="https://www.sherlock.stanford.edu/docs/getting-started/connecting/" target="_blank">instructions here</a> for Sherlock. You
can see what you have by looking at your `~/.ssh/config`:

```bash
cat ~/.ssh/config
```

What we want to do is have this configuration. Add this to your `~/.ssh/config`. The repository also has a quick way to generate your ssh config, and print it to the screen. See the hosts folder? In there are small helper scripts to print the configurations to the screen! We have one for Sherlock:

```bash
ls hosts/
  sherlock_ssh.sh
```
Running the script will print the configuration to the screen:

```bash
$ bash hosts/sherlock_ssh.sh 

Sherlock username > tacocat
Host sherlock
    User tacocat
    Hostname login.sherlock.stanford.edu
    GSSAPIDelegateCredentials yes
    GSSAPIAuthentication yes
    ControlMaster auto
    ControlPersist yes
    ControlPath ~/.ssh/%l%r@%h:%p
```

If you don't have a file existing at `~/.ssh/config` then you will need to create it, and you
can do the entire step above programatically:

```bash
bash hosts/sherlock_ssh.sh >> ~/.ssh/config
```

Your computer is ready to go! Let's move onto Sherlock. We will come back here when it's time to start a notebook.

## 2. On Sherlock

<br>

### 2.1. Jupyter notebook
We will want to set up a password for our Jupyter notebook, and we can do this programatically
before starting it. First, let's load the module for Python, and install jupyter notebook! The version of
Python that you choose is up to you.


```bash
-ln07 login! ~/]$ module spider jupyter

----------------------------------------------------------------------------
  py-jupyter:
----------------------------------------------------------------------------
    Description:
      Jupyter is a browser-based interactive notebook for programming,
      mathematics, and data science. It supports a number of languages via
      plugins.

     Versions:
        py-jupyter/1.0.0_py27
        py-jupyter/1.0.0_py36

----------------------------------------------------------------------------
  For detailed information about a specific "py-jupyter" module (including how to load the modules) use the module's full name.
  For example:

     $ module spider py-jupyter/1.0.0_py36
----------------------------------------------------------------------------
```

Let's load the Python 3.6 jupyter.

```bash
$ module load py-jupyter/1.0.0_py36
$ which jupyter
/share/software/user/open/py-jupyter/1.0.0_py36/bin/jupyter
```

We will need to secure our notebooks with a password. Pick a strong one!

```bash
jupyter notebook password
```
It will be associated with your username, saved to a local file:

```bash
Enter password: 
Verify password: 
[NotebookPasswordApp] Wrote hashed password to /home/users/vsochat/.jupyter/jupyter_notebook_config.json
```

The above might pause or hang a little bit, at least it did when I did (and I suspect someone
was running a resource intensive process on the node...)

## 3. Usage

We have just set up a password on Sherlock, and are now back on our _local machine_. Here are the general commands to start and stop sessions. In the tutorial below, we will walk through using Jupyter notebook.


### 3.1. Start a Session
From the directory where we cloned, we can start a session using the [start.sh](https://github.com/vsoch/forward/blob/master/start.sh) script. This is a general script to start any kind of session, and here we will show how to start a jupyter notebook in a specific directory:

```bash
bash start.sh <software> <path>
bash start.sh jupyter /path/to/dir
```

What's going on? It will look in the folder of [sbatch scripts](https://github.com/vsoch/forward/blob/master/sbatches) and find one named correpondingly to the command we issued. This means that there is a file called [jupyter.sbatch](https://github.com/vsoch/forward/blob/master/sbatches/jupyter.sbatch) in that folder. Note that, to
make this simple, we also have sbatch scripts for using jupyter with different Python kernels:


Let's fire up a notebook with Python 3!

```bash
$ bash start.sh py3-jupyter /scratch/users/vsochat
== Checking for previous notebook ==
No existing py3-jupyter jobs found, continuing...
== Getting destination directory ==
== Uploading sbatch script ==
py3-jupyter.sbatch                                                                                      100%  132     0.1KB/s   00:00    
== Submitting sbatch ==
Submitted batch job 21659278
== Waiting for job to start, using exponential backoff ==

Attempt 0: not ready yet... retrying in 1..
Attempt 1: not ready yet... retrying in 2..
Attempt 2: not ready yet... retrying in 4..
Attempt 3: not ready yet... retrying in 8..
Attempt 4: not ready yet... retrying in 16..
Attempt 5: not ready yet... retrying in 32..
Attempt 6: not ready yet... retrying in 64..
...
Attempt 6: resources allocated to sh-101-21!..
sh-101-21
notebook running on sh-101-21
== Setting up port forwarding ==
ssh -L 56143:localhost:56143 sherlock ssh -L 56143:localhost:56143 -N sh-101-21 &
== Connecting to notebook ==
Open your browser to http://localhost:56143
```

When you open your browser to the address, you will see a prompt for the password that you
created previously:

![/lessons/assets/img/tutorials/jupyter-password.png](/lessons/assets/img/tutorials/jupyter-password.png)

If you have an already running session, you will see this message:

and then log in to see your scratch!

![/lessons/assets/img/tutorials/jupyter-scratch.png](/lessons/assets/img/tutorials/jupyter-scratch.png)


### 3.2. Resume a Session
Sometimes the job can still be running, but the port forwarding has stopped (has your computer
ever gone to sleep?) In this case, you can resume the session using the equivalent command, but
use resume.sh:

```bash
bash resume.sh py3-jupyter`
```

```bash
$ bash start.sh py3-jupyter /scratch/users/vsochat
== Checking for previous notebook ==
Found existing job for py3-jupyter, sh-101-04 end.sh or resume.sh
```

### 3.3. Stop a Session

When you want to stop the session (killing the job) run the equivalent command, but use the end.sh script. 
The command is based on the name of the job (the sbatch script name) so to kill the previous job we created,
we would do:

```bash
$ bash end.sh py3-jupyter /scratch/users/vsochat
Killing py3-jupyter slurm job on sherlock
Killing listeners on sherlock
```

There are additional details and debugging tips in the <a href="https://github.com/vsoch/forward" target="_blank">main repository</a>! Happy Jupyter-ing!

## Contribute!
The cool thing about small efforts like these is that they provide very useful tools for you
and the larger Stanford community. We can continue to work on them together to make them even better,
and add support for more functions and kinds of sessions. This is the awesome thing about open source!
Here are some suggestions for how you can contribute:


### Adding new sbatch scripts
You can add more sbatch scripts by putting them in the sbatches directory. You should assume that an SBATCH script
will be run on a job node, and should take a port as the first argument, and directory to work from as the second. Other additional arguments are up to you! For example, here is what the jupyter batch job looks like:

```bash
#!/bin/bash

PORT=$1
NOTEBOOK_DIR=$2
cd $NOTEBOOK_DIR

module load py-jupyter/1.0.0_py27
jupyter notebook --no-browser --port=$PORT
```

Contributing can also be as simple as suggesting a feature or reporting a bug on the <a href="https://github.com/vsoch/forward/issues" target="_blank">issues board</a>. 
See the <a href="https://github.com/vsoch/forward" target="_blank">main repository</a> for how to contribute. 

### Add Functionality

Right now we can start, stop (end), and resume. Wouldn't it be cool to also have a script to print a status?

Do you have questions or want to see another tutorial? Please <a href="https://www.github.com/vsoch/lessons/issues">reach out</a>!
