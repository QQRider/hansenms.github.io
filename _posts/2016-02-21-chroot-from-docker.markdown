---
layout: post
title:  "chroot jail from Docker image"
date:   2016-02-21 19:34:46 -0500
categories: DevOps Tips
---

So you like [Docker](https://www.docker.com/) and have decided to make it an integral part of your application life-cycle? Me too. I have been keeping an eye on Docker for a while and finally decided that it was mature enough to take a closer look at as a deployment strategy for the [Gadgetron](https://github.com/gadgetron/gadgetron). The Gadgetron has a ton of dependencies. Some of them have to be compiled with the right options to get the best performance. It is not complicated, but it is tedious and containerization makes a lot of sense. In the past we have used `chroot` images to pack all the dependencies, etc. Docker is sort of a fancy chroot jail, but it has a lot of additional bells and whistles. A simple example is environment variables that need to be set correctly for an application to run. We have been managing this with a set of scripts for starting the Gadgetron in the chroot image, but it is much more elegant to have such settings associated with the image. Docker provides that and many other things for you.

Now we have taken the step and are build system is automatically generating Docker images. You can find them at [https://hub.docker.com/r/gadgetron](https://hub.docker.com/r/gadgetron). It is great. But from time to time, we still need to deploy on platforms where Docker engine is not available and cannot be installed. An example would be a reconstruction box delivered by a medical device manufacturer. It runs 64-bit Linux, so we are in good shape, but we cannot install Docker. So we need a way to turn the nice Docker image into a chroot image.

Fortunately it is easy. Let's say that you wanted to make a chroot image out of the Docker image [gadgetron/ubuntu_1404_cuda75](https://hub.docker.com/r/gadgetron/ubuntu_1404_cuda75/), it can actually be done with a single command:

    docker run -ti --rm --volume=`pwd`:/opt/backup gadgetron/ubuntu_1404_cuda75 tar -cvpzf /opt/backup/chroot.tar.gx --exclude=/opt/backup --one-file-system /

And the file `chroot.tar.gz` will contain the entire file system (minus the virtual file systems). Of course when you run something in this chroot jail, you still need to worry about setting environment variables, etc. But at least it is possible to deploy on legacy systems where you can't install software.

One final note about using CUDA in chroot jails. With Docker it is relatively easy with tools such as [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) but with the chroot jail you have to manage a few things yourself. Specifically you have to mount `/proc`, `/dev` and `/sys` inside the chroot before starting it. You also need to make sure that the chroot has the same versions of files such as `libcuda.so`, `nvidia-smi`, etc. as the host file system. An earlier version of `nvidia-docker` was actually a shell script that took care of all of that. We made a script heavily inspired by that early `nvidia-docker` script, which copies all the required files into the chroot image before you run it. You can find it [here](https://github.com/gadgetron/gadgetron/blob/master/chroot/nvidia-copy.sh).

Good luck. Let me know if you have comments/suggestions. 

