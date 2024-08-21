#
# NOTE: Use the Makefiles to build this image correctly.
#

ARG BASE_IMG=<jupyter>
FROM $BASE_IMG

# Content below is based on the scripts/Dockerfiles here:
# https://github.com/HabanaAI/Setup_and_Install/blob/1.17.0/dockerfiles/base/Dockerfile.ubuntu22.04
# https://github.com/HabanaAI/Setup_and_Install/blob/1.17.0/dockerfiles/pytorch/Dockerfile.ubuntu

# args - software versions
ARG GAUDI_VERSION=1.17.1
ARG GAUDI_REVISION=40
ARG PYTORCH_VERSION=2.3.1
ARG AWS_EFA_VERSION=1.29.0
ARG LIBFABRIC_VERSION=1.20.0

# Gaudi stack doesn't (yet) support 3.11, thus
# downgrade python from 3.11.x to 3.10.14
RUN source /opt/conda/etc/profile.d/conda.sh \
 && conda activate base \
 && conda install -y -q --no-pin python=3.10.14 \
 && sed -i 's/python ==.*/python ==3.10.14/' /opt/conda/conda-meta/pinned \
 && conda clean -a -f -y

USER root

# install - support libraries
RUN export DEBIAN_FRONTEND=noninteractive \
 && apt-get update -yq \
 && apt-get install -yq --no-install-recommends \
    apt-utils \
    bc \
    build-essential \
    graphviz \
    iproute2 \
    libcairo2-dev \
    libgl1 \
    libglib2.0-dev \
    libgnutls30 \
    libgoogle-glog0v5 \
    libgoogle-perftools-dev \
    libhdf5-dev \
    libjemalloc2 \
    libjpeg-dev \
    liblapack-dev \
    libmkl-dev \
    libnuma-dev \
    libopenblas-dev \
    libpcre2-dev \
    libpq-dev \
    libselinux1-dev \
    lsof \
    moreutils \
    numactl \
    protobuf-compiler \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

ENV LD_PRELOAD=/lib/x86_64-linux-gnu/libtcmalloc.so.4

# install - elastic fabric adapter
RUN export DEBIAN_FRONTEND=noninteractive \
 && apt-get -yq update \
 && mkdir /tmp/efa \
 && curl -fsSL https://efa-installer.amazonaws.com/aws-efa-installer-$AWS_EFA_VERSION.tar.gz | tar xz -C /tmp/efa \
 && cd /tmp/efa/aws-efa-installer \
 && ./efa_installer.sh -y --skip-kmod --skip-limit-conf --no-verify \
 && rm -rf /tmp/efa \
 && rm -rf /etc/ld.so.conf.d/efa.conf /etc/profile.d/efa.sh \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# config - fabric adapter and mpi variables
ENV MPI_ROOT=/opt/amazon/openmpi
ENV PATH=${MPI_ROOT}/bin:$PATH
ENV MPICC=${MPI_ROOT}/bin/mpicc
ENV OPAL_PREFIX=${MPI_ROOT}
ENV RDMAV_FORK_SAFE=1
ENV FI_EFA_USE_DEVICE_RDMA=1
ENV LD_LIBRARY_PATH=${MPI_ROOT}/lib

# install - habana packages
RUN curl -fsSL https://vault.habana.ai/artifactory/api/gpg/key/public | gpg --dearmor -o /usr/share/keyrings/habana-artifactory.gpg \
 && chown root:root /usr/share/keyrings/habana-artifactory.gpg \
 && chmod 644 /usr/share/keyrings/habana-artifactory.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/habana-artifactory.gpg] https://vault.habana.ai/artifactory/debian jammy main" | tee /etc/apt/sources.list.d/habana.list \
 && export DEBIAN_FRONTEND=noninteractive \
 && apt-get update -yq \
 && apt-get install -yq --no-install-recommends \
    habanalabs-rdma-core="${GAUDI_VERSION}"-"${GAUDI_REVISION}" \
    habanalabs-thunk="${GAUDI_VERSION}"-"${GAUDI_REVISION}" \
    habanalabs-firmware-tools="${GAUDI_VERSION}"-"${GAUDI_REVISION}" \
    habanalabs-graph="${GAUDI_VERSION}"-"${GAUDI_REVISION}" \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# config - habana variables
ENV RDMA_CORE_ROOT=/opt/habanalabs/rdma-core/src
ENV RDMA_CORE_LIB=${RDMA_CORE_ROOT}/build/lib
ENV GC_KERNEL_PATH=/usr/lib/habanalabs/libtpc_kernels.so
ENV HABANA_LOGS=/var/log/habana_logs/
ENV HABANA_SCAL_BIN_PATH=/opt/habanalabs/engines_fw
ENV HABANA_PLUGINS_LIB_PATH=/opt/habanalabs/habana_plugins
ENV DATA_LOADER_AEON_LIB_PATH=/usr/lib/habanalabs/libaeon.so
ENV LD_LIBRARY_PATH=/usr/lib/habanalabs:$LD_LIBRARY_PATH

# install - libfabric
RUN curl -fsSL -o /tmp/libfabric-${LIBFABRIC_VERSION}.tar.bz2 https://github.com/ofiwg/libfabric/releases/download/v${LIBFABRIC_VERSION}/libfabric-${LIBFABRIC_VERSION}.tar.bz2 \
 && cd /tmp/ \
 && tar xf /tmp/libfabric-${LIBFABRIC_VERSION}.tar.bz2 \
 && cd /tmp/libfabric-${LIBFABRIC_VERSION} \
 && export LIBFABRIC_ROOT="/opt/habanalabs/libfabric-${LIBFABRIC_VERSION}" \
 && ./configure --prefix=$LIBFABRIC_ROOT --enable-psm3-verbs --enable-verbs=yes --with-synapseai=/usr \
 && make -j \
 && make install \
 && cd / \
 && rm -rf /tmp/libfabric-${LIBFABRIC_VERSION}.tar.bz2 /tmp/libfabric-${LIBFABRIC_VERSION}

# config - add libfabric to loadable libraries
ENV LD_LIBRARY_PATH=/opt/habanalabs/libfabric-${LIBFABRIC_VERSION}/lib:$LD_LIBRARY_PATH
ENV PATH=/opt/habanalabs/libfabric-${LIBFABRIC_VERSION}/bin:$PATH

# install - hccl wrapper for ofi
RUN curl -fsSL -o /tmp/main.zip https://github.com/HabanaAI/hccl_ofi_wrapper/archive/db1209054e186071e2da80d62a6efdd1fbaf5261.zip \
 && unzip /tmp/main.zip -d /tmp \
 && cd /tmp/hccl_ofi_wrapper-* \
 && export LIBFABRIC_ROOT="/opt/habanalabs/libfabric-${LIBFABRIC_VERSION}" \
 && make \
 && cp -f libhccl_ofi_wrapper.so /usr/lib/habanalabs/libhccl_ofi_wrapper.so \
 && cd / \
 && rm -rf /tmp/main.zip /tmp/hccl_ofi_wrapper-main \
 && /sbin/ldconfig

USER $NB_UID

# install - habana pytorch media modules
RUN python3 -m pip --no-cache --quiet install habana_media_loader=="${GAUDI_VERSION}"."${GAUDI_REVISION}"

# install - habana pytorch modules
RUN mkdir /tmp/gaudipt \
 && curl -fsSL https://vault.habana.ai/artifactory/gaudi-pt-modules/${GAUDI_VERSION}/${GAUDI_REVISION}/pytorch/ubuntu2204/pytorch_modules-v${PYTORCH_VERSION}_${GAUDI_VERSION}_${GAUDI_REVISION}.tgz | tar xz -C /tmp/gaudipt \
 && cd /tmp/gaudipt \
 && PIP_QUIET=1 PYTHON_VERSION=3.10 bash install.sh ${GAUDI_VERSION} ${GAUDI_REVISION} \
 && rm -rf /tmp/gaudipt

# copy $HOME to $HOME_TMP so it can be copied over to the PVC later
RUN cp -p -r -T "${HOME}" "${HOME_TMP}" \
     # give group same access as user (needed for OpenShift)
  && chmod -R g=u "${HOME_TMP}"

# config - enable generic support for gaudi devices
ENV PYTHONPATH=/home/$NB_UID:/usr/lib/habanalabs/

# install - requirements.txt
COPY --chown=${NB_USER}:users requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
 && rm -f /tmp/requirements.txt
