---
date: 2017-01-15
title: Transfer Singularity Containers
video_id: va5eAZ0p0no
description: Transfer Singularity containers using Globus
categories:
  - transfer
resources:
  - name: Documentation
    link: https://singularityhub.github.io/interface
  - name: Globus
    link: https://globus.org
  - name: Singularity
    link: https://singularityware.github.io
  - name: Github
    link: https://github.com/singularityhub/interface
type: Video
set: getting-started
set_order: 1
tags: [transfer]
---


Singularity containers allow us to build containers locally, and run them on
a shared resource, giving you the power to do things like install libraries
without needing to ask your cluster administrator. Since containers can be 
quite bit, manually transfer might get burdensome, and so Tunel can help.

"Tunel" is a Docker container that you run locally that can act as a "tunnel" 
between your local machine and any cluster with a Globus endpoint.
