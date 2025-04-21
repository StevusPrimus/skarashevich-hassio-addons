# syntax = docker/dockerfile:1.4
ARG BUILD_FROM BUILD_ARCH
ARG CPAI_VERSION_AARCH64 CPAI_VERSION_AMD64
ARG CPAI_INSTALLER_URL_AARCH64 CPAI_INSTALLER_URL_AMD64


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

# It's important to pull the correct base for the target architecture, otherwise
# building this image on, say, an Arm64 machine will result in the base image
# being Arm64, not x86_64 as assumed.
FROM ${BUILD_FROM} AS base

RUN     if [ "${BUILD_ARCH}" = "aarch64" ]; then \
                export INSTALLER_URL="${CPAI_INSTALLER_URL_AARCH64}"; \
        elif [ "${BUILD_ARCH}" = "amd64" ]; then \
                export INSTALLER_URL="${CPAI_INSTALLER_URL_AMD64}"; \
        fi \
        && echo "Installer URL: ${INSTALLER_URL}"

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

ENV LANGUAGE=C
ENV LC_ALL=C
ENV LANG=C
ENV LC_CTYPE=C

# Setup the ports -------------------------------------------------------------
EXPOSE 5000
EXPOSE 32168/tcp
EXPOSE 32168/udp

RUN apt-get update  -y && \
        apt-get upgrade -y --no-install-recommends apt-utils && \
        apt-get install -y --no-install-recommends \
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
        gcc             \
        python3-dev     \
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
RUN apt-get install software-properties-common -y 
RUN add-apt-repository ppa:dotnet/backports -y   
RUN apt-get update
RUN apt-get install -y aspnetcore-runtime-9.0

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


RUN mkdir -p /install
WORKDIR /install
RUN \
        set -x \
        && if [ "${BUILD_ARCH}" = "aarch64" ]; then \
                        export INSTALLER_URL=${CPAI_INSTALLER_URL_AARCH64}; \
                elif [ "${BUILD_ARCH}" = "amd64" ]; then \
                        export INSTALLER_URL=${CPAI_INSTALLER_URL_AMD64}; \
                fi \
        \
        && wget ${INSTALLER_URL}
RUN \
        set -x \
        && if [ "${BUILD_ARCH}" = "aarch64" ]; then \
                export FILE_NAME=${CPAI_FILE_NAME_AARCH64}; \
        elif [ "${BUILD_ARCH}" = "amd64" ]; then \
                export FILE_NAME=${CPAI_FILE_NAME_AMD64}; \
        fi \
        \ 
        && unzip ${FILE_NAME}.zip
RUN \
        set -x \
        && if [ "${BUILD_ARCH}" = "aarch64" ]; then \
                export FILE_NAME=${CPAI_FILE_NAME_AARCH64}; \
        elif [ "${BUILD_ARCH}" = "amd64" ]; then \
                export FILE_NAME=${CPAI_FILE_NAME_AMD64}; \
        fi \
        \ 
        && dpkg -i ${FILE_NAME}.deb
RUN rm -rf /install

WORKDIR /

RUN \
        set -x \
        && if [ "${BUILD_ARCH}" = "aarch64" ]; then \
                export CPAI_VERSION=${CPAI_VERSION_AARCH64}; \
        elif [ "${BUILD_ARCH}" = "amd64" ]; then \
                export CPAI_VERSION=${CPAI_VERSION_AMD64}; \
        fi \
        && \
        mkdir -p /app \
        && \
        ln -s /usr/bin/codeproject.ai-server-${CPAI_VERSION} /app/server 

RUN pushd "/app/server" && bash ../setup.sh && popd

COPY rootfs /

WORKDIR /app/server
