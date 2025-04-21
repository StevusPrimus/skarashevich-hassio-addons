# syntax = docker/dockerfile:1.4
ARG BUILD_FROM
ARG BUILD_ARCH

# CodeProject.AI Server 3.0+
#
# Docker file for x86_64 CPU enabled image
#
#
# ASSUMPTION: This is being built from the root directory of the solution. This
# is set in the build_docker script as the last param to the docker buildx command,
# namely the "../..". This takes us from /Installers/Docker to /. and then this
# image file proceeds


# INITIAL SETUP ===============================================================

# Note: we're before the first 'FROM' statement so these values are not available
# inside the build stages. If you wish to use these inside a stage, re-grab them
# inside that stage
ARG UBUNTU_VERSION=22.04
ARG DOTNET_VERSION=9.0
ARG CPAI_VERSION=2.9.7

# It's important to pull the correct base for the target architecture, otherwise
# building this image on, say, an Arm64 machine will result in the base image
# being Arm64, not x86_64 as assumed.
FROM ${BUILD_FROM} AS base

WORKDIR /app

# Replace the sudo command, which doesn't exist in this image, with a noop so
# our scripts, which do contain sudo calls, will work
RUN if [ ! -x "$(command -v sudo)" ] && [ ! type sudo 2>/dev/null ]; then echo "#!/bin/sh\n\${@}" > /usr/sbin/sudo; chmod +x /usr/sbin/sudo; fi

# Environment vars ------------------------------------------------------------

ENV ASPNETCORE_URLS=http://+:32168;http://+:5000

# The simple log format is easier on my brain than json
ENV LOGGING__CONSOLE__FORMATTERNAME=simple

# Magic that was being done by the Microsoft ASP.NET base image that we aren't using anymore
ENV DOTNET_RUNNING_IN_CONTAINER=true

# noninteractive frontend means no prompts or questions are asked and whenever a call requires an
# answer, the default will be used. Installs will be non-interrupted and so won't hang. The Python
# installs, for instance, require this in this current environment
ENV DEBIAN_FRONTEND=noninteractive

# Grab the values passed in via command line here so we can use inside this stage
ARG UBUNTU_VERSION=22.04
ARG DOTNET_VERSION=9.0
ARG CPAI_VERSION

ENV CPAI_VERSION=$CPAI_VERSION
ENV UBUNTU_VERSION=$UBUNTU_VERSION
ENV DOTNET_VERSION=$DOTNET_VERSION

# This is so perl doesn't get upset
ENV LANGUAGE=C
ENV LC_ALL=C
ENV LANG=C
ENV LC_CTYPE=C

# Setup the ports -------------------------------------------------------------

EXPOSE 5000
EXPOSE 32168/tcp
EXPOSE 32168/udp

# Install required libraries --------------------------------------------------

# *** TODO: *** all of this is in /app/server/install.sh. This script should be
# called rather than having the same commands copy and pasted here.

# RUN apt-get update -y && apt-get upgrade -y --no-install-recommends apt-utils

# Install packages. In order:
#
# Required for SkiaSharp
#   libfontconfig1
# Required for System.Drawing
#   libgdplus
#   libjpeg-dev (maybe?)
#   zlib1g-dev (maybe?)
# Needed for opencv-python
#   ffmpeg libsm6 libxext6 libc6-dev
# So we can query glxinfo for GPU info
#   mesa-utils
# So we can install modules
#   curl jq unzip wget rsync ca-certificates (ca-certificates so we can do --no-check-certificate, jq for json parsing)
# So we can build the wheels of some modules (eg PaddleOCR)
#   patch
# This stops the "lsmod: not found" error
#   kmod
# This allows us to query process info
#   psmisc

RUN apt-get update -y && apt-get install -y --no-install-recommends \
    libfontconfig1  \
    libgdiplus      \
    libjpeg-dev     \
    zlib1g-dev      \
                    \
    ffmpeg          \
    libc6-dev       \
    libsm6          \
    libxext6        \
                    \
    mesa-utils      \
                    \
    ca-certificates \
    curl            \
    jq              \
    rsync           \
    unzip           \
    wget            \
                    \
    psmisc          \
                    \
    patch           \
                    \
    kmod            > /dev/null


# .NET -------------------------------------------------------------------------

# Update packages and install .NET
RUN apt install software-properties-common -y
RUN add-apt-repository ppa:dotnet/backports -y
RUN apt-get update && apt-get install aspnetcore-runtime-$DOTNET_VERSION -y > /dev/null


# Setup Python ----------------------------------------------------------------

# *** TODO: *** all of this is in /app/SDK/Scripts/utils.sh under "setupPython".
# We should use that method rather than having the same commands copy and pasted
# here. BEWARE that the list of apt installed here differs a little from what is
# installed via setupPython. installRequiredPythonPackages also installs some
# tools from the list below

# Combining all these together into a single layer
# 1. Add deadsnakes repo
# 2. Install pip for python3,  python3-apt and python3-setuptools
#    Note that this (currently) installs python3.8 as a dependency. It could be
#    in the future that a different version of python is installed instead, in
#    which case we may need to pin the installed python version.
# 3. Install Python 3.8 as well as dev and distutils so packages can be built
#    if needed, and venv for creating virtual environments
# 4. Using the now installed Python 3.8, upgrade the python setuptools and dev
#    packages to the latest
# 5. Install virtualenv virtualenvwrapper for python 3.8

RUN apt-get install software-properties-common -y
RUN add-apt-repository ppa:deadsnakes/ppa -y
RUN apt-get install -y --no-install-recommends            \
        python3-pip                                       \
        python3-apt                                       \
        python3-setuptools
RUN apt-get install -y --no-install-recommends            \
        python3.8
RUN apt-get install -y --no-install-recommends            \
        python3.8-distutils                               \
        python3.8-dev                                     \
        python3.8-venv
RUN python3.8 -m pip install --upgrade setuptools
RUN python3.8 -m pip install --upgrade pip
RUN python3.8 -m pip install virtualenv virtualenvwrapper > /dev/null

# Python 3.9: Combining all these together into a single layer, but leveraging
# the work done in Python 3.8:
# 1. Install Python 3.9 as well as dev and distutils so packages can be built
#    if needed, and venv for creating virtual environments
# 4. Using the now installed Python 3.8, upgrade the python setuptools and dev
#    packages to the latest
# 5. Install virtualenv virtualenvwrapper for python 3.8

RUN apt-get install -y --no-install-recommends            \
        python3.9
RUN apt-get install -y --no-install-recommends            \
        python3.9-distutils                               \
        python3.9-dev                                     \
        python3.9-venv
RUN python3.9 -m pip install --upgrade setuptools
RUN python3.9 -m pip install --upgrade pip
RUN python3.9 -m pip install virtualenv virtualenvwrapper > /dev/null

# Install Python3.11, distutils and dev tools (so packages can be built if needed)
# and pip install so we can install python packages. We don't need this for the
# built in modules, but we know we have downloadable modules that could use Python3.11
# so we preempt the need.
RUN apt-get install -y --no-install-recommends             \
        python3.11
RUN apt-get install -y --no-install-recommends             \
        python3.11-distutils                               \
        python3.11-dev                                     \
        python3.11-venv
RUN python3.11 -m pip install --upgrade setuptools
RUN python3.11 -m pip install --upgrade pip
RUN python3.11 -m pip install virtualenv virtualenvwrapper > /dev/null


ENV BUILD_ARCH = $BUILD_ARCH
# CREATE THE FINAL IMAGE ======================================================
RUN echo Build arch = $BUILD_ARCH

RUN \
        set -x \
        echo $BUILD_ARCH \
        if  [ "${BUILD_ARCH}" = "aarch64" ]; then \
                export ARCH_PREFIX="arm64-"; \
        else \
                export ARCH_PREFIX=${BUILD_ARCH}; \
        fi

FROM codeproject/ai-server:${BUILD_ARCH}2.9.7 AS build
FROM base AS final

# Note: UBUNTU_VERSION, DOTNET_VERSION and all ENV declared in 'base' are still here.

ARG REPO_NAME=CodeProject.AI-Server
ENV REPO_NAME=$REPO_NAME

# Move published server and modules into place --------------------------------


WORKDIR /app
COPY --from=build /app .

# Install required Python packages --------------------------------------------

# *** TODO: *** all of this is in /app/SDK/Scripts/utils.sh under
# "installRequiredPythonPackages". We should use that method rather than having
# the same commands copy and pasted here. NOTE that installRequiredPythonPackages
# installs some packages that, combined with the packages installed by setupPython,
# provide coverage for *almost* all the packages installed in this docker file.
# When switching to using our scripts we need to ensure we cover all installs and
# test.

# *** WARNING ON ABOVE NOTE *** To save space in the Docker containers we don't
# create venv's for pre-installed/hardcoded modules. We install these modules
# directly into the system's python installation folders. This means that if we
# choose to use the utils.sh script for setting up python then we'd need to take
# this into account

# See https://towardsdatascience.com/how-to-shrink-numpy-scipy-pandas-and-matplotlib-for-your-data-product-4ec8d7e86ee4
# for a discussion on reducing PIP install sizes. That article is woefully out of date, with the
# --compile and --global-option now deprecated, the CLFAGS to nothing, but the --no-cache-dir does help

RUN python3.8 -m pip --no-cache-dir install \
    -r "/app/SDK/Python/requirements.txt" \
    -r "/app/preinstalled-modules/FaceProcessing/requirements.linux.txt" \
    -r "/app/preinstalled-modules/ObjectDetectionYOLOv5-6.2/requirements.linux.txt"

RUN python3.9 -m pip --no-cache-dir install \
    -r /app/SDK/Python/requirements.txt

# Add folders for storing persisted user data and modules. This should be mapped
# to a folder on the host. Typically C:\ProgramData\CodeProject\AI on Windows,
# /etc/codeproject/ai on Linux, and /Library/Application Support/CodeProject/AI
# on macOS.
# We also make a .vscode folder so we have some bits in place if we wish to run
# VSCode inside the container
RUN mkdir --parents /etc/codeproject/ai

# For debug
# ENTRYPOINT ["tail", "-f", "/dev/null"]
ENV CONFIG_PATH=/data/options.json
LABEL \
  io.hass.version="2.9.7" \
  io.hass.type="addon" \
  io.hass.arch="aarch64|amd64"

WORKDIR /app/server