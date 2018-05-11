---
date: 2018-05-07
title: "SLURM Job Submission with R"
description: Getting started with the Sherlock Cluster at Stanford University
categories:
  - tutorial
type: Tutorial
set: clusters
set_order: 1
tags: [resources]
---

**Scenario** 

You just finished up a really cool analysis in R, and you need to scale it. An HPC
cluster with a job manager such as SLURM is a great way to do this! In this tutorial,
we will walk through a very simple method to do this. First, let's talk about our
strategy for today.

 1. Write an executable script in R / Python
 2. Organize your inputs, output location, and scripts.
 3. Loop over some set of input variables and submit a SLURM job to use your executable to process each one.

We are going to have the following scripts, for each of Python and R (to exemplify both)

 1. `run.py` / `run.R` will be the executable script
 2. /scratch/users/vsochat/project/LizardLips will be our organized data folder, and we will store scripts in a version controlled $HOME/projects/LizardLips
 3. submit_jobs.py, submit_jobs.R, and submit_jobs.sh will be example scripts that loop over some set of input variables to submit SLURM jobs. We We show a basic shell
script to do the submission, because very often you don't need much more than that.
 
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
```

if I saved this in a script called "run.R" I could then execute:

```bash
Rscript run.R tomato potato shiabato
```

and input1 would be assigned to "tomato," and "potato" and "shiabato" to input2 and input3, respectively. By the way, if you aren't familiar with Rscript, it's the executable that will let you run and R script out of the R statistical environment. We are going to be using Rscript in our work today!

### Using Python
Python is just as easy! Instead of commandArgs, we use the `sys` module. The
same would look like this:

```python
import sys

input1 = sys.argv[1]
input2 = sys.argv[2]
input3 = sys.argv[3]
```

Calling would then look like:

```bash
python run.py tomato potato shiabato
```

`sys.argv` is actually just a list of your calling script and the input arguments
following it. If you are keen, you'll realize that Python starts indexing at 0, and we are skipping over the value at `sys.argv[0]`. This would actually coincide to 
the name of your script. If you are interested in advanced input parsing, then you
should look at <a href="https://docs.python.org/3/library/argparse.html" target="_blank">argparse</a>. You can read about our example using argparse for a module <a href="https://vsoch.github.io/lessons/python-packaging/#setuppy" target="_blank">entrypoint here</a>, go <a href="https://gist.github.com/vsoch/da2362f1c5d2747e34c92b99a3473842#file-scripts-py" target="_blank">directly to the gist</a>, or just download the script directly:

```bash
wget https://gist.githubusercontent.com/vsoch/da2362f1c5d2747e34c92b99a3473842/raw/1d379e37b33ad2a1c748819b4c6d49f1fea85b30/scripts.py
```

### A Little About Executables
When you write your executable, it's good practice to **not hard code any variables**
For example, if my script is going to process an input file, I could take in just
a subject identifier and then define the full path in the script:

```
## R Example DONT DO THIS
...
subid = args[3]
input_path = paste('/scratch/users/vsochat/projects/LizardLips/',subid,'/data.csv', sep='')
```
```python
## Python Example DONT DO THIS
...
subid = sys.argv[3]
input_path = '/scratch/users/vsochat/projects/LizardLips/%s/data.csv' % subid
```

But guess what? If you change a location, your script breaks. Instead, assign this path duty to the calling script (we haven't written this yet) that is going to loop over variables and call this script. This means that your executable should instead
expect a _general_ `input_path`:

```
## R Example DO THIS
...
input_path = args[3]

(!file.exists(input_path)){
   cat('Cannot find', input_path, 'exiting!\n')
   stop()
}
```
```python
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
your `$HOME` folder is backed up. This also means you have a stricter quota, and should use it for scripts and valuables (and not data). Under `$HOME`, it's recommended to adopt a structure based on version controlled code. For example, I might have a folder `$HOME/projects` and inside that folder I would clone Github repositories that are relevant to my work. **Everything** would be commit, and if you are a pro, you would have testing. Generally,

> You should be able to completely rm -rf your entire `$HOME` directory and be able 
to easily reproduce it because your code is version controlled.

### Data goes in $SCRATCH
`$SCRATCH` is a good place for temporary data outputs, and `$SCRATCH_LOCAL` for even
more temporary (files that are only needed at runtime). If you have a more long term data storage resource (e.g., `$OAK` at Stanford) then you might store data here too). This means that you might have an equivalent folder setup on `$SCRATCH` for projects,
`$SCRATCH/projects` and subfolders of projects under that. If it's a shared project between your group, you should put it in `$PI_SCRATCH` or `$OAK`. For example, for my project "LizardLips" with subject folders "LizardA" and "LizardB" I might decide on this output
structure:

```bash
/scratch/users/vsochat/
                 projects/
                     LizardLips/
                              LizardA
                              LizardB
```

### Inputs go in $SCRATCH
Now, arguably if you have a small input file (e.g., a csv) it would be OK to store it in your `$HOME`. But with all this big data I'm betting that your input data is large too. The trick here is that you want to create an organizational setup where you can
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

of course you can call it whatever you like, or is appropriate for your domain or work.
There are many known data organization standards (e.g., <a href="http://bids.neuroimaging.io/" target='_blank'>BIDS</a> for neuroimaging) and you should have
a discussion with your lab mates, and (highly recommended) reach out to <a href="https://www.twitter.com/StanfordCompute">research computing</a> to have a meeting
to discuss a data storage strategy.

## loop submission using your executable
You then want to loop over some set of input variables (for example, csv files with data.) You can imagine doing this on your computer - each of the inputs would be processed in serial. Your time to completion, where T is the time for one script to run, is N (number of inputs) * T. That can take many hours if you have a few hundred
inputs that each take 10 minutes, and it's totally unfeasible if you have thousands of simulations, each of which might need 30 minutes to an hour.

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

If it falls within the node limits and accounting, you will see that the job is submit.

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
Here I will show you very basic example in each of R, Python, and shell to loop
through a set up input variables (the subject identifier to derive an input path)
to generate job files, and then submit them. We can assume the following:

 - **the number of jobs to submit is within our accounting limits**, so we will submit them all at once (meaning they get added to the queue).
 - **at the start of the script, you check for existence of directories.** Usually you will need to create a top level or subject level directory somewhere in the loop, given that it doesn't exist. Don't forget to do this.
 - **you have permission to write to where you need to write.** This not only means that you have write permission, but if you are writing to a shared space, you make sure that others will too. If you are running analyses with containers, you must be certain that you are not attempting to write anything inside the container (they are read only!) but rather in a writable directory mounted to it.

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
newly created job with the last line.

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

#### R Submission
This example shows using a relative path to the job file, along
with referencing `$SCRATCH` as a variable. Careful doing this!

For the below, we are going to use "sink", which is just a lazy man's way to write to file, and then the output of cat goes into the file. You can also use cat and specify the file="" argument.

```R
!#/usr/bin/env Rscript

scratch = os.environ['SCRATCH']
data_dir = os.path.join(scratch, '/project/LizardLips')

lizards=c("LizardA","LizardB")

for (lizard in lizards) {

    lizard_data = paste(data_dir,lizard,sep="")

    # Start writing to this file
    sink(paste(".job/",lizard,'.job',sep=""))

    # the basic job submission script is a bash script
    cat("#!/bin/bash\n")
    cat("#SBATCH --job-name=",lizard",".job\n", sep="")
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
Again,I recommend NOT doing this programatically first when testing, but to do `sbatch LizardA.job` from the console 

#### Final Notes
Finally, here are a few good job submission practices.

 - **Always use full paths**. For the scripts, it might be reasonable to use relative paths, and this is because they are run in context of their own location. However, in the case of data and other files, you should always use full paths.
 - **You should generally not run anything en-masse before it is tested.** This means that after you write your loop script, you might step through it manually (don't be ashamed to open up an interactive R or Python console and copy paste away!), submit the job, ensure that it runs successfully, and inspect the output. I can almost guarantee you will have a bug or find a detail that you want to change. It's much easier to do this a few times until you are satisfied and then launch en-masse over launching en-masse and realizing they are all going to error.


Additional Tips

You can run sacct to see how busy the sherlock allocation is.  Each node (how many do we have?) has 128 Gigs of RAM.  Anyone in your group will have exclusive priority on these nodes.  Your group members need to run jobs with the -p option:

srun/sbatch -p langlotz

This will put their jobs on your nodes and they will have priority, any jobs not run by Langlotz/Rubin/Napel members will be stopped.


Also, i/o will always be way faster in /scratch, after analysis is done their files can be moved to /project or /home.

More about storage options-

http://sherlock.stanford.edu/mediawiki/index.php/DataStorage#.24PI_HOME


Website with common UNIX Commands

http://mally.stanford.edu/~sr/computing/basic-unix.html





Using Sherlock
Get started
Logging on
What is Sherlock?
To logon to the system
Mounting Sherlock on your local machine
Setting up your workspace
Data storage
HOME
PI_HOME
SCRATCH
PI_SCRATCH
LOCAL_SCRATCH
OAK
Using Sherlock
Moving data
Submitting jobs
Other useful commands

Get started
Logging on
What is Sherlock?
Sherlock is the Stanford cluster.  This means that you can login to Sherlock from a secure shell (SSH), and run scripts across many computational nodes.  This quick start guide should get your familiar with working with Sherlock. For additional information you should consult the Sherlock documentation: http://www.sherlock.stanford.edu/docs/
To logon to Sherlock 1.0
1. Set up kerberos on your workstation:
 i. 	Download Kerberos Commander
1.     https://uit.stanford.edu/service/ess/mac/kerberos
ii. 	Set-up Kerberos on your computer
1.     http://sherlock.stanford.edu/mediawiki/index.php/SetupKerberos
2.     You can set up an alias for logging into Sherlock from your local machine. This will mean that you don’t need to type the whole login path each time.
i. 	Enter a Terminal/Shell
 ii. 	Change into your home directory and open your bash profile
1.     Type: ‘cd ~/’
2.     Type: ‘pwd’
a.     This will tell you if you’re on your local machine or in Sherlock
b.     Look at the path to make sure you’re on your local machine!
3.     Type: ‘nano .bash_profile’
4.     Paste the following into your .bash_profile:
5.     alias sherlock='ssh –XY username@sherlock.stanford.edu'
6.     Save the ‘.bash_profile’ by typing ‘control + X’
7.     Press Enter
8.     Press Y (to save changes)
NOTE: you still need to initialize credentials before your session to be able to use your alias
To logon to Sherlock 2.0
No more Kerberos necessary! (You can still use it if you want)
ssh <sunet>@login.sherlock.stanford.edu

This will prompt you to 2 factor authentication so you should make sure to have that set up for your SUNet ID beforehand.
Mounting Sherlock 1.0 on your local machine
If you would like to be able to edit files locally on your machine one way to do this is mount Sherlock locally. This mean you would create directories on your local machine and temporarily access Sherlock directories through them. Once your connection is turned off these local directories are empty.

i. Download osxfuse
1.     Mac: https://osxfuse.github.io/
2.     PC through a Linux virtual box: http://manpages.ubuntu.com/manpages/wily/man1/fusermount.1.html
ii. Create three new folders on your local machine where you can access the Sherlock data (i.e. your mount point). We recommend putting them in your LOCAL HOME directory. (For the below example they are called e.g. sherlock_scratch/)
Iii. To mount the Sherlock directories in your local directories use the following commands
1.     sshfs username@sherlock.stanford.edu:/scratch/PI/russpold ~/sherlock_scratch/
2.     sshfs username@sherlock.stanford.edu:/share/PI/russpold ~/sherlock_share/
3.     sshfs username@sherlock.stanford.edu:/home/PI/russpold ~/sherlock_home/
For more information on how to ssh to sherlock.stanford.edu, check out:
For Sherlock 1.0: http://sherlock.stanford.edu/mediawiki/index.php/LogonCluster
For Sherlock 2.0:
Setting up your workspace

Your .bash_profile is a script that is read when you log on to a node (this applies to when you’re running jobs as well). 

To get to your .bash_profile 
cd 
nano .bash_profile

Pasting the following to the shared stack into your .bash_profile will automatically install some software that the lab uses frequently
source /share/PI/russpold/software/setup_all.sh

Any additional aliases etc. can be added in the .bash_profile as well (see alias example for Sherlock under ‘Logging on’). 

If you want to install different versions of software you can add paths to these versions in your .bash_profile as well. The following is an example of adding a specific version of miniconda3 installed in the user’s home directory. 
export PATH=$HOME/miniconda3/bin:$PATH

To check whether your paths have been updated successfully you can check with e.g.
which python

Modules
An alternative to hard-coded settings in your .bash_profile file is using modules. Sherlock provides a complete list of modules, but does not cover neuroimaging-specific software. To get access to our share list of software, please insert the following command line (or execute all times before your scripts and batch scripts):

Using russpold’s modules
$ module use /share/PI/russpold/modules

After that, a new entry in your available modules (that you can list with module av) will appear:

Custom modules available after adding russpold’s modules
$ module av
----------------------- /share/PI/russpold/modules ------------------------
   afni/17.3.03 (D)      anaconda/5.0.0-py36 (D)    ants/2.1.0.post710 (D)
   c3d/1.1.0             freesurfer/6.0.1 (D)       git-annex/6.20171109
   gsl/2.3 (D)

… clipped output … 

Running mriqc natively:
# load dependencies
module load afni ants c3d fsl freesurfer
# load anaconda, which has the latest stable mriqc installed:
module load anaconda
mriqc --version

File systems

Information on the structure of Sherlock can be found here:
For Sherlock 1.0: http://sherlock.stanford.edu/mediawiki/index.php/DataStorage
For Sherlock 2.0: http://www.sherlock.stanford.edu/docs/user-guide/storage/

All scratch space is now subject to automatic purge. That means that files that are older than a certain age will be deleted regularly. Our rule of thumb therefore is now to store all data on OAK. 

For Sherlock 2.0
Be advised that the paths of the /home and /share/PI directories have changed on Sherlock 2.0. They are now respectively /home/users and /home/groups.
We strongly recommend referring to them through the $HOME and $PI_HOME environment variables instead of using the full paths.

HOME
This where you will start out when you log in (unless you change it in your .bash_profile)
/home/username 
/home/users/username

It’s not very big or efficient. You can store code etc. here but it wouldn’t be useful for big datasets. You could store your own software here.
PI_HOME
/share/PI/russpold
/home/groups/russpold

This will NOT be purged. It is a better candidate to store shared lab data and software.
SCRATCH
/scratch/users/username

Efficient and large but this is subject to purge. Nothing should be stored here for long term.
PI_SCRATCH
/scratch/PI/russpold

This directory will be subject to automatic purging.
LOCAL_SCRATCH
/local-scratch/username/17299371


Using Sherlock
Moving data
For Sherlock 1.0: When moving things from your local machine you should use SFTP to connect to sherlock (I used gFTP) to copy files from your local machine to scratch.  The client must support kerberos - this means that Filezilla will not work, nor will konqueror.
If you are moving larger files from elsewhere to OAK ….
Submitting jobs
More thorough information on this can be found here:
http://www.sherlock.stanford.edu/docs/getting-started/submitting/

Here we list some useful commands that we often use.

Running on a Computing Node / Interactive Jobs
When you login to Sherlock, you're actually on one of two login nodes. There are two: sherlock-ln01, and sherlock-ln02. You can specify which node to log into from your ssh, like:

ssh -Y myname@sherlock-ln01.stanford.edu

would log in to login node 1.  On these home nodes you can submit batch jobs, but for any other work you will need to move to a computing node:

srun -t 2-00 --pty bash

With this command you get an interactive node available for 2 days (-t 2-00) in an interactive mode (--pty) Bash shell. If you want to be the exclusive user of the node, add --exclusive.

You can also designate how much memory you would like to use:

srun --mem=32G --pty bash

For short term interactive jobs you can use
sdev
which gives you a 8G node for 2 hours.

Remember: no real work should be done on login nodes!

Reserving an interactive node
srun -p russpold --exclusive --time=48:00:00 --qos=russpold_interactive --pty bash
The above command reserves an interactive node for 48 hours on the russpold partition

Only 2 QOSes are allowed on the "russpold" partition:
* --qos=russpold replaces the previous default of "normal", must be specified for each job
* --qos=russpold_interactive can be used to submit interactive jobs there, and get priority over batch jobs.

Running parametric jobs

You can run your (shell) scripts in parallel using sbatch command:

sbatch  -o out.%j -e err.%j yourScript.sh in1 in2

This command will submit yourScript.sh to run on the cluster with in1 and in2 as input parameters (i.e., $1 and $2 in the script).

(Standard) Output of this script will be saved as out.%j (%j will be replaced with the jobID) in the current directory, and $STDERR will be in err.%j.

More information on this can be found here: http://sherlock.stanford.edu/mediawiki/index.php/SLURMSubmit

Job-arrays
If you are to run a series of command lines that are independent (e.g. you prepared a for loop with an sbatch call to run 1st level analysis for each participant of your sample), you are better off using a job-array. Since you can limit the number of tasks in the job array that are to be run simultaneously, this is a way of being a better citizen sharing the resources. This is a template you can use for your sbatch file:

Job-array template
#!/bin/bash
#
#SBATCH -J firstlevel
#SBATCH --array=1-250%10  # <-- note the %10 that limits the parallel tasks to 10

#SBATCH --time=02:30:00
#SBATCH -n 1
#SBATCH --cpus-per-task=16
#SBATCH --mem-per-cpu=4G

#SBATCH --qos=russpold
#SBATCH -p russpold
#SBATCH --export=NONE

# Outputs ----------------------------------
#SBATCH -o %A-%a.out
#SBATCH -e %A-%a.err
#SBATCH --mail-user=<your-sunet-id>@stanford.edu
#SBATCH --mail-type=ALL
# ------------------------------------------

module use /share/PI/modules/
module load singularity
export FS_LICENSE=$PWD/.freesurfer.txt
eval $( sed "${SLURM_ARRAY_TASK_ID}q;d" tasks_list.sh )

This script will:
Create a job with name “firstlevel” (#SBATCH -J firstlevel).
Make it a job array, with 250 tasks with identifiers from 1 to 250. Also, a maximum of 10 tasks will be running in parallel at any given time (#SBATCH --array=1-250%10).
Request the following resources: one task per node (-n 1), all 16 available cpus (--cpus-per-task=16), a total of 64GB of RAM (--mem-per-cpu=4G), for 2h 30min (--time=02:30:00) and allocated in the “russpold” queue (-p russpold, --qos=russpold).
The tasks run are written in a file called (tasks_list.sh) that is found in the folder used as working directory. Each line of the tasks list file may look like (make sure this is only one line!):
my_first_level.py $OAK/data/openfmri/ds000224 $OAK/data/openfmri/derivatives/ds000224 participant -w work/ds000224 --participant_label sub-MSC05


$ module load system gdrive
$ gdrive help

You can load modules or set environment variables before the tasks (which are indexed by the variable $SLURM_ARRAY_TASK_ID) are run.
Useful commands

Check version of Sherlock you are on
echo $SHERLOCK

See what modules are available
ml av

Load and run a module (e.g. rstudio)
module load rstudio
rstudio
Useful SLURM commands

Show the overall status of each partition
sinfo

Submit a job
sbatch .jobs/jobFile.job

See the entire job queue
squeue

See only jobs for a given user
squeue -u username

Count number of running / in queue jobs
squeue -u username | wc -l

Get estimated start times for your jobs (when Sherlock is busy)
squeue --start -u username

Show the status of a currently running job
sstat -j jobID

Show the final status of a finished job
sacct -j jobID

Kill a job with ID $PID
scancel $PID

Kill ALL jobs for a user
scancel -u username

Kill all pending jobs
scancel -u username --state=pending

Run interactive node with 16 cores (12 plus all memory on 1 node) for 4 hours:
srun -n 12 -N 1 --mem=64000 --time 4:0:0 --pty bash

Claim interactive node for exclusive use, 8 hours
srun --exclusive --time 8:0:0 --pty bash

Same as above, but with X11 forwarding
srun --exclusive --time 8:0:0 --x11 --pty bash

Same as above, but with priority over your other jobs
srun --nice=9999 --exclusive --time 8:0:0 --x11 --pty -p dev -t 12:00 bash

Check utilization of group allocation
sacct

Running jobs in the group allocation
srun -p russpold
sbatch -p russpold

This will put jobs on your nodes where they will have priority. The poldracklab has 15 reserved computing nodes (also 16 cpu’s, 64GB memory each).  To run your job on these nodes, add this to your job file (or replace the link to the ‘normal’ queue):
SBATCH --qos=russpold
SBATCH -p russpold

Stop/restart jobs interactively
To stop:
scancel -s SIGSTOP job id
To restart:
scancel -s SIGCONT job id
(p.s.- this wont free-up memory)

Get usage for file systems
df -k
df -h $HOME

Get usage for your home directory 
du

Checking Disk Quota
lfs quota -u <sunetid> -h /scratch/
lfs quota -g <pi_sunetid> -h /scratch/

Counting Files
find . -type f | wc -l
```
lizards=["LizardA","LizardB"]
