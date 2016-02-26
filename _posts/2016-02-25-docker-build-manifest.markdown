---
layout: post
title:  "Creating a Docker Build Manifest"
date:   2016-02-25 20:45:00 -0500
categories: DevOps Tips
---

You may know I like [Docker](https://www.docker.com/). It has really made a big difference in how we think about continuous integration testing and deployment for our [Gadgetron](https://github.com/gadgetron/gadgetron) project. In fact, we now think about that as the same thing. We build images to test the software and when we have tested, we can deploy the actual image that we just tested. It is great.

However, one issues that I have had is that we would like a simple way of figuring out which versions of the Gadgetron itself and all the dependencies that a given image contains. You can see me going on about this at some length in this [github issue](https://github.com/docker/docker/issues/13178#issuecomment-184023295). The issue is that a given build (image) of our application will have a build of the code itself but also all the dependencies. Some of these dependencies are also cloned from repositories and built in the image. It would be nice to capture all the SHA1 hashes of all the git repos that go into a build. This information is useful if we discover a bug and we need to figure out what changed. Was it our code or was it a dependency?

It is possible to add `LABEL`s to images when they are built. But these `LABEL`s cannot take values that are based on a command. Specifically, what we would like to do is to have something like this in a `Dockerfile`:

    LABEL com.mycompany.myproduct.sha1=$(git rev-parse HEAD)

But that is not allowed. The labels have to either be specified directly in the file (they can not be the result of a command executed at build time) or they can be based on `--build-arg` passed in with the `docker build` command. But that is problematic. We would have to check all the code bases out locally and pass in the SHA1 hashes as arguments, but even that is not good enough, since the code bases may change between the time we check the code out locally and the time that it is cloned during a build. It would of course be possible to add the code to the build context to get around that, but that approach would not work if we are using a remote build system such as [https://hub.docker.com/](https://hub.docker.com/).

So what to do? The simply solution is to record SHA1 hashes and other meta data in files that can then be inspected. So for instance, we could build the [Gadgetron](https://github.com/gadgetron/gadgetron) framework like this:

    RUN cd /opt/code && \
        git clone https://github.com/gadgetron/gadgetron.git && \
        cd gadgetron && \
        mkdir build && \
        cd build && \
        cmake ../ && \
        make -j $(nproc) && \
        make install && \
        git rev-parse HEAD >> /opt/code/gadgetron_sha1.txt

And the SHA1 hash of that library would be accessible by running a command like:

    docker run --rm -ti <imagename> cat /opt/code/gadgetron_sha1.txt

So far so good. The problem is that it gets a bit hard to manage if there are many libraries and we have to look for a bunch of files all over the file system to do diagnostics. Instead it would be nice to build a "manifest" in some format that is easy to parse, e.g., JSON. My solution has been to create a small script which uses [`jq`](https://stedolan.github.io/jq/) to add values and query values in this manifest. The script looks something like this:

{% highlight bash %}
#!/bin/bash

function usage
{
    echo "usage manifest --key <com.mykey.blah> [--value <VALUE>] [--file <FILE>]"
}

filename=/opt/manifest.json
key="."
value=""

while [ "$1" != "" ]; do
    case $1 in
        -f | --file )           shift
                                filename=$1
                                ;;
        -k | --key )            shift
                                key=$1
                                ;;
        -v | --value )          shift
                                value=$1
                                ;;
        -h | --help )           usage
                                exit
                                ;;
        * )                     usage
                                exit 1
    esac
    shift
done

if [ ! -f $filename ]; then
    echo "{}" > $filename
fi

if [ -z "$value" ]; then
    #This is a query
    jq $key $filename
else
    #This is an assignment
    out=`jq "$key=\"$value\"" $filename`
    echo $out > $filename
fi
{% endhighlight %}


The example build command above would then be:

    RUN cd /opt/code && \
        git clone https://github.com/gadgetron/gadgetron.git && \
        cd gadgetron && \
        mkdir build && \
        cd build && \
        cmake ../ && \
        make -j $(nproc) && \
        make install && \
        /opt/code/gadgetron/docker/manifest --key .io.gadgetron.gadgetron.sha1 --value `git rev-parse HEAD` 

Obviously the `manifest` command (the script above) would need to be supplied with the full path (`/opt/code/gadgetron/docker/manifest`) or it would need to be in the PATH at this point, but that is easy to ensure. Each library dependency that we build, we can add to the manifest. It is then easy to subsequently get the whole manifest with:

    docker run --rm <imagename> /opt/code/gadgetron/manifest

Which would give you the entire manifest or you can of course query a specific key. If you get the entire manifest, specific keys can of course be pulled with `jq`. In our case, a typical manifest would look like:

{% highlight json %}
{
    "io": {
        "gadgetron": {
            "siemens_to_ismrmrd": {
                "sha1": "ec7e5d3948a4bb6318d3e2257bcd2c671a6e4a0f"
            },
            "ismrmrdpython": {
                "sha1": "c38cd60c3a69416374c276b7820b0ffd70a6c4e0"
            },
            "gadgetron": {
                "sha1": "2903a58a356e04d42e37c780220c0c69625f241a"
            },
            "ismrmrd": {
                "sha1": "69c6c15d032eb4d7dfcbaf88dd7b6478b1c9e72b"
            },
            "ismrmrdpythontools": {
                "sha1": "7a77ad422b09f74075069a2bb919f8f6e23ad9b4"
            },
            "philips_to_ismrmrd": {
                "sha1": "606e25b64f4bf84e8b991960a7667d166481ca5f"
            }
        }
    }
}
{% endhighlight %}

We use this manifest to tag each image with some of these hashes. An example of this is the [`tag_image_with_version.sh`](https://github.com/gadgetron/gadgetron/blob/master/docker/tag_image_with_version.sh) script:
{% highlight bash %}
#!/bin/bash

image_name=$1

gadgetron_version=$(docker run --rm $image_name gadgetron_info | awk '/-- Version/ {print $4}')
manifest=$(docker run --rm $image_name /opt/code/gadgetron/docker/manifest)
gadgetron_sha1=$(echo $manifest|jq '.io.gadgetron.gadgetron.sha1'| tr -d '"'|cut -c-8)
gtprep_sha1=$(echo $manifest|jq '.io.gadgetron.gtprep.sha1'| tr -d '"' | cut -c-8 )

tag_value="${gadgetron_version}-${gadgetron_sha1}"

if [ "$gtprep_sha1" != "null" ]; then
    tag_value="${tag_value}-${gtprep_sha1}"
fi

docker tag ${image_name} ${image_name}:${tag_value}

{% endhighlight %}

This is clearly not a perfect solution. The preferable thing would of course be to have build time `LABEL`s, but until that day (if it every comes), we have something that sort of works and is not too, too ugly. Maybe a similar approach could be useful for your project. 

Good luck. Let me know if you have comments/suggestions. 

