#
# NOTE: Use the Makefiles to build this image correctly.
#

ARG BASE_IMG=<jupyter>
FROM $BASE_IMG

# args - software versions
# https://github.com/pytorch/pytorch/releases
# https://github.com/pytorch/audio/releases
# https://github.com/pytorch/vision/releases
# NOTE: correct version alignments are found at https://pytorch.org/get-started/previous-versions/
#       they sometimes forget to publish a github release of `torchaudio` or `torchvision` when the cut a
#       patch release of `torch`, but they still publish a patch of `torchaudio` and `torchvision` on pypi
ARG PYTORCH_VERSION=2.7.0.dev20250310
#ARG TORCHAUDIO_VERSION=2.6.0.dev2025XXXX
#ARG TORCHVISION_VERSION=0.22.0.dev2025XXXX

# nvidia container toolkit
# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/docker-specialized.html
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV NVIDIA_REQUIRE_CUDA "cuda>=12.8"

# install - pytorch (cuda)
RUN python3 -m pip install --quiet --no-cache-dir --index-url https://download.pytorch.org/whl/nightly/cu128 \
    torch==${PYTORCH_VERSION} \
    torchaudio \
    torchvision

# install - requirements.txt
COPY --chown=${NB_USER}:${NB_GID} requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
 && rm -f /tmp/requirements.txt

# install -- build tools
RUN export DEBIAN_FRONTEND=noninteractive \
 && apt-get -yq update \
 && apt-get -yq install --no-install-recommends \
    build-essential \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*
