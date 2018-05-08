---
date: 2018-05-06
title: Contributing
categories:
  - contribute
description: How to Contribute
type: Document
tags: [contribute]
---

It's so great that you want to contribute! There are several ways to do this.

 - [Contribute Code](https://www.github.com/vsoch/lessons) meaning these documentation pages
 - [Add A Tutorial](#add-a-tutorial) a snippet of knowledge to render on these pages
 - [Ask a question](https://www.github.com/vsoch/lessons/issues), anything on your mind.

## Using Jekyll
The contents of the lessons repository renders directly on Github Pages because
it's a Jekyll site. Jekyll is a framework written in ruby that takes some configuration
values, a template and markdown files, and prettifies it up to render as a website.
The way it works is that any content that you push to the master branch of the repository
renders automatically at `https://vsoch.github.io/lessons`. It's pretty neat!

### Local Setup
To set up Jekyll locally, you can follow the [instructions](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)
provided on Github. When you have jekyll installed, you can start a local server with:

```bash
$ jekyll serve
```

Changes that you make to the files are rendered automatically, and the static folder
that is being rendered is at the path `_site`. If you make any changes to the `_config.yml`,
however, you would need to hit Control+c to kill the process, and start it again.

## Adding Content
Content could be a tutorial, a video, example code, or a combination of any of those
things. All content is organized as subfolders of the `_posts` folder of the repository.
For example, if we wanted to find examples, videos, and/or lessons about linux, we
would find them here! 

```bash
ls _posts
    linux/
```

The first step you should do, in addition to deciding what folder to use above,
is to decide if your contribution fits into a set. A set is a grouping of lessons
that the user will be suggested to read when in context of similar posts. The
lessons are defined by the markdown files in the `_sets` folder, and you are free to choose
one of these lessons, or generate a new file there to define a new one. For example,
here we see sets defined for getting-started and clusters, along with some defaults:

```bash
$ ls _sets/
clusters.md  _defaults.md  getting-started.md
```

Each set is just a name and a description! Take a look:

```bash
cat _sets/clusters.md
---
title: Cluster Computing
description: This series guides you through getting started with HPC cluster computing.
---
```

To add a post to this set, you would simply add some metadata to the post's front 
matter, like this:

```bash
set: clusters
set_order: 1
```

### 1. Create a New File
A post is just a markdown file. It has two sections, a header "front matter"
section, and then the content:

```
---
layout: post
title:  "Welcome to Jekyll!"
date:   2015-11-17 16:16:01 -0600
categories: jekyll update
---

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `bundle exec jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.md` and includes the necessary front matter.
```

### 2. Customize Front Matter
The front matter is the header at the top of the file, and it's important to categorize the 
lesson on the site. Let's take a look at an example for a basic Document to teach
someone to do something

```
---
date: 2018-05-06
title: Contributing
description: How to Contribute
type: Document
categories:
  - contribute
tags: [contribute]
---

Here is the content!
```
The "tags" should be one or more tags (single words) that describe what you are writing about. These
are the groups on the front page, so try to be conservative in choosing a tag, and having preference
for those that already exist. For example, if I am writing a post about Singularity containers, instead of
a tag of "singularity" I would ascribe the tag "containers." The reason is because if the user is looking
for Singularity, he or she can search the site and find the post easily.

You can read more about [posts here](https://jekyllrb.com/docs/posts/) or just copy
an already existing post (and rename the date and title of the file, and other metadata
in the font matter.

### 2. Adding Resources
It's often the case that a post will serve as a portal to direct the user to other
documentation bases. We've made that easy to do! Just add a resources list to the 
front matter:

```
resources:
  - name: Documentation
    link: https://singularityhub.github.io/interface
  - name: Globus
    link: https://globus.org
  - name: Singularity
    link: https://singularityware.github.io
  - name: Github
    link: https://github.com/singularityhub/interface
```

This will render into a nice list to go with the post. In fact, entire pages can
just be sets of resources for your user! Sometimes this is all that is needed.

You can also link to images directly in the post, using just standard markdown for images, or
if you need a diagram, try creating an <a href="http://asciiflow.com/" target="_blank">asciiflow</a>
diagram and embedding it in code blocks.

### 3. Adding Videos
You can embed a video directly in the post! Just specify that the type is "Video"
instead of "Document," and add the YouTube Video ID and the site will do it's magic.

```
type: Video
video_id: va5eAZ0p0no
```

### 4. Adding Supplementary Files
Supplementary Content can come in the way of example scripts, external documentation
or files (e.g., PDF) or even jupyter notebooks (and generally files that render 
on Github). To add supplementary files, it's just a matter of adding them to the
repository to live alongside the content, and then linking to it. We have 
a site variable that will make it easy to generate this link. For example, if I add
this file:

```bash
ls _posts
    linux/
        examples/
            find-examples.sh
```

in the template I can generate a path to the file (on Github) with:

{% raw %}
```
{{ site.github_posts }}/linux/examples/find-examples.sh
```
{% endraw %}

If I want the user to be able to download the raw file with something like wget,
I can do:

{% raw %}
```
wget {{ site.github_posts_raw }}/linux/examples/find-examples.sh
```
{% endraw %}
The organization is up to you! We will have a nice method to render and show
other kinds of rendered pages (e.g., notebooks) soon.
