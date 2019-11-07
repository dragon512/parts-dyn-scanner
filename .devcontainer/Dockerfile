#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------
FROM fedora:30

# This Dockerfile adds a non-root 'vscode' user with sudo access. However, for Linux,
# this user's GID/UID must match your local user UID/GID to avoid permission issues
# with bind mounts. Update USER_UID / USER_GID if yours is not 1000. See
# https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Configure apt and install packages
RUN dnf update -y
# \
    #
    # Verify git, process tools, lsb-release (common in install instructions for CLIs) installed
    #&&
RUN dnf -y install git python3 make gcc gcc-c++ rpm rpm-build cmake expat-devel docbook-style-xsl libxslt-devel autoconf automake libtool bison flex libcap-devel pcre-devel hwloc-devel ncurses-devel zlib-devel c-ares-devel perl-ExtUtils-MakeMaker perl-Pod-Html json-c-devel readline-devel patchelf \
    #
    # Create a non-root user to use if preferred - see https://aka.ms/vscode-remote/containers/non-root-user.
    && groupadd --gid $USER_GID $USERNAME \
    && useradd -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    # [Optional] Add sudo support for the non-root user
    && dnf install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME\
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    #
    # Clean up
    # && apt-get autoremove -y \
    && dnf clean all
    # && rm -rf /var/lib/apt/lists/*
    # add pip setup in user directory
# Set user to be used for the rest of the command
USER $USERNAME

# to deal with python 3 unicode issues
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV PATH=/home/$USERNAME/myenv/bin:$PATH
# set up a virtual environment to avoid issues with adding non-system pip packages
RUN python3 -m venv /home/$USERNAME/myenv \
    && pip install pip --upgrade \
    && pip install scons-parts\
    # set git to not use /r/n
    && git config --global core.autocrlf false

