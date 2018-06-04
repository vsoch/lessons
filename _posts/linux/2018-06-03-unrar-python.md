---
date: 2018-06-02
title: "UnRAR an Archive"
description: How to create and extract rar archives with Python and containers
resources:
  - name: "Comparison of Archive Formats"
    link: https://en.wikipedia.org/wiki/Comparison_of_archive_formats#Containers_and_compression
  - name: "How to Open, Extract, and Create RAR files in Linux"
    link: https://www.tecmint.com/how-to-open-extract-and-create-rar-files-in-linux/
  - name: "RAR on Wikipedia"
    link: https://en.wikipedia.org/wiki/RAR_(file_format)
  - name: "Other Python Modules for RAR (Stack Overflow)"
    link: https://stackoverflow.com/questions/17614467/how-can-unrar-a-file-with-python
  - name: singularityhub/rar Github repository
    link: https://www.github.com/singularityhub/rar
  - name: Singularity Hub Container
    link: https://www.singularity-hub.org/collections/1080
  - name: "Singularity Containers"
    link: https://singularityware.github.io
type: Document
set: clusters
set_order: 6
tags: [linux,python,archive]
---

Today we are going to use Python to extract a <a href="https://en.wikipedia.org/wiki/RAR_(file_format)" target="_blank">RAR archive</a>. You may have never heard of this format, or vaguely remember something called 
<a href="https://en.wikipedia.org/wiki/WinRAR" target="_blank">WinRAR</a>. Keep in mind there are a 
<a href="https://en.wikipedia.org/wiki/Comparison_of_archive_formats#Containers_and_compression" target="_blank">crapton</a>
of options, and if you get to choose, you should choose optimally for your problem at hand. Today we will focus on RAR files and work with them using Python and then with a Singularity container. Let's get started!

## Why Containers or Python?

> You are working on a shared resource where you can't install system tools for working with RAR, but you can install Python packages, or use Singularity containers!

We are going to use some standard <a href="https://www.tecmint.com/how-to-open-extract-and-create-rar-files-in-linux/" target="_blank"> command line linux tools</a> to create a dummy archive to extract, and then demonstrate doing the extraction
in Python. Yes, we _could_ just interact with the files using the linux tools, but we are under the assumption that you can't do these installations. Perhaps you are a researcher and have downloaded RAR archives from a web address, and you also want to interact with the contents in Python.

## Create an Archive
If you ever wanted to create a RAR archive on linux (and please consider <a href="https://en.wikipedia.org/wiki/Comparison_of_archive_formats#Containers_and_compression" target="_blank"> others first </a> you could do this with "rar" and "unrar" (cue dinosaur "RAAAWR!"

```bash
# Install in Debian/Ubuntu
sudo apt-get install -y rar unrar
```

Let's make a silly folder of useless files to turn into an archive.

```bash
mkdir -p noodles
echo "This is what you live on in graduate school." >> noodles/ramen.txt
echo "This is a meaty noodle that probably your Dad likes." >> noodles/udon.txt
echo "Garfield approved, best with a napkin." >> noodles/lasagna.txt
mkdir -p noodles/sauce
touch noodles/sauce/marinara
touch noodles/sauce/cheese
touch noodles/sauce/alfredo
```

Oh yeah, we have a nice little thing going on here!

```bash
$ tree noodles/
noodles/
├── lasagna.txt
├── ramen.txt
├── sauce
│   ├── alfredo
│   ├── cheese
│   └── marinara
└── udon.txt

1 directory, 6 files
```

We can now create the archive with "rar"

```bash
$ rar a noodles.rar noodles

RAR 5.30 beta 2   Copyright (c) 1993-2015 Alexander Roshal   4 Aug 2015
Trial version             Type RAR -? for help

Evaluation copy. Please register.

Creating archive noodles.rar

Adding    noodles/sauce/alfredo                                       OK 
Adding    noodles/sauce/cheese                                        OK 
Adding    noodles/sauce/marinara                                      OK 
Adding    noodles/udon.txt                                            OK 
Adding    noodles/ramen.txt                                           OK 
Adding    noodles/lasagna.txt                                         OK 
Adding    noodles/sauce                                               OK 
Adding    noodles                                                      1%
Done
```

Great! We now have an archive to work with. We can guess that the `a` says to "add files" to an archive. You should be comfortable with reading the man (manual) pages to learn more about rar, try this:

```bash
$ man rar
```

## Extract in Python

There are actually a <a href="https://stackoverflow.com/questions/17614467/how-can-unrar-a-file-with-python" target="_blank">couple of ways to do this</a>, and I'll use a module that I know called "patool."
<a href="https://pypi.org/project/patool/" target="_blank">Check it out</a> because it handles
much more kinds of archives than rar. Here is how to install it:

```bash

pip install patool

# On a shared resource
pip install patool --user
```

Now let's open up Python! Note that the noodles.rar is in our present working directory.

```python
rarfile = 'noodles.rar'
```

The simplest thing to do, in order to extract the archive to the present working
directory (and let's create a new folder for it first) would look something like this:

```python

from patoolib import extract_archive
import os
extract_to = 'new-noodles'
os.mkdir(extract_to) 
extract_archive(rarfile, outdir=extract_to)

patool: Extracting noodles.rar ...
patool: running /usr/bin/rar x -- /home/vanessa/Documents/Dropbox/Code/shub/containers/rar/noodles.rar
patool:     with cwd='new-noodles'
patool: ... noodles.rar extracted to `new-noodles'.
```

Now you probably want to parse over them. How can you do that? Let's write a quick function! This
would return a list:

```python
def list_files(base):
    files = []
    for root, dirnames, filenames in os.walk(base):
        for filename in filenames:
            files.append(os.path.join(root, filename))
    return files
```

Test it out!

```

In [11]: list_files(extract_to)
Out[11]: 
['new-noodles/noodles/udon.txt',
 'new-noodles/noodles/ramen.txt',
 'new-noodles/noodles/lasagna.txt',
 'new-noodles/noodles/sauce/alfredo',
 'new-noodles/noodles/sauce/cheese',
 'new-noodles/noodles/sauce/marinara']
```

And here is a function that would return an iterator. You might want this for
larger listings for more efficient parsing.

```python

def iter_files(base):
    for root, dirnames, filenames in os.walk(base):
        for filename in filenames:
            yield os.path.join(root, filename)
```
```
for filename in iter_files(extract_to):
    ...:     print('I found a file %s!' %filename)
    ...:     
I found a file new-noodles/noodles/udon.txt!
I found a file new-noodles/noodles/ramen.txt!
I found a file new-noodles/noodles/lasagna.txt!
I found a file new-noodles/noodles/sauce/alfredo!
I found a file new-noodles/noodles/sauce/cheese!
I found a file new-noodles/noodles/sauce/marinara!
```

The paths above are relative. You would want to use `os.path.abspath(os.path.join(root, filename))`
for a full path. It depends what you are trying to do.

```python
def list_fullpaths(base):
    files = []
    for root, dirnames, filenames in os.walk(base):
        for filename in filenames:
            files.append(os.path.abspath(os.path.join(root, filename)))
    return files
```
```
In [15]: list_fullpaths(extract_to)
Out[15]: 
['/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/udon.txt',
 '/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/ramen.txt',
 '/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/lasagna.txt',
 '/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/sauce/alfredo',
 '/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/sauce/cheese',
 '/home/vanessa/Documents/Dropbox/Code/shub/containers/rar/new-noodles/noodles/sauce/marinara']
```

## Rar-ing with a Container
Let's say you don't want to deal with Python, but you still can't install rar or unrar. You can use a container!
If you aren't familar with Singularity, it's a container technology (like Docker) that works on a shared resource.
You can install it with <a href="https://singularityware.github.io/install-linux" target="_blank">these instructions</a>.

[![https://www.singularity-hub.org/static/img/hosted-singularity--hub-%23e32929.svg](https://www.singularity-hub.org/static/img/hosted-singularity--hub-%23e32929.svg)](https://singularity-hub.org/collections/1080)

The container is hosted on Singularity Hub, and built from the <a href="https://www.github.com/singularityhub/rar" target="_blank">repository here</a>.


```bash
$ singularity pull --name rar.simg shub://singularityhub/rar
```


If you are extracting on a shared resource, make sure to export your `SINGULARITY_CACHEDIR` first, as
pulling to `$HOME` can fill up the quota almost immediately `#fatcontainers`.

```bash

export SINGULARITY_CACHEDIR=$SCRATCH/.singularity
mkdir -p ${SINGULARITY_CACHEDIR}
$ singularity pull --name rar.simg shub://singularityhub/rar

```

You can always ask the container for help before blindly running it.

```bash
$ singularity help rar.simg 


    This container provides the rar and unrar utilities. Running the container
    as is is akin to running the rar utility on Linux:
       ./rar.simg --help

    If you want to create or extract an archive, you can also use one of the apps
    provided:

        singularity apps rar.simg
          create
          extract

    Or ask for help for usage for one:

       $ singularity help --app create rar.simg 
       $ singularity help --app extract rar.simg 
   
```

## Create and Extract An Archive
Here is how to create an archive using the Singularity container:

```bash
$ singularity run --app create rar.simg archive.rar noodles/
```

And now extract the same one, but somewhere else.

```bash
$ singularity run --app extract rar.simg archive.rar another-noodles
```

## Advanced Usage
You can interact with unrar and rar in the containers directly if you use "exec".
Here are the same commands, but we are calling the executables directly!

```bash
# Create
$ singularity exec rar.simg rar a new-archive.rar noodles/

# Extract
$ singularity exec rar.simg unrar x new-archive.rar more-noodles/

UNRAR 5.30 beta 2 freeware      Copyright (c) 1993-2015 Alexander Roshal


Extracting from new-archive.rar

Creating    more-noodles                                              OK
Creating    more-noodles/noodles                                      OK
Extracting  more-noodles/noodles/udon.txt                             OK 
Extracting  more-noodles/noodles/ramen.txt                            OK 
Extracting  more-noodles/noodles/lasagna.txt                          OK 
All OK
```

Finally, if you want to be working inside the image, you can use "shell"

```bash
$ singularity shell rar.simg
Singularity: Invoking an interactive shell within container...

Singularity rar.simg:~/Documents/Dropbox/Code/shub/containers/rar> which rar
/usr/bin/rar
Singularity rar.simg:~/Documents/Dropbox/Code/shub/containers/rar> which unrar
/usr/bin/unrar
```

Did you enjoy this post? Please <a href="https://github.com/vsoch/lessons" target="_blank">star the repository on Github</a> to show support! If you have a question for the dinosaur debugger, or would like to request a lesson, you can 
<a href="https://github.com/vsoch/lessons/issues" target="_blank"> do that here.</a>

Can you make this post better? <a href="{{ site.repo }}/issues" target="_blank">Submit an issue</a> or
better yet, make the edit yourself with a pull request <a href="{{ site.repo }}/issues" target="_blank">to the repository!</a>.
