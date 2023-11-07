#
# NOTE: Use the Makefiles to build this image correctly.
#

ARG BASE_IMG=<jupyter>
FROM $BASE_IMG

ARG TARGETARCH

# args - software versions
ARG TENSORFLOW_VERSION=2.13.0

# args - cuda versions
#  - TF CUDA version matrix: https://www.tensorflow.org/install/source#gpu
#  - CUDA repo (for package version numbers):
#     - amd64 (ubuntu 22.04): https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/
#     - arm64 (ubuntu 20.04): https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/sbsa/
ARG CUDA_VERSION=11.8

USER root

# install - cuda
# NOTE: we use the Ubuntu 20.04 repo for arm64 as TensorRT for CUDA 11.8 is not available in the 22.04 repo
RUN case "${TARGETARCH}" in \
      amd64) \
        CUDA_DISTRO="ubuntu2204"; \
        CUDA_ARCH="x86_64"; \
        CUDNN_VERSION="8.6.0.163"; \
        TENSORRT_VERSION="8.6.1.6" ;; \
      arm64) \
        CUDA_DISTRO="ubuntu2004"; \
        CUDA_ARCH="sbsa"; \
        CUDNN_VERSION="8.6.0.163"; \
        TENSORRT_VERSION="8.5.3" ;; \
      *) \
        echo "Unsupported architecture: ${TARGETARCH}"; \
        exit 1 ;; \
    esac \
 && curl -sL "https://developer.download.nvidia.com/compute/cuda/repos/${CUDA_DISTRO}/${CUDA_ARCH}/3bf863cc.pub" | apt-key add - \
 && echo "deb https://developer.download.nvidia.com/compute/cuda/repos/${CUDA_DISTRO}/${CUDA_ARCH}/ /" > /etc/apt/sources.list.d/cuda.list \
 && apt-get -yq update \
 && apt-get -yq install --no-install-recommends --no-install-suggests \
    cuda-command-line-tools-${CUDA_VERSION/./-} \
    cuda-compat-${CUDA_VERSION/./-} \
    cuda-cudart-${CUDA_VERSION/./-} \
    cuda-libraries-${CUDA_VERSION/./-} \
    cuda-nvcc-${CUDA_VERSION/./-} \
    libcudnn8=${CUDNN_VERSION}-1+cuda${CUDA_VERSION} \
    libnvinfer8=${TENSORRT_VERSION}-1+cuda${CUDA_VERSION} \
    libnvinfer-plugin8=${TENSORRT_VERSION}-1+cuda${CUDA_VERSION} \
 && apt-mark hold \
    libcublas-${CUDA_VERSION/./-} \
    libnccl2 \
    libcudnn8 \
    libnvinfer8 \
    libnvinfer-plugin8 \
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
ENV NVIDIA_REQUIRE_CUDA "cuda>=${CUDA_VERSION}"

# install - tensorflow
RUN python3 -m pip install --quiet --no-cache-dir \
    tensorflow==${TENSORFLOW_VERSION}

# install - requirements.txt
COPY --chown=${NB_USER}:users requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
 && rm -f /tmp/requirements.txt