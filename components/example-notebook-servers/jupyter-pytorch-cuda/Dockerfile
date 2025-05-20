#
# NOTE: Use the Makefiles to build this image correctly.
#

ARG BASE_IMG=<jupyter>
FROM $BASE_IMG

ARG TARGETARCH

# args - cuda versions
ARG CUDA_VERSION=12.8
ARG CUDA_TOOLKIT_VERSION=12.8.1-1
ARG CUDNN_VERSION=9.8.0.87-1

USER root

##
## TODO: move the ssh install to a lower image, if we make SSH the standard
##
# install -- build tools and ssh
RUN export DEBIAN_FRONTEND=noninteractive \
 && apt-get -yq update \
 && apt-get -yq install --no-install-recommends \
    build-essential \
    cmake \
    openssh-client \
    openssh-server \
    zlib1g \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# install - cuda
RUN case "${TARGETARCH}" in \
      amd64) CUDA_ARCH="x86_64" ;; \
      arm64) CUDA_ARCH="arm64" ;; \
      *) echo "Unsupported architecture: ${TARGETARCH}"; exit 1 ;; \
    esac \
 && curl -sL "https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/${CUDA_ARCH}/3bf863cc.pub" | apt-key add - \
 && echo "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/${CUDA_ARCH}/ /" > /etc/apt/sources.list.d/cuda.list \
 && apt-get -yq update \
 && apt-get -yq install --no-install-recommends \
    cuda-command-line-tools-${CUDA_VERSION/./-} \
    cuda-cudart-${CUDA_VERSION/./-} \
    cuda-libraries-${CUDA_VERSION/./-} \
    cuda-nvcc-${CUDA_VERSION/./-} \
    cuda-toolkit-${CUDA_VERSION/./-}=${CUDA_TOOLKIT_VERSION} \
    libcudnn9-cuda-12=${CUDNN_VERSION} \
 && apt-mark hold \
    libcublas-${CUDA_VERSION/./-} \
    libcudnn9-dev-cuda-12 \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && ln -s /usr/local/cuda-${CUDA_VERSION} /usr/local/cuda

USER $NB_UID

# envs - cuda
ENV PATH /usr/local/nvidia/bin:/usr/local/cuda/bin:${PATH}
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64

# nvidia container toolkit
# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/docker-specialized.html
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV NVIDIA_REQUIRE_CUDA "cuda>=12.8"

# args - python versions
# https://github.com/pytorch/pytorch/releases
# https://github.com/pytorch/audio/releases
# https://github.com/pytorch/vision/releases
# NOTE: correct version alignments are found at https://pytorch.org/get-started/previous-versions/
#       they sometimes forget to publish a github release of `torchaudio` or `torchvision` when the cut a
#       patch release of `torch`, but they still publish a patch of `torchaudio` and `torchvision` on pypi
ARG PYTORCH_VERSION=2.7.0
#ARG TORCHAUDIO_VERSION=2.6.0.dev2025XXXX
#ARG TORCHVISION_VERSION=0.22.0.dev2025XXXX
ARG BITSANDBYTES_VERSION=0.45.5

# install - bitsandbytes
# NOTE: upstream bitsandbytes dose not ship an aarch64 wheel
#       build from source (with cuda support) at the specified version according to:
#       https://huggingface.co/docs/bitsandbytes/main/en/installation#cuda-compile
RUN curl -fsSL https://github.com/bitsandbytes-foundation/bitsandbytes/archive/refs/tags/${BITSANDBYTES_VERSION}.tar.gz -o /tmp/bitsandbytes.tar.gz \
 && mkdir /tmp/bitsandbytes-${BITSANDBYTES_VERSION}/ \
 && tar xzf /tmp/bitsandbytes.tar.gz -C /tmp \
 && cd /tmp/bitsandbytes-${BITSANDBYTES_VERSION}/ \
 && cmake -DCOMPUTE_BACKEND=cuda -S . \
 && make \
 && python3 -m pip install --quiet --no-cache-dir . \
 && rm -f /tmp/bitsandbytes.tar.gz \
 && rm -rf /tmp/bitsandbytes-${BITSANDBYTES_VERSION}/

# install - pytorch (cuda)
RUN python3 -m pip install --quiet --no-cache-dir --index-url https://download.pytorch.org/whl/cu128 \
    torch==${PYTORCH_VERSION} \
    torchaudio \
    torchvision

# install - requirements.txt
COPY --chown=${NB_USER}:${NB_GID} requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
 && rm -f /tmp/requirements.txt

# s6 - copy scripts
COPY --chown=${NB_USER}:${NB_GID} --chmod=755 s6/ /etc