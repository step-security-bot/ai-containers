# Copyright (c) 2022 Intel Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# ============================================================================
#
# This file was assembled from multiple pieces, whose use is documented
# throughout. Please refer to the TensorFlow dockerfiles documentation
# for more information.

# based on https://github.com/pytorch/pytorch/blob/master/Dockerfile
#
# NOTE: To build this you will need a docker version >= 19.03 and DOCKER_BUILDKIT=1
#
#       If you do not use buildkit you are not going to have a good time
#
#       For reference:
#           https://docs.docker.com/develop/develop-images/build_enhancements/


# This image provides a Python 3.9 environment you can use to run your Python
# applications.

ARG BASE_IMAGE=ubuntu
ARG BASE_TAG=22.04

FROM ${BASE_IMAGE}:${BASE_TAG} AS dgpu-base

ENV DEBIAN_FRONTEND=noninteractive

# See http://bugs.python.org/issue19846
ENV LANG C.UTF-8
ARG PYTHON_VERSION

EXPOSE 8080

ENV LANG=C.UTF-8

SHELL ["/bin/bash", "-c"]

RUN apt-get update -y && \
    apt-get install -y --no-install-recommends --fix-missing \
    apt-utils \
    build-essential \
    bzip2 \
    ca-certificates \
    clinfo \
    cmake \
    curl \
    diffutils \
    g++ \
    gcc \
    git \
    gnupg2 \
    gpg-agent \
    gzip \
    make \
    patch \
    rsync \
    unzip \
    wget \
    xz-utils && \
    rm -rf /var/lib/apt/lists/*

#GPU Drivers setup

ARG DEVICE
ARG ICD_VER
ARG LEVEL_ZERO_GPU_VER
ARG LEVEL_ZERO_VER
ARG LEVEL_ZERO_DEV_VER
ARG DPCPP_VER
ARG MKL_VER

RUN wget -qO - https://repositories.intel.com/graphics/intel-graphics.key | \
    gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg && \
    printf 'deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/graphics/ubuntu jammy %s\n' "$DEVICE" | \
    tee  /etc/apt/sources.list.d/intel.gpu.jammy.list && \
    wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
   | gpg --dearmor | tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null && \
   echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
   | tee /etc/apt/sources.list.d/oneAPI.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends --fix-missing \
    intel-opencl-icd=${ICD_VER} \
    intel-level-zero-gpu=${LEVEL_ZERO_GPU_VER} \
    level-zero=${LEVEL_ZERO_VER} \
    level-zero-dev=${LEVEL_ZERO_DEV_VER} \
    intel-oneapi-runtime-dpcpp-cpp=${DPCPP_VER} \
    intel-oneapi-runtime-mkl=${MKL_VER} \
    libgl1-mesa-glx \
    libglib2.0-0  && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

FROM dgpu-base as deep-learning-python

#Setting up non-root directories
ARG USERNAME
RUN useradd --uid 1000 -d /home/${USERNAME} -s /bin/bash -m ${USERNAME}
USER $USERNAME
WORKDIR /home/${USERNAME}

ENV CONDA_ROOT=/home/${USERNAME}/conda

#Miniconda Python Installation
ARG MINICONDA_VERSION
ARG PYTHON_VERSION
ARG IDP_VERSION

RUN wget --no-check-certificate https://repo.anaconda.com/miniconda/Miniconda3-${MINICONDA_VERSION}.sh -O miniconda.sh && \        
    chmod +x miniconda.sh && \
    ./miniconda.sh -b -p ${CONDA_ROOT} && \
    rm ./miniconda.sh && \
    ln -s ${CONDA_ROOT} ${CONDA_ROOT}/../miniconda3 && \
    export PATH=${CONDA_ROOT}/bin/:${PATH} && \
    conda update -y conda && \
    conda config --add channels intel && \
    conda init --all && \
    python -m pip install --no-cache-dir jupyterlab jupyterhub notebook jupyter-server-proxy && \
    conda clean -y --all

ENV PATH ${CONDA_ROOT}/condabin:${CONDA_ROOT}/bin/:${PATH}
ENV LD_LIBRARY_PATH=/opt/intel/oneapi/lib:/opt/intel/oneapi/lib/intel64:${LD_LIBRARY_PATH}

#PyTorch Installation
ARG TORCH_VERSION
ARG TORCHVISION_VERSION
ARG IPEX_VERSION
ARG TORCH_WHL_URL
ARG IPEX_WHL_URL
ARG TORCHVISION_WHL_URL

RUN \
    conda create -yn torch intelpython3_core=${IDP_VERSION} python=${PYTHON_VERSION} && \
    conda install -yn torch ipython ipykernel kernda -c conda-forge && \
    conda install -yn torch matplotlib -c intel && \
    conda run -n torch python -m pip install --no-cache-dir -f ${TORCH_WHL_URL} \
        torch==${TORCH_VERSION} \
        intel_extension_for_pytorch==${IPEX_VERSION} \
        torchvision==${TORCHVISION_VERSION} -f ${TORCHVISION_WHL_URL} && \
    conda run -n torch python -m pip install --no-cache-dir --ignore-installed \
        cloud-data-connector \
        dataset-librarian && \
    conda run -n torch python -m pip install --no-cache-dir \
        scipy==1.11.1 \
        certifi=='2023.07.22' && \
    conda clean -y --all

RUN echo "unset OCL_ICD_VENDORS" >> ${CONDA_ROOT}/envs/torch/etc/conda/activate.d/activate-opencl-rt.sh

#Tensorflow Installtion
ARG TF_PACKAGE
ARG TF_PACKAGE_VERSION
ARG ITEX_VERSION

RUN conda create -yn tensorflow intelpython3_core python=${PYTHON_VERSION} && \
    conda install -yn tensorflow ipython ipykernel kernda -c conda-forge && \
    conda run -n tensorflow python -m pip install --no-cache-dir ${TF_PACKAGE}${TF_PACKAGE_VERSION:+==${TF_PACKAGE_VERSION}} \
        intel-extension-for-tensorflow[xpu]==${ITEX_VERSION} \
        requests==2.31.0 \
        cryptography==41.0.2 && \
    conda run -n tensorflow python -m pip install --no-cache-dir --ignore-installed \
        cloud-data-connector \
        dataset-librarian && \
    conda run -n tensorflow python -m pip install --no-cache-dir \
        scipy==1.11.1 \
        certifi=='2023.07.22' && \
    conda clean -y --all

RUN echo "unset OCL_ICD_VENDORS" >> ${CONDA_ROOT}/envs/tensorflow/etc/conda/activate.d/activate-opencl-rt.sh

FROM deep-learning-python as deep-learning-jupyter

ARG KERNEL_NAME_PT="Intel PyTorch"
ARG KERNEL_NAME_TF="Intel TensorFlow"
ENV PORT=8888
EXPOSE $PORT

RUN mkdir -p ~/jupyter/ && chmod -R a+rwx ~/jupyter/ && \
    mkdir ~/.local && chmod a+rwx ~/.local

RUN \
    ${CONDA_ROOT}/envs/torch/bin/python -m ipykernel install --user --name torch --display-name "${KERNEL_NAME_PT}" && \
    ${CONDA_ROOT}/envs/torch/bin/kernda -o -y ~/.local/share/jupyter/kernels/$(echo torch | sed -e 's/\(.*\)/\L\1/')/kernel.json && \
    ${CONDA_ROOT}/envs/tensorflow/bin/python -m ipykernel install --user --name tensorflow --display-name "${KERNEL_NAME_TF}" && \
    ${CONDA_ROOT}/envs/tensorflow/bin/kernda -o -y ~/.local/share/jupyter/kernels/$(echo tensorflow | sed -e 's/\(.*\)/\L\1/')/kernel.json && \
    python -m ipykernel.kernelspec --user

WORKDIR /home/${USERNAME}/jupyter

CMD ["bash", "-c", "jupyter notebook --notebook-dir=~/jupyter --port $PORT --ip 0.0.0.0 --no-browser --allow-root"]

FROM deep-learning-jupyter as distributed-deep-learning

USER root

#oneccl installation
ARG CCL_VER="N/A"
RUN if [ ${CCL_VER} != "N/A" ]; then \
        apt-get update && \
        apt-get install -y --no-install-recommends --fix-missing \
        intel-oneapi-runtime-ccl=${CCL_VER} && \
        rm -rf /var/lib/apt/lists/*; \
    fi

#OneCCl specific flags
ENV CCL_ROOT="/opt/intel/oneapi/lib/intel64" 
ENV CCL_ZE_IPC_EXCHANGE=sockets
ENV LD_LIBRARY_PATH=/opt/intel/oneapi/lib/intel64/libfabric:${LD_LIBRARY_PATH}

#Install OpenMPI
RUN apt-get update -y && apt-get install -y --no-install-recommends --fix-missing \
    libopenmpi-dev \
    openmpi-bin \
    openmpi-common

ENV OMPI_ALLOW_RUN_AS_ROOT=1
ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1
ENV OMPI_MCA_tl_tcp_if_exclude="lo,docker0"

#Install OpenSSH
RUN apt-get install -y --only-upgrade --no-install-recommends --fix-missing \
    openssh-client \
    openssh-server && \
    rm -rf /var/lib/apt/lists/*

RUN rm -rf /usr/bin/apt*

RUN mkdir -p /var/run/sshd

#https://github.com/openucx/ucx/issues/4742#issuecomment-584059909
ENV UCX_TLS=ud,sm,self

ARG USERNAME
USER $USERNAME

ARG ONECCL_BIND_PT_WHL_URL="N/A"
ARG ONECCL_BIND_PT_VERSION="N/A"

RUN conda run -n torch python -m pip install --no-cache-dir \
        oneccl_bind_pt==${ONECCL_BIND_PT_VERSION} -f ${ONECCL_BIND_PT_WHL_URL} \
        py-cpuinfo \
        intel-extension-for-deepspeed && \
    conda run -n torch python -m pip install --no-cache-dir deepspeed

# Install Horovod
ARG HOROVOD_VERSION
ARG HOROVOD_WITH_TENSORFLOW=1
ARG HOROVOD_WITHOUT_MXNET=1
ARG HOROVOD_WITHOUT_GLOO=1
ARG HOROVOD_WITH_MPI=1


RUN conda run -n tensorflow python -m pip install --no-cache-dir intel-optimization-for-horovod${HOROVOD_VERSION+==${HOROVOD_VERSION}} && \
    conda clean -y --all

FROM deep-learning-jupyter as inc-base

ENV SIGOPT_PROJECT=.

ARG INC_VERSION: ${INC_VERSION:-2.2.1}

RUN conda run -n torch python -m pip install --no-cache-dir \
        neural-compressor${INC_VERSION:+==${INC_VERSION}} \
        numpy==1.23.5 && \
    conda clean -y --all

RUN conda run -n tensorflow python -m pip install --no-cache-dir \
        neural-compressor${INC_VERSION:+==${INC_VERSION}} \
        numpy==1.23.5 \
        tf2onnx \
        onnxruntime && \
    conda clean -y --all

USER root
RUN rm -rf /usr/bin/apt*

ARG USERNAME
USER $USERNAME