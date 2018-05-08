---
date: 2018-05-08
title: An Introduction to Python Packages
description: A tutorial to help create your first python package
categories:
  - packaging
resources:
  - name: "The Python Package Index"
    link: https://pypi.org/
type: Document
set: packaging
set_order: 1
tags: [packaging]
---

So you want to properly publish your code. Here we are going to walk through the steps, specifically for Python. The same general ideas would apply to any other language, with adjustments for package managers, etc.

## Install Dependencies

You will want to <a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git" target='_blank'>install Git</a>, have a <a href="https://www.github.com" target="_blank">Github</a> account, and additionally, an account on the Python Package Manager, <a href="https://pypi.python.org/pypi" target="_blank">PyPi</a>.

### Setting up PyPi
Pypi used to allow package submission before creating the package (in the online interface), but it looks like this functionality has been removed (likely people were using it to reserve names in advance, shame! Just kidding, I did that all the time!). You have several <a href="https://pypi.python.org/pypi?%3Aaction=submit_form" target="_blank">options for package submission</a>, and the easiest I find to be via the command line, and will walk through the steps. First, create a file called `.pypirc` in your `$HOME` directory:


```bash
vim $HOME/.pypirc
```

inside should be the following:


```bash
[distutils]
index-servers =
  pypi

[pypi]
repository=https://pypi.python.org/pypi
username=USERNAME
password=PASSWORD
```

You should replace `USERNAME` and `PASSWORD` with your pypi username and password, respectively.

Next, to make my life easy, I like to keep a file called `pypi.sh` in each of my Python package folders, which I can just run when I want to submit a new version. This is what the contents of that file look like:

```bash
python setup.py sdist upload -r pypi
```                            

Note that you will want to <a href="https://packaging.python.org/installing/" target="_blank">install setuptools</a> to be able to run this command, along with other stuffs related to doing packaging with `setup.py`.

## Version Control
What does that mean? On the simplest level, this means storing your code in a Github Repository. That also gives you huge power to do things like set up continuous integration (testing) of your code. First, let's look at what the guts of your package should look like. This isn't the standard, but rather how I've done it over the years, which seems to work pretty well.


## Package Files

### README.md
The first thing a user is going to look at is the README. Here you should minimally have the title of your package, what it does, and any links to proper <a href="http://read-the-docs.readthedocs.io/en/latest/getting_started.html" target="_blank">documentation</a> and testing <a href="https://circleci.com/docs/status-badges/" target="_blank">badges</a>.

### LICENSE.md
Open source code should have the appropriate license, and Github is great in that it will automatically generate one for you when you create the repository, if you select it as an option when you first create the repository. I prefer the MIT license, and recommend that you <a href="http://choosealicense.com/" target="_blank">read about the different options</a>.

### The package folder
Ok, let's talk about what a python package is. Your system (or virtual environment, or local environment) python has a few special folders, `site-packages` and `dist-packages`, that other folders (the python modules) get dumped into. These aren't special other than being folders that Python "knows" to look for on its path when you try to import a module. When you import a module, it's just looking in the folders on its path for the first thing called that. By default, your present working directory (with path "") is the first place it looks, so if you have a script called `functions.py` in your present working directory and then do:

```python
from functions import make_pancakes
```

It's going to find the `functions.py` in your present directory first, and try to `make_pancakes` from that. So how do folders work, then? A folder just becomes part of that path import. For example, if I want my module to be called `pancakes` then I can make a folder called `pancakes` and put different python files in it:


```bash
pancakes/
    eating.py
    scripts.py
    stacks.py
    toppings.py
```

If I were to "install" this package now (more detail coming later), assuming there is a function called `chocolate` in `toppings.py` I would want to do something like the following:

```python
from pancakes.toppings import chocolate
```

If the package is installed correctly, that folder (along with some other metadata) is moved into my python `site-packages` folder, it's found on the path, and the import works. Now let's be a little more detailed, and talk about what that folder, and generally your package repo, actually should look like.

#### The init dot py file
If we created our package like the above, we would have a problem. I might try to do something like this:

```python
from pancakes.toppings import chocolate
```

and I would get an error in the flavor of `ImportError`. Why is that? If I investigated by looking into the `site-packages` of my Python, I would quickly discover that my files under `pancakes` weren't actually copied there! This is because we are missing an `__init__.py` in the folder. Python sniffs around for these `__init__.py` files to determine if a folder is part of a package on install. So our package needs to look like this:

```bash
pancakes/
    __init__.py
    eating.py
    scripts.py
    stacks.py
    toppings.py
```

The other cool thing about this file is that it allows you to define variables on the level of your package. Do a <a href="https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=about%20init.py" target="_blank">Google search</a> or read <a href="http://stackoverflow.com/questions/448271/what-is-init-py-for" target="_blank">Stack Overlow</a> to learn more about this.


### setup.py
The most important file, and the controller for your package, is the `setup.py` This is the thing that is going to define the version, and build and deploy the distribution to Pypi. Let's take a look at a simple example. This isn't a minimal example, but it includes the things I find most useful:

```python
from setuptools import setup, find_packages
import os

setup(
    # Application name:
    name="pancakes",

    # Version number:
    version="0.01",

    # Application author details:
    author="Vanessa Sochat",
    author_email="vsochat@stanford.edu",

    # Packages
    packages=find_packages(),

    # Data files
    include_package_data=True,
    zip_safe=False,

    # Details (put the full repo URL here)
    url="http://www.github.com/vsoch",

    license="LICENSE.md",
    description="optimal pancake making, in python",
    keywords='pancakes breakfast syrup chocolate stacks',

    install_requires = ['gitpython','numpy'],

    entry_points = {
        'console_scripts': [
            'pancakes=pancakes.scripts:main',
        ],
    },

)
```

Let's talk about the above. First, notice that we are using <a href="https://packaging.python.org/installing/" target="_blank">setuptools</a>, as was previously mentioned. You need to have that installed.

**entry_points**: refers to an executable that is put in the user's bin when they install your package. In this case, the executable is going to call the `main` function in the script `pancakes/scripts.py`. Thus, this script should know how to take in command line arguments, print help, etc. <a href="https://gist.github.com/vsoch/da2362f1c5d2747e34c92b99a3473842" target="_blank">Here</a> is an example:

{% gist da2362f1c5d2747e34c92b99a3473842 %}

**name**: should coincide with your package name (the folder name), meaning what the user imports. Simpler is better and easier, and the same rules with regard to not having weird characters and spaces applies.

**version**: is your package version. I usually go with the format `0.00` and go up by increments of hundredths (`0.00` --> `0.01`), but some go up by tenths (`0.0` --> `0.1`)

**author**: This is your full name.

**author_email**: This is your email. I think it is proper to have it correspond to your pypi account, but not required.

**packages**: I like to use the `find_packages()` function, which will basically automatically determine packages and submodules based on `__init__.py` files.

**include_package_data**: This is a boolean, True or False, to indicate if you want to include data. I prefer actually to not include this, and do the following to specify data folders (and file types) specifically:

```python
    package_data = {'pancakes.templates':['html/*.html','static/*.zip','js/*.js','css/*.css','img/*'],
                    'pancakes.data':['*.tsv','*.csv']},
```

For example, the first line says to add all files with extension `.html` in the `pancakes/templates/html` folder. The second says to add all `.csv` and `.tsv` files in the `pancakes/data` folder. If you have issues ever with not finding your data, you probably need to check this setup, or just forgot to add it period (speaking from personal experience here).

**url**: should point to the repository of your code. I prefer the code over the documentation because it usually links to the documentation and continuous integration (testing), but you easily do the opposite.

**license**: points to your license file

**description**: is a description of the package

**keywords**: are keywords to help with organization

**install_requires**: refers to a list of other packages that are required by your package. You should be sure to capture them all, and include versions if necessary. The other places to put these dependencies is in the `requirements.txt`, discussed next.


### requirements.txt
This file is essentially an easy way to use `pip` (or other) to immediately install all of the dependencies. You could do:

```bash
pip install -r requirements.txt
```

The install itself uses the `install_requires`, but a lot of automated methods (such as continuous integration services like Travis or CircleCI) look for a requirements.txt file to install first. If you are working in a clean virtual environment, you can use `pip freeze` to generate this file, which also will capture versions:

```bash
pip freeze >> requirements.txt
```

Minimally, inside of your `requirements.txt` you can have a single column list of the same packages in your `setup.py`, but without versions:

```bash
gitpython
numpy
```

This will install the latest version, and you have to be careful here in case a package update breaks some of your functionality.

## Creating a package

First, you need to create the `PKG-INFO` file to upload to pypi, which will properly register the package. When your package is tested and ready to go, do:

```bash
python setup.py sdist

Writing pancakes-0.1/setup.cfg
creating dist
Creating tar archive
removing 'pancakes-0.1' (and everything under it)
```

If you do `ls`, you will see new folders have been added, `dist` and `pancakes.egg-info`. If we look in the egg folder, we see a bunch of meta data files about our package:

```bash
ls pancakes.egg-info/
dependency_links.txt  not-zip-safe  requires.txt  top_level.txt
entry_points.txt      PKG-INFO      SOURCES.txt
```

The file called `PKG-INFO` is what we want to <a href="https://pypi.python.org/pypi?%3Aaction=submit_form" target="_blank">submit to pypi</a>. Once you add the package, you should be able to run your `pypi.sh` script to create a new distribution and submit the package. Usually like:

```bash
bash pypi.sh
```

And it will find your `~/.pypirc` file with credentials and know what to do! There are other ways to automate publishing to coincide with github pushes, but I prefer to have my Github master branch be a "development" branch, and pypi a production version. You likely want to include instructions for the user to install both in your README.md (or other docs)


```bash
# Install production version
pip install pancakes

# Install development
git clone https://www.github.com/vsoch/pancakes
cd pancakes
python setup.py install
```

## Best Practices

### Testing
Any changes to your code could break it. You should <a href="http://docs.python-guide.org/en/latest/writing/tests/" target="_blank">write proper tests</a> so that any additions are tested before adding them officially. This means that you also want a continuous integration suite like <a href="travis-ci.org" target="_blank">Travis</a> or <a href="circleci.com" target="_blank">CircleCI</a> to test your code. You should enable the testing to trigger any time a commit or PR (pull request) is issued to master, and integrate with Github to ensure that nothing is merged that has not passed testing.

### Digital Object Identifiers
A DOI is nice to have if you want to give others something to reference for a version, or in a publication. I would recommend using something like <a href="https://zenodo.org/" target="_blank">Zenodo</a> to link your repo up with a DOI. Each time you <a href="https://help.github.com/articles/about-releases/" target="_blank">create a release</a> for your Github repo, you can associate it with a DOI. This is much quicker, and much more useful, than going through the painful process of peer review, especially for something like a little piece of software that you just want to get out into the open source community.


### Documentation
Users themselves are like pancake eaters. Some prefer pancakes to be standard, maybe a little dry, and by the book. These kinds of users would really appreciate having documentation, such as with <a href="https://pythonhosted.org/an_example_pypi_project/sphinx.html" target="_blank">sphinx</a>.  For python, my preference is to use Sphinx, built automatically via integrated with <a href="http://read-the-docs.readthedocs.io/en/latest/getting_started.html" target="_blank">Read The Docs</a>.

### Examples
Others don't want pancakes by the book. They want a chocolate-chip monstrositycake appended with drizzles and bits of fruity, nutty glory. They want you to give it to them, and they want it fast. These modern, on demand users will be very happy to see a folder called `examples` in your repo, where you have a bunch of scripts that show them how to use your package. For example, I might have these examples for my users:

```bash
pancakes/
    __init__.py
    eating.py
    stacks.py
    toppings.py
examples/
    get_toppings.py
    make_stacks.py
    make_chocolatechip_monstrosity.py
```

Notice that the examples aren't included within the package, because they are most likely to be found and copy pasted from the browser. If you have data or examples that should be imported somewhere in your package, you would want to change this.


### Documentation in Code
Code style and formatting is outside of the scope of this post, but be conscious in how you name and organize your variables, and your functions. Other developers that contribute to your code base should know to put/find utility functions in a file called `utils.py`, and you should adopt some standard for comments in functions that will then be rendered automatically by something like sphinx docs. For example, I like to something along the lines of this for a function:

```python
def zip_up(file_list,zip_name,output_folder=None):
    '''zip_up will zip up some list of files into a package (.zip)
    :param file_list: a list of files to include in the zip.
    :param output_folder: the output folder to create the zip in.
    :param zip_name: the name of the zipfile to return.
    :returns zip_package: the file path to the finished zip
    '''
```

The first line has the name of the function, and a brief description of what it does. Then each of my arguments is defined as a `param`, along with what the function `returns` on the following lines. I put it into triple quotes in the case that I need to use quotes inside of it.

There are ample <a href="https://ewencp.org/blog/a-brief-introduction-to-packaging-python/" target="_blank">other great resources</a> about the additional functionality you might want! For example, you might want to have an executable written in another language like C that is compiled on install, or have your module be included with other software.

## Other questions?
If you have other questions, or want help for your project, please don't hesitate to <a href="https://researchapps.github.io/pages/support">reach out</a>.
