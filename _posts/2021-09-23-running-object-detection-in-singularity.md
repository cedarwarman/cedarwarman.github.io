---
layout: post
title: "Running the Tensorflow Object Detection API in a Singularity container"
description:
author: Cedar Warman
date: 2021-09-23
hero_image: /img/blog/2021-09-22_singularity_tensorflow.jpg
image: /img/blog/2021-09-22_singularity_tensorflow.jpg
tags: singularity object-detection tensorflow hpc
canonical_url: https://www.cedarwarman.com/2020/09/23/running-object-detection-in-singularity.html
---

TL;DR <a href ="https://github.com/cedarwarman/object_detection_singularity/">Here’s my Singularity Definition File for the Tensorflow Object Detection API (including X11 forwarding)</a>.

I love the <a href ="https://github.com/tensorflow/models/tree/master/research/object_detection">Tensorflow Object Detection API</a>. It sits on top of Tensorflow and makes powerful object detection models like <a href="https://arxiv.org/abs/1512.03385">ResNet</a>, <a href="https://arxiv.org/abs/1904.07850">CenterNet</a>, and <a  href="https://arxiv.org/abs/1911.09070">EfficientDet</a> (relatively) easy to implement. In the past, I’ve used the API to <a href="https://doi.org/10.1111/tpj.15166">detect fluorescent kernels on maize ears</a>. Unfortunately, setting up the Object Detection API for that project was a pain. I had to manually install the right versions of CUDA/Tensorflow on my cluster and I still worry that the <a href="https://github.com/fowler-lab-osu/EarVision">code</a> is difficult for other people (and maybe even for me) to run. To solve these problems, I’m developing a new object detection pipeline in a containerized environment. Using a container should reduce the effort needed to run the object detection code, make the methods more reproducible, and make it portable between the cluster and the cloud. In addition to setting up the basic container, I’m including the configurations I used to allow X11 forwarding from the cluster. This allows me to do things like view images and to complete the <a href="https://github.com/tensorflow/models/blob/master/research/object_detection/colab_tutorials/eager_few_shot_od_training_tf2_colab.ipynb">example code</a> included with the Object Detection API.

Conveniently, the Object Detection API comes with a <a href="https://github.com/tensorflow/models/tree/master/research/object_detection/dockerfiles/tf2">Dockerfile</a> which can be used to build a Docker container. Inconveniently, Docker can’t be run on most university clusters, including the <a href="https://public.confluence.arizona.edu/display/UAHPC/Containers">cluster</a> here at the University of Arizona. Singularity solves this problem: it doesn’t require Docker’s extensive permissions and it interacts better with the shared environment. Here I’ll describe the method I used to convert the Object Detection API Dockerfile into a working Singularity container.

## Object Detection API Dockerfile

First, take a look at the <a href="https://github.com/tensorflow/models/blob/master/research/object_detection/dockerfiles/tf2/Dockerfile">Dockerfile</a> included in the Object Detection API:

```bash
FROM tensorflow/tensorflow:2.2.0-gpu

ARG DEBIAN_FRONTEND=noninteractive

# Install apt dependencies
RUN apt-get update && apt-get install -y \
    git \
    gpg-agent \
    python3-cairocffi \
    protobuf-compiler \
    python3-pil \
    python3-lxml \
    python3-tk \
    wget

# Install gcloud and gsutil commands
# https://cloud.google.com/sdk/docs/quickstart-debian-ubuntu
RUN export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)" && \
    echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
    apt-get update -y && apt-get install google-cloud-sdk -y

# Add new user to avoid running as root
RUN useradd -ms /bin/bash tensorflow
USER tensorflow
WORKDIR /home/tensorflow

# Copy this version of of the model garden into the image
COPY --chown=tensorflow . /home/tensorflow/models

# Compile protobuf configs
RUN (cd /home/tensorflow/models/research/ && protoc object_detection/protos/*.proto --python_out=.)
WORKDIR /home/tensorflow/models/research/

RUN cp object_detection/packages/tf2/setup.py ./
ENV PATH="/home/tensorflow/.local/bin:${PATH}"

RUN python -m pip install -U pip
RUN python -m pip install .

ENV TF_CPP_MIN_LOG_LEVEL 3
```

It seems pretty straightforward: it uses the tensorflow:2.2.0-gpu container as a foundation, installs some dependencies with apt-get, adds a new user to avoid running as root, then copies the Object Detection API into the image and installs it. I first tried using the conversion utility from <a href="https://singularityhub.github.io/singularity-cli/recipes">Singularity Python</a> to convert the Dockerfile to a Singularity Definition File. While I ended up converting everything by hand, the utility gave me a good first glance of how the different parts of the build files compared between the two platforms. I’ll go through my changes, then share the completed Definition File below.

## Singularity Definition File

The first line of a Singularity <a href="https://sylabs.io/guides/3.5/user-guide/definition_files.html">Definition File</a> defines the bootstrap agent. This tells Singularity how the container will be constructed. Here we tell it to use an image hosted on Docker Hub, then give it the address of the Docker Hub image in the second line (note that we’ve switched to Tensorflow 2.6.0 for compatibility with some of the other packages).

```bash
Bootstrap: docker
From: tensorflow/tensorflow:2.6.0-gpu
```

Next is the `%post` section. This is where I download and install the required packages. It starts by telling the operating system to install itself without requiring any input from the user.

```bash
%post
    export DEBIAN_FRONTEND=noninteractive
```

It then installs all the dependencies that require the Debian `apt` package manager. Note that I added x11-apps and xauth to the dependencies. These will enable X11 forwarding when we run the container on the cluster.

```bash
    apt-get update && apt-get install -y \
    git \
    gpg-agent \
    python3-cairocffi \
    protobuf-compiler \
    python3-pil \
    python3-lxml \
    python3-tk \
    wget \
    x11-apps \
    xauth
```

Now comes the download and installation of the Object Detection API (still in the `%post` section). First the script downloads gcloud and gsutil, associated utilities that are required for some parts of the API. Next, the script clones the API repository into the container using git. It then compiles the configuration for the installation of the API using <a href="https://developers.google.com/protocol-buffers">protocol buffer</a> (protobuf) files. These files are an efficient way to store data, similar to XML. They can be used for lots of things, but in this case they store the instructions for the installation of the API, including a list of Python packages to be installed with `pip`. 

```bash
    export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)" && \
    echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
    apt-get update -y && apt-get install google-cloud-sdk -y

    # Download the git version of the model garden
    cd /opt
    git clone https://github.com/tensorflow/models.git

    # Compile protobuf configs
    (cd /opt/models/research/ && protoc object_detection/protos/*.proto --python_out=.)
    cd /opt/models/research/

    cp object_detection/packages/tf2/setup.py ./
    python -m pip install -U pip	
    python -m pip install .
```

In the next part of the `%post` section, I install three Python packages that are required for the <a href="https://github.com/tensorflow/models/blob/master/research/object_detection/colab_tutorials/eager_few_shot_od_training_tf2_colab.ipynb">API tutorial</a> but are not included in the standard installation. If you are making your own Singularity Definition File, you can add any additional `apt` or Python packages that you require by adding them to the relevant sections of `%post`.

```bash
    python -m pip install imageio ipython google-colab
```

Finally, I download the model checkpoint for the tutorial and make sure the permissions allow it to be accessed.

```bash
    # Downloading the model checkpoint for the demo (optional if not running demo)
    wget http://download.tensorflow.org/models/object_detection/tf2/20200711/ssd_resnet50_v1_fpn_640x640_coco17_tpu-8.tar.gz
    tar -xf ssd_resnet50_v1_fpn_640x640_coco17_tpu-8.tar.gz
    mv ssd_resnet50_v1_fpn_640x640_coco17_tpu-8/checkpoint /opt/models/research/object_detection/test_data/
	
    # Fix permissions problem
    chmod -R 777 /opt/models/research/object_detection/test_data/checkpoint
```

The next section of the Definition File is called `%environment`. This is where you put environmental variables that will be set when you run the container. The original Dockerfile has `TF_CPP_MIN_LOG_LEVEL` set to `3`, so I’ll include it here as well. This Tensorflow option suppresses log outputs. `0` logs all messages, `1` omits `INFO`, `2` omits `INFO` and `WARNING`, and `3` omits `INFO`, `WARNING`, and `ERROR`. Since I want to see error messages for troubleshooting, I’ll comment out this line.

```bash
%environment
    # Makes Tensorflow warnings quiet (optional)
    # export TF_CPP_MIN_LOG_LEVEL=3
```

The following two sections are `%runscript` and `%startscript`. These sections write to a file that’s executed when the container is run and at container build time, respectively. `%startscript` is designed to run when the container is run as an instance, while `%runscript` is executed when the container is run with the `singularity run` command. In this case, I’ll include the same two commands for both sections: first changing the directory to the base directory of the script and second executing any arguments that were included when the container was run.

```bash
%runscript
    cd /opt/models/research/ && \
    exec /bin/bash "$@"

%startscript
    cd /opt/models/research/ && \
    exec /bin/bash "$@"
```

And finally, the help section describes the purpose of the container.

```bash
%help
    This container runs the Tensorflow 2 Object Detection API. It was modified for Singularity from the Dockerfile contained in the Object Detection API repository: https://github.com/tensorflow/models/blob/master/research/object_detection/dockerfiles/tf2/Dockerfile
```

## Building the container

Now that we have a Singularity Definition File, we can build the container. On the University of Arizona cluster, users don’t have the root authority required to build containers. However, Singularity has the option to use <a href="https://cloud.sylabs.io/builder">Remote Builder</a> to get around root requirements by building the container in the cloud. After creating a Remote Builder account, set up the authentication token by running `singularity remote login`. Now you can build the container using the following command (from the base directory of the <a href="https://github.com/cedarwarman/object_detection_singularity">repository</a>):

```bash
singularity -v build --remote ./singularity/tf_od.sif ./singularity/Singularity
```

The build process takes about 15 minutes and results in a SIF formatted container.

## Running the container with X11 forwarding

To run the demo, first sign in to your HPC environment with `ssh` and option `-X` to enable X11 forwarding. At the University of Arizona, once you have signed in to the bastion host, you must also use the `-X` option when signing into the login node, e.g. `shell -X`. You can then run the demo from an interactive GPU node or using a Slurm script.

```bash
singularity exec --nv -B ~/.Xauthority ./singularity/tf_od.sif python3 ./python/eager_few_shot_od_training_tf2_singularity.py &>/dev/null &
```

The above command runs the demo using the newly created Singularity container. The `--nv` option properly configures the container for Nvidia GPUs, while the `-B` option adds a file to the container that's  necessary for X11 forwarding. Here I’ve silenced output messages with `&>/dev/null` and put the process in the background with the final `&`. With the process in the background you can monitor GPU activity with `nvtop` or `nvidia-smi`. The demo displays images at several steps through X11 and finally outputs images with bounding boxes and an animated gif. The training and inference took less than five minutes using an Nvidia V100S GPU.

## Conclusion

Hopefully this guide will help you run the Object Detection API in a Singularity container. While all clusters are unique, my goal is to at least provide a starting point from running containerized object detection. If you have any questions please feel free to reach out!

## <a href="https://github.com/cedarwarman/object_detection_singularity"><strong>Repository containing all code described in this post.</strong></a>

