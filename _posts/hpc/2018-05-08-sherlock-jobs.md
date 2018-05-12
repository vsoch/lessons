---
date: 2018-05-07
title: "SLURM Job Submission with R, Python, Bash"
description: Getting started with the Sherlock Cluster at Stanford University
categories:
  - tutorial
type: Tutorial
set: clusters
set_order: 1
tags: [resources]
---

In this tutorial we will write a job submission script for SLURM. If you haven't yet,
you should:

 - be comfortable accessing <a href="https://vsoch.github.io/lessons/sherlock/">the Sherlock cluster</a>
 - understand <a href="https://vsoch.github.io/lessons/slurm/">SLURM job submission</a>

and then move forward with this tutorial!

<hr>

### Scenario

You just finished up a really cool analysis, and you need to scale it. An HPC
cluster with a job manager such as SLURM is a great way to do this! In this tutorial,
we will walk through a very simple method to do this. First, let's talk about our
strategy for today.

 1. Write an executable script in R / Python
 2. Organize your inputs, output location, and scripts.
 3. Loop over some set of variables and submit a SLURM job to use your executable to process each one.

We will cover each of these steps in detail.

## Write an Executable Script
You first have some script in R or Python. It likely reads in data, processes it, and creates a result. You will need to turn this script into an executable, meaning that it accepts variable arguments. 

### Using R
R actually makes this very easy. While there are advanced input parsers, you can retrieve your script inputs with just a few lines:

```R
args = commandArgs(TRUE)
input1 = args[1]
input2 = args[2]
input3 = args[3]

# Your code here!
```

if I saved this in a script called "run.R" I could then execute:

```bash
$ Rscript run.R tomato potato shiabato
```

and `input1` would be assigned to "tomato," and "potato" and "shiabato" to `input2` and `input3`, respectively. By the way, if you aren't familiar with <a href="https://stat.ethz.ch/R-manual/R-devel/library/utils/html/Rscript.html" target="_blank">Rscript</a>, it's literally
the R script executable. We are going to be using it in our work today!

### Using Python
Python is just as easy! Instead of commandArgs, we use the `sys` module. The
same would look like this:

```python
import sys

input1 = sys.argv[1]
input2 = sys.argv[2]
input3 = sys.argv[3]

# Your code here!
```

Calling would then look like:

```bash
$ python run.py tomato potato shiabato
```

`sys.argv` is actually just a list of your calling script and the input arguments
following it. If you are keen, you'll realize that Python starts indexing at 0, and we are skipping over the value at `sys.argv[0]`. This would actually coincide to 
the name of your script. If you are interested in advanced input parsing, then you
should look at <a href="https://docs.python.org/3/library/argparse.html" target="_blank">argparse</a>. You can read about our example using argparse for a module <a href="https://vsoch.github.io/lessons/python-packaging/#setuppy" target="_blank">entrypoint here</a>, or go <a href="https://gist.github.com/vsoch/da2362f1c5d2747e34c92b99a3473842#file-scripts-py" target="_blank">directly to the gist</a>.

### A Little About Executables
When you write your executable, it's good practice to **not hard code any variables**
For example, if my script is going to process an input file, I could take in just
a subject identifier and then define the full path in the script:

```R
## R Example DONT DO THIS
...
subid = args[3]
input_path = paste('/scratch/users/vsochat/projects/LizardLips/',subid,'/data.csv', sep='')

## Python Example DONT DO THIS
...
subid = sys.argv[3]
input_path = '/scratch/users/vsochat/projects/LizardLips/%s/data.csv' % subid
```

But guess what? If you change a location, your script breaks. Instead, assign this path duty to the calling script (we haven't written this yet). This means that your executable should instead expect a _general_ `input_path`:

```R
## R Example DO THIS
...
input_path = args[3]

(!file.exists(input_path)){
   cat('Cannot find', input_path, 'exiting!\n')
   stop()
}

## Python Example DO THIS
input_path = sys.argv[3]
if not os.path.exists(input_path)
    print('Ruhroh, %s doesn't exist!' %input_path)
    sys.exit 1
```

Notice that for both, as a sanity check we check that `input_path` exists.

> Path errors are extremely common in scientific programming, and you should always do this check.


## Organize inputs, outputs, scripts
You will hugely benefit
 if you keep your scripts and inputs / outputs organized. This generally means the following:

### Scripts go in $HOME
your $HOME folder is backed up. This also means you have a stricter quota, and should use it for scripts and valuables (and not data). Under $HOME, it's recommended to adopt a structure based on version controlled code. For example, I might have a folder $HOME/projects and inside that folder I would clone Github repositories that are relevant to my work. **Everything** would be commit, and if you are a pro, you would have testing. Generally,

> You should be able to completely lose your entire $HOME and be OK because the code is under version control.

### Data goes in $SCRATCH
$SCRATCH is a good place for temporary data outputs. If you have a more long term data storage resource (e.g., $OAK at Stanford) then you might store data here too). This means that you might have an equivalent folder setup here for project data,
$SCRATCH/projects and subfolders of projects under that. If it's a shared project between your group, you could put it in $PI_SCRATCH. For example, for my project "LizardLips" with subject folders "LizardA" and "LizardB" I might decide on this output
structure:

```bash
/scratch/users/vsochat/
                 projects/
                     LizardLips/
                              LizardA
                              LizardB
```

#### Inputs
Now, arguably if you have a small input file (e.g., a csv) it would be OK to store it in your $HOME. But with all this big data I'm betting that your input data is large too. The trick here is that you want to create an organizational setup where you can
always link an input object (subject, sample, timepoint, etc.) to its output based on a unique identifier. In the data organization above, we see that our data is organized based on subjects (LizardA and LizardB) and you can imagine now having a programmatically defined input and output location for each:

```bash
LizardLips/
    LizardA/
         input/
        result/
    LizardB/
         input/
        result/
```

With the above structure, I can have confidence that my inputs for `LizardA`, if they exist, are in `/scratch/users/vsochat/LizardLips/LizardA/inputs`. What do you name these folders? It's largely up to you, and you should choose names appropriate for your data.
There are many known data organization standards (e.g., <a href="http://bids.neuroimaging.io/" target='_blank'>BIDS</a> for neuroimaging) and you should have
a discussion with your lab mates, and (highly recommended) reach out to <a href="https://www.twitter.com/StanfordCompute">research computing</a> to have a meeting
to discuss a data storage strategy.

## Loop submission using your executable
You then want to loop over some set of input variables (for example, csv files with data.) You can imagine doing this on your computer - each of the inputs would be processed in serial. Your time to completion, where T is the time for one script to run, and N is the number of inputs, is N * T. That can take many hours if you have a few hundred
inputs each taking 10 minutes, and it's totally unfeasible if you have thousands of simulations, each of which might need 30 minutes to an hour.

### Strategy 1: Submit a Job File
As a graduate student I liked having a record of what I had run, and an easy way to
re-run any single job without needing to run my submission script again. Before we make a job file, let me show you what it looks like:

```bash
#!/bin/bash
#SBATCH --job-name=LizardA.job
#SBATCH --output=.out/LizardA.out
#SBATCH --error=.out/LizardB.err
#SBATCH --time=2-00:00
#SBATCH --mem=12000
#SBATCH --qos=normal
#SBATCH --mail-type=ALL
#SBATCH --mail-user=$USER@stanford.edu
Rscript $HOME/project/LizardLips/run.R tomato potato shiabato
```

Importantly, notice the last line! It's just a single line that calls our script to run our job. In fact, look at the entire file, and the interpreter at the top - `#!/bin/bash` - it's **just** a bash script! The only thing that makes it a little different is all of the `#SBATCH` commands. What's that about? This is actually the
syntax that SLURM understands as a configuration argument for your job. It just corresponds with the way that you submit the job to slurm using the 
<a href="https://slurm.schedmd.com/sbatch.html" target="_blank">`sbatch` command</a>.
In fact, you are free to write whatever lines that you need after the `SBATCH` lines.
You can expect that the node running the job will have all the same information
that you have on a login node. This means it will source your bash profile, and know the locations of your $HOME and $SCRATCH. It also means that you can run the same
commands like `module load` if you need special software for the job.

**What do all the different variables mean?**

Some of the above are obvious, like `--mem` corresponds to memory in GB, `--time` in
the format above means 2 days, and the output and error correspond to file paths to write output and error to. For full descriptions of all the options, the best source is the man pages (linux manual) which you can read via:

```
$ man sbatch
```

If you just had the one job file above, let's say it were called `LizardA.job`, you 
would submit it like this to slurm:

```bash
sbatch LizardA.job
```

If it falls within the node limits and accounting, you will see that the job is submit. If you need a helper tool to generate
just one template, check out the <a href="https://researchapps.github.io/job-maker/" target="_blank">Job maker</a> that I put together
a few years ago.

### Strategy 1: Submit Directly to sbatch
What if you had the script RScript, and you didn't want to create a job file? You can do the exact same submission using sbatch directly:

```bash
sbatch --job-name=LizardA.job \
       --output=.out/LizardA.out \
       --error=.out/LizardB.err
       --time=2-00:00 \
       --mem=12000 \
       --qos=normal \
       Rscript $HOME/project/LizardLips/run.R tomato potato shiabato
```

and then of course you would need to reproduce that command to run it again. This is why my preference is for writing a job file. But then it follows that if we have hundreds of jobs to submit, we need hundreds of files. How do we do that?

### Write a Loop Submission Script
Here I will show you very basic example in each of R, Python, and bash to loop
through a set up input variables (the subject identifier to derive an input path)
to generate job files, and then submit them. We can assume the following:

 - **the number of jobs to submit is within our accounting limits**, so we will submit them all at once (meaning they get added to the queue).
 - **at the start of the script, you check for existence of directories.** Usually you will need to create a top level or subject level directory somewhere in the loop, given that it doesn't exist.
 - **you have permission to write to where you need to write.** This not only means that you have write permission, but if you are writing to a shared space, you make sure that others will too.


#### Bash Submission
Bash scripting is the most general solution, and we will start with it. Here is a basic
template to generate the SBATCH script above. Let's call this script `run_jobs.sh`

```bash
#!/bin/bash

# We assume running this from the script directory
job_directory=$PWD/.job
data_dir="${SCRATCH}/project/LizardLips"

lizards=("LizardA" "LizardB")

for lizard in ${lizards[@]}; do

    job_file="${job_directory}/${lizard}.job"

    echo "#!/bin/bash
#SBATCH --job-name=${lizard}.job
#SBATCH --output=.out/${lizard}.out
#SBATCH --error=.out/${lizard}.err
#SBATCH --time=2-00:00
#SBATCH --mem=12000
#SBATCH --qos=normal
#SBATCH --mail-type=ALL
#SBATCH --mail-user=$USER@stanford.edu
Rscript $HOME/project/LizardLips/run.R tomato potato shiabato" > $job_file
    sbatch $job_file

done

```
Notice that we are echoing the job into the `$job_file`. We are also launching the
newly created job with the last line `sbatch $job_file`.

#### Python Submission

With Python, You basically need to do the above, but print to a file using Python.
There are multiple ways to do this, here is one example!

```python
#!/usr/bin/env python

import os

def mkdir_p(dir):
    '''make a directory (dir) if it doesn't exist'''
    if not os.path.exists(dir):
        os.mkdir(dir)
    

job_directory = "%s/.job" %os.getcwd()
scratch = os.environ['SCRATCH']
data_dir = os.path.join(scratch, '/project/LizardLips')

# Make top level directories
mkdir_p(job_directory)
mkdir_p(data_dir)

lizards=["LizardA","LizardB"]

for lizard in lizards:

    job_file = os.path.join(job_directory,"%s.job" %lizard)
    lizard_data = os.path.join(data_dir, lizard)

    # Create lizard directories
    mkdir_p(lizard_data)

    with open(job_file) as fh:
        fh.writelines("#!/bin/bash\n")
        fh.writelines("#SBATCH --job-name=%s.job\n" % lizard)
        fh.writelines("#SBATCH --output=.out/%s.out\n" % lizard)
        fh.writelines("#SBATCH --error=.out/%s.err\n" % lizard)
        fh.writelines("#SBATCH --time=2-00:00\n")
        fh.writelines("#SBATCH --mem=12000\n")
        fh.writelines("#SBATCH --qos=normal\n")
        fh.writelines("#SBATCH --mail-type=ALL\n")
        fh.writelines("#SBATCH --mail-user=$USER@stanford.edu\n")
        fh.writelines("Rscript $HOME/project/LizardLips/run.R %s potato shiabato\n" %lizard_data)

    os.system("sbatch %s" %job_file)
```

The last line submits a job by sending the command to the console. Note that if you
want to do this function in actual software, you would want to use <a href="https://docs.python.org/3/library/subprocess.html" target="_blank">subprocess</a>.

#### R Submission
This example shows using a relative path to the job file, along
with printing $SCRATCH as a variable in the file instead of the actual path.

For the below, we are going to use "sink", which is just a lazy man's way to write to file, and then the output of cat goes into the file. You can also use cat and specify the file="" argument.

```R
!#/usr/bin/env Rscript

lizards=c("LizardA","LizardB")

for (lizard in lizards) {

    lizard_data = paste(data_dir,lizard,sep="")

    # Start writing to this file
    sink(paste(".job/",lizard,'.job',sep=""))

    # the basic job submission script is a bash script
    cat("#!/bin/bash\n")
    cat("#SBATCH --job-name=",lizard,".job\n", sep="")
    cat("#SBATCH --output=.out/",lizard,".out\n", sep="")
    cat("#SBATCH --error=.out/",lizard,".err\n", sep="")
    cat("#SBATCH --time=2-00:00\n")
    cat("#SBATCH --mem=12000\n")
    cat("#SBATCH --qos=normal\n")
    cat("#SBATCH --mail-type=ALL\n")
    cat("#SBATCH --mail-user=$USER@stanford.edu\n")
    cat("Rscript $HOME/project/LizardLips/run.R",lizard_data,"potato shiabato\n", sep=" ")

    # Close the sink!
    sink()

    # Submit to run on cluster
    system(paste("sbatch",paste(".job/",lizard,".job",sep="")))

}
```
Again,I recommend NOT doing this programatically first when testing, but to do manual runs first.

## Good Practices

Finally, here are a few good job submission practices.

 - **Always use full paths**. For the scripts, it might be reasonable to use relative paths, and this is because they are run in context of their own location. However, in the case of data and other files, you should always use full paths.
 - **Don't run large computation on the login nodes!** It negatively impacts all cluster users. Grab a development node with `sdev`.
 - **Think about how much memory you actually need**. You want to set a bit of an upper bound so a spike doesn't kill the job, but you also don't want to waste resources when you (or someone else) could be running more jobs.
 - **You should generally not run anything en-masse before it is tested.** This means that after you write your loop script, you might step through it manually (don't be ashamed to open up an interactive R or Python console and copy paste away!), submit the job, ensure that it runs successfully, and inspect the output. I can almost guarantee you will have a bug or find a detail that you want to change. It's much easier to do this a few times until you are satisfied and then launch en-masse over launching en-masse and realizing they are all going to error.

And as a reminder, here are some useful SLURM commands for checking your job.

```bash
# Show the overall status of each partition
sinfo

# Submit a job
sbatch .jobs/jobFile.job

# See the entire job queue
squeue

# See only jobs for a given user
squeue -u username

# Count number of running / in queue jobs
squeue -u username | wc -l

# Get estimated start times for your jobs (when Sherlock is busy)
squeue --start -u username

# Show the status of a currently running job
sstat -j jobID

# Show the final status of a finished job
sacct -j jobID

# Kill a job with ID $PID
scancel $PID

# Kill ALL jobs for a user
scancel -u username

# Kill all pending jobs
scancel -u username --state=pending

# Run interactive node with 16 cores (12 plus all memory on 1 node) for 4 hours:
srun -n 12 -N 1 --mem=64000 --time 4:0:0 --pty bash

# Claim interactive node for exclusive use, 8 hours
srun --exclusive --time 8:0:0 --pty bash

# Same as above, but with X11 forwarding
srun --exclusive --time 8:0:0 --x11 --pty bash

# Same as above, but with priority over your other jobs
srun --nice=9999 --exclusive --time 8:0:0 --x11 --pty -p dev -t 12:00 bash

# Check utilization of group allocation
sacct

# Running jobs in the group allocation
srun -p groupid
sbatch -p groupid

# Stop/restart jobs interactively
# To stop:
scancel -s SIGSTOP job id

# To restart (this won't free up memory):
scancel -s SIGCONT job id

# Get usage for file systems
df -k
df -h $HOME

# Get usage for your home directory 
du

# Checking Disk Quota
lfs quota -u <sunetid> -h /scratch/
lfs quota -g <pi_sunetid> -h /scratch/

# Counting Files
find . -type f | wc -l
```

Do you have questions or want to see another tutorial? Please <a href="https://www.github.com/vsoch/lessons/issues">reach out</a>!
