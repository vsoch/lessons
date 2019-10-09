---
date: 2019-10-08
title: Rebasing in Practical Terms
video_id: 9F4RE2_yn6I
description: A Practical Guide to Git Rebasing
categories:
  - linux
resources:
  - name: To Squash or Not to Squash
    link: https://seesparkbox.com/foundry/to_squash_or_not_to_squash
  - name: Always Squash and Rebase your Git Commits
    link: https://blog.carbonfive.com/2017/08/28/always-squash-and-rebase-your-git-commits/
type: Video
tags: [linux]
---


Rebasing is really useful for creating a clean git history. It technically means
changing the original base commit of a branch, but for this post, I want to talk
about it in practical terms. If you want the technical details, there are
plenty of resources available to you. 
First, let's quickly answer some questions.

## When might I want to rebase?

Generally, when you are working on a branch and want to clean up your git history,
it's easy to rebase with master.

## What are the general steps?

While this isn't a copy paste and go example, it will walk you through general steps.
We generally start with checkout of a new branch from an updated master:

```bash
# checkout master
git checkout master

# make sure your upstream master is up to date
git pull origin master

# now checkout a new branch
git checkout -b add/cool-feature
```

And then we make an awesome change! And we commit it.

```bash
# make some changes here, and then commit
git commit -a -s -m "This is my awesome change"
```

Then we maybe push, and open up a pull request, and are told to make more changes.
Our commits get sloppy.

```bash
git push origin add/cool-feature

# opens pull request, request for changes

# each of the below is associated with some mistake/more changes
git commit -a -s -m 'making this other change'
git commit -a -s -m 'oops'
git commit -a -s -m 'oops'
git commit -a -s -m 'whyyyyyy'
```

Then our git history is a mess! So we do an interactive rebase against master.

```bash
git rebase -i master
```

And then change all of the crappy commits to "fixup" to squash and disregard the
squashed commit messages. And we push to the branch with force!

```bash
git push origin add/cool-feature --force
```

That's it! Happy rebasing, folks.
