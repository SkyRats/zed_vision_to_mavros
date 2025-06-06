# syntax=docker/dockerfile:1
FROM osrf/ros:humble-desktop-full AS main-setup

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install --no-install-recommends -y \
  lsb-release \
  sudo \
  wget \
  software-properties-common \
  build-essential  \
  ccache \
  g++ \
  gdb \
  gawk \
  git \
  tmux \
  vim \
  make \
  cmake \
  ninja-build \
  libtool \
  libxml2-dev \
  libxml2-utils \
  libxslt1-dev \
  python3-numpy \
  python3-pyparsing \
  python3-serial \
  python-is-python3 \
  libpython3-stdlib \
  libtool-bin \
  zip \
  unzip \
  default-jre \
  socat \
  ros-dev-tools \
  ros-humble-launch-pytest \
  && apt-get clean \
  && apt-get -y autoremove \
  && apt-get autoclean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# TAKEN from https://github.com/docker-library/python/blob/a58630aef106c8efd710011c6a2a0a1d551319a0/3.11/bullseye/Dockerfile
# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION=23.1.2
# https://github.com/docker-library/python/issues/365
ENV PYTHON_SETUPTOOLS_VERSION=65.5.1
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL=https://github.com/pypa/get-pip/raw/9af82b715db434abb94a0a6f3569f43e72157346/public/get-pip.py
ENV PYTHON_GET_PIP_SHA256=45a2bb8bf2bb5eff16fdd00faef6f29731831c7c59bd9fc2bf1f3bed511ff1fe

RUN set -eux; \
  \
  wget -O get-pip.py "$PYTHON_GET_PIP_URL"; \
  echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum -c -; \
  \
  export PYTHONDONTWRITEBYTECODE=1; \
  \
  python get-pip.py \
  --disable-pip-version-check \
  --no-cache-dir \
  --no-compile \
  "pip==$PYTHON_PIP_VERSION" \
  "setuptools==$PYTHON_SETUPTOOLS_VERSION" \
  ; \
  rm -f get-pip.py; \
  \
  pip --version

RUN python -m pip install --no-cache-dir -U future lxml pexpect flake8 empy==3.3.4 pyelftools tabulate pymavlink pre-commit junitparser

FROM osrf/ros:humble-simulation AS gazebo-harmonic

RUN apt-get update && apt-get install -y curl lsb-release gnupg
RUN curl https://packages.osrfoundation.org/gazebo.gpg --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null
RUN apt-get update && apt-get install -y gz-harmonic

FROM eclipse-temurin:19-jdk-jammy AS dds-gen-builder

RUN apt-get update && apt-get install --no-install-recommends -y \
  git \
  && apt-get clean \
  && apt-get -y autoremove \
  && apt-get autoclean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN git clone -b master --recurse-submodules https://github.com/ArduPilot/Micro-XRCE-DDS-Gen.git --depth 1 --no-single-branch --branch master dds-gen \
  && cd dds-gen \
  && ./gradlew assemble

FROM main-setup

WORKDIR /dds-gen
COPY --from=dds-gen-builder /dds-gen/scripts scripts/
COPY --from=dds-gen-builder /dds-gen/share share/
WORKDIR /

# Get STM32 GCC10 toolchain
ARG ARM_ROOT="gcc-arm-none-eabi-10"
ARG ARM_ROOT_EXT="-2020-q4-major"
ARG ARM_TARBALL="$ARM_ROOT$ARM_ROOT_EXT-x86_64-linux.tar.bz2"
ARG ARM_TARBALL_URL="https://firmware.ardupilot.org/Tools/STM32-tools/$ARM_TARBALL"

RUN cd /opt \
  && wget -qO- "$ARM_TARBALL_URL" | tar jx \
  && mv "/opt/$ARM_ROOT$ARM_ROOT_EXT" "/opt/$ARM_ROOT" \
  && rm -rf "/opt/$ARM_ROOT/share/doc"

# manual ccache setup for arm-none-eabi-g++/arm-none-eabi-gcc
RUN ln -s /usr/bin/ccache /usr/lib/ccache/arm-none-eabi-g++ \
  && ln -s /usr/bin/ccache /usr/lib/ccache/arm-none-eabi-gcc

# Set STM32 toolchain to the PATH
ENV PATH="/opt/$ARM_ROOT/bin:$PATH"

RUN mkdir -p $HOME/arm-gcc \
  && ln -s -f /opt/gcc-arm-none-eabi-10/ g++-10.2.1


ENV PATH="/dds-gen/scripts:$PATH"
# Set ccache to the PATH
ENV PATH="/usr/lib/ccache:$PATH"

# Gain some time by disabling mavnative
ENV DISABLE_MAVNATIVE=True

# Set the buildlogs directory into /tmp as other directory aren't accessible
ENV BUILDLOGS=/tmp/buildlogs

ENV TZ=UTC

RUN useradd -rm -d /home/zed -s /bin/bash -g root -G sudo -u 1000 zed -p "$(openssl 1234 -1 zed)"
USER zed
WORKDIR /home/zed

RUN echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc

RUN sudo apt update && sudo apt upgrade -y

RUN sudo apt-get install ros-humble-mavros ros-humble-mavros-extras -y 
RUN wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
RUN chmod a+x install_geographiclib_datasets.sh
RUN sudo ./install_geographiclib_datasets.sh

RUN git clone --recurse-submodules https://github.com/ArduPilot/ardupilot.git 
RUN cd ardupilot/
RUN perl -pi -e 's/\r\n|\n|\r/\n/g' ./Tools/environment_install/install-prereqs-ubuntu.sh -y 
RUN . ~/.profile
RUN pip3 install mavproxy
RUN perl -pi -e 's/\r\n|\n|\r/\n/g' ./waf configure --board sitl
RUN perl -pi -e 's/\r\n|\n|\r/\n/g' ./waf copter

RUN echo "export PATH=$PATH:$HOME/ardupilot/Tools/autotest" >> ~/.bashrc
RUN echo "export PATH=/usr/lib/ccache:$PATH" >> ~/.bashrc
RUN echo "alias sa='sim_vehicle.py -v ArduCopter'" >> ~/.bashrc
RUN echo "alias sag='sim_vehicle.py -v ArduCopter -f gazebo-iris'" >> ~/.bashrc
