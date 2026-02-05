FROM docker.io/rockylinux:9 AS bootstrap

ARG GPU_ARCH=gfx90a
ARG GPU_BACKEND_VERSION=6.4.3

ENV SPACK_ROOT=/opt/spack \
    CURRENTLY_BUILDING_DOCKER_IMAGE=1 \
    container=docker

RUN dnf update -y \
 && dnf install -y epel-release \
 && dnf update -y \
 && dnf --enablerepo epel install -y \
        bzip2 \
        cmake \
        curl-minimal \
        file \
        findutils \
        gcc-c++ \
        gcc \
        gcc-gfortran \
        git \
        gnupg2 \
        hg \
        hostname \
        iproute \
        make \
        patch \
        python3 \
        python3-pip \
        python3-setuptools \
        svn \
        unzip \
        xz \
        zstd \
 && pip3 install boto3 \
 && rm -rf /var/cache/dnf \
 && dnf clean all

# Install HIP  #
COPY ./envs/x86/rocm/rocm.repo /etc/yum.repos.d/rocm.repo
RUN sed -i "s/@GPU_BACKEND_VERSION@/${GPU_BACKEND_VERSION}/g" /etc/yum.repos.d/rocm.repo

RUN dnf clean all && \
    dnf update -y && \
    dnf install -y rocm-hip-sdk rocm-llvm rocm-smi-lib rocminfo

RUN ls -l /opt/rocm-${GPU_BACKEND_VERSION}/include/hip/hip_version.h

RUN mkdir $SPACK_ROOT && cd $SPACK_ROOT && \
    git init --quiet && git remote add origin https://github.com/spack/spack.git && git fetch --depth=1 origin develop && git checkout --detach FETCH_HEAD && \
    mkdir -p $SPACK_ROOT/opt/spack

RUN ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/docker-shell \
 && ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/interactive-shell \
 && ln -s $SPACK_ROOT/share/spack/docker/entrypoint.bash \
          /usr/local/bin/spack-env

RUN mkdir -p /root/.spack \
 && cp $SPACK_ROOT/share/spack/docker/modules.yaml \
        /root/.spack/modules.yaml \
 && rm -rf /root/*.* /run/nologin

# [WORKAROUND]
# https://superuser.com/questions/1241548/
#     xubuntu-16-04-ttyname-failed-inappropriate-ioctl-for-device#1253889
RUN [ -f ~/.profile ]                                               \
 && sed -i 's/mesg n/( tty -s \\&\\& mesg n || true )/g' ~/.profile \
 || true


WORKDIR /root
SHELL ["docker-shell"]

# Creates the package cache
RUN spack bootstrap now \
    && spack bootstrap status --optional \
    && spack spec hdf5+mpi

ENTRYPOINT ["/bin/bash", "/opt/spack/share/spack/docker/entrypoint.bash"]
CMD ["interactive-shell"]

# Build stage with Spack pre-installed and ready to be used
FROM bootstrap AS builder


# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir -p /opt/spack-environment && \
set -o noclobber \
&&  (echo spack: \
&&   echo '  specs:' \
&&   echo '  - feq-parse@2.2.2' \
&&   echo '  - openmpi@5.0.8 +rocm' \
&&   echo '  - hdf5@1.14.5 +fortran +mpi' \
&&   echo '  packages:' \
&&   echo '    all:' \
&&   echo '      require:' \
&&   echo '      - target=x86_64_v3' \
&&   echo '      prefer:' \
&&   echo "      - amdgpu_target=$GPU_ARCH" \
&&   echo '    cmake:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo '        - spec: "cmake@3.26.5"' \
&&   echo '          prefix: "/usr"' \
&&   echo '    rocm-smi-lib:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo "        - spec: \"rocm-smi-lib@${GPU_BACKEND_VERSION}\"" \
&&   echo "          prefix: \"/opt/rocm-${GPU_BACKEND_VERSION}\"" \
&&   echo '    hip:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo "        - spec: \"hip@${GPU_BACKEND_VERSION}\"" \
&&   echo "          prefix: \"/opt/rocm-${GPU_BACKEND_VERSION}\"" \
&&   echo '    hsa-rocr-dev:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo "        - spec: \"hsa-rocr-dev@${GPU_BACKEND_VERSION}\"" \
&&   echo "          prefix: \"/opt/rocm-${GPU_BACKEND_VERSION}\"" \
&&   echo '    llvm-amdgpu:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo "        - spec: \"llvm-amdgpu@${GPU_BACKEND_VERSION}\"" \
&&   echo "          prefix: \"/opt/rocm-${GPU_BACKEND_VERSION}\"" \
&&   echo '' \
&&   echo '  concretizer:' \
&&   echo '    unify: true' \
&&   echo '  config:' \
&&   echo '    install_tree:' \
&&   echo '      root: /opt/software' \
&&   echo '  view: /opt/views/view') > /opt/spack-environment/spack.yaml

# Apply feq-parse patch to add "c" build dependency
COPY ./envs/x86/rocm/feq-parse.patch /tmp/feq-parse.patch
#
RUN SPACK_PKGS_ROOT=$(spack repo list | awk '{print $NF}') &&\
    SPACK_BUILTIN_PKGS_ROOT=${SPACK_PKGS_ROOT/repos\/spack_repo\/builtin} &&\
    patch -p1 -d $SPACK_BUILTIN_PKGS_ROOT < /tmp/feq-parse.patch

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && spack env activate . && spack repo list && spack install --fail-fast && spack gc -y

# Strip all the binaries
RUN find -L /opt/views/view/* -type f -exec readlink -f '{}' \; | \
    xargs file -i | \
    grep 'charset=binary' | \
    grep 'x-executable\|x-archive\|x-sharedlib' | \
    awk -F: '{print $1}' | xargs strip

# Modifications to the environment that are necessary to run
RUN cd /opt/spack-environment && \
    spack env activate --sh -d . > activate.sh


# Bare OS image to run the installed executables
FROM docker.io/rockylinux:9

COPY --from=builder /opt/spack-environment /opt/spack-environment
COPY --from=builder /opt/software /opt/software

# Install HIP  #
COPY ./envs/x86/rocm/rocm.repo /etc/yum.repos.d/rocm.repo
ARG GPU_BACKEND_VERSION=6.4.3
RUN sed -i "s/@GPU_BACKEND_VERSION@/${GPU_BACKEND_VERSION}/g" /etc/yum.repos.d/rocm.repo && \
 && dnf update -y \
 && dnf install -y epel-release \
 && dnf update -y \
    dnf clean all && \
    dnf update -y && \
    dnf --enablerepo epel install -y rocm-hip-libraries rocm-hip-runtime \ 
        bzip2 \
        cmake \
        curl-minimal \
        file \
        findutils \
        gcc-c++ \
        gcc \
        gcc-gfortran

# paths.view is a symlink, so copy the parent to avoid dereferencing and duplicating it
COPY --from=builder /opt/views /opt/views

RUN { \
      echo '#!/bin/sh' \
      && echo '.' /opt/spack-environment/activate.sh \
      && echo 'exec "$@"'; \
    } > /entrypoint.sh \
&& chmod a+x /entrypoint.sh \
&& ln -s /opt/views/view /opt/view


LABEL "mpi"="mpich"
ENTRYPOINT [ "/entrypoint.sh" ]
CMD [ "/bin/bash" ]

