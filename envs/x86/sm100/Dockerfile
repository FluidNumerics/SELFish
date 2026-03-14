FROM docker.io/rockylinux:9 AS bootstrap

ARG CUDA_VERSION=13.0

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

# Install CUDA toolkit from NVIDIA repo
RUN dnf config-manager \
        --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo \
 && dnf clean all \
 && dnf update -y \
 && dnf install -y cuda-toolkit-$(echo ${CUDA_VERSION} | tr '.' '-')

RUN ls -l /usr/local/cuda-${CUDA_VERSION}/include/cuda.h

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
&&   echo '  - openmpi@5.0.8 +cuda cuda_arch=100' \
&&   echo '  - hdf5@1.14.5 +fortran +mpi' \
&&   echo '  - cmake@3.31.11'\
&&   echo '  packages:' \
&&   echo '    all:' \
&&   echo '      require:' \
&&   echo '      - target=x86_64_v3' \
&&   echo '      prefer:' \
&&   echo '      - cuda_arch=100' \
&&   echo '    cuda:' \
&&   echo '      buildable: false' \
&&   echo '      externals:' \
&&   echo "        - spec: \"cuda@${CUDA_VERSION}\"" \
&&   echo "          prefix: \"/usr/local/cuda-${CUDA_VERSION}\"" \
&&   echo '' \
&&   echo '  concretizer:' \
&&   echo '    unify: true' \
&&   echo '  config:' \
&&   echo '    install_tree:' \
&&   echo '      root: /opt/software' \
&&   echo '  view: /opt/views/view') > /opt/spack-environment/spack.yaml

# Apply feq-parse patch to add "c" build dependency
COPY ./envs/x86/sm100/feq-parse.patch /tmp/feq-parse.patch
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
        lcov

# Install CUDA runtime libraries
ARG CUDA_VERSION=13.0
RUN dnf config-manager \
        --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo \
 && dnf clean all \
 && dnf update -y \
 && dnf install -y \
        cuda-libraries-$(echo ${CUDA_VERSION} | tr '.' '-') \
        cuda-nvtx-$(echo ${CUDA_VERSION} | tr '.' '-')

# paths.view is a symlink, so copy the parent to avoid dereferencing and duplicating it
COPY --from=builder /opt/views /opt/views

RUN { \
      echo '#!/bin/sh' \
      && echo '.' /opt/spack-environment/activate.sh \
      && echo 'exec "$@"'; \
    } > /entrypoint.sh \
&& chmod a+x /entrypoint.sh \
&& ln -s /opt/views/view /opt/view


LABEL "mpi"="openmpi"
ENTRYPOINT [ "/entrypoint.sh" ]
CMD [ "/bin/bash" ]
