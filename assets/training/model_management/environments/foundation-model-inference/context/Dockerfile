FROM nvidia/cuda:12.1.0-devel-ubuntu20.04

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    TZ=Etc/UTC \
    DEBIAN_FRONTEND=noninteractive
RUN apt update && apt upgrade -y && apt install software-properties-common -y && add-apt-repository ppa:deadsnakes/ppa -y
RUN apt install git -y

ENV MINICONDA_VERSION py310_23.10.0-1
ENV PATH /opt/miniconda/bin:$PATH
RUN apt-get update && \
    apt-get install -y --no-install-recommends wget runit
RUN wget -qO /tmp/miniconda.sh https://repo.anaconda.com/miniconda/Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
    bash /tmp/miniconda.sh -bf -p /opt/miniconda && \
    conda update --all -c conda-forge -y && \
    conda clean -ay && \
    rm -rf /opt/miniconda/pkgs && \
    rm /tmp/miniconda.sh && \
    find / -type d -name __pycache__ | xargs rm -rf

ENV AZUREML_CONDA_ENVIRONMENT_PATH /azureml-envs/default

# Create conda environment with py310
RUN conda create -p $AZUREML_CONDA_ENVIRONMENT_PATH \
    python=3.10 \
    -c conda-forge 

ENV PATH $AZUREML_CONDA_ENVIRONMENT_PATH/bin:$PATH

ENV CONDA_DEFAULT_ENV=$AZUREML_CONDA_ENVIRONMENT_PATH
ENV CONDA_PREFIX=$AZUREML_CONDA_ENVIRONMENT_PATH

WORKDIR /

# Needed for megablocks
RUN pip install torch~=2.3.0 torchvision --index-url https://download.pytorch.org/whl/cu121

# mixtral specific
ARG torch_cuda_arch_list='7.0 7.5 8.0 8.6 8.9 9.0+PTX'
ENV TORCH_CUDA_ARCH_LIST=${torch_cuda_arch_list}
RUN pip install git+https://github.com/stanford-futuredata/megablocks.git@5897cd6

# For local testing
# Need to copy src code and install in editable mode
# COPY . .
# RUN pip install -e ./ --no-cache-dir

# Flash attention needed for phi-3-mini-v
RUN pip install packaging
RUN pip install flash-attn --no-build-isolation

# When copied to assets repo, change to install from public pypi
RUN pip install llm-optimized-inference==0.1.8 --no-cache-dir

RUN pip uninstall -y vllm

# Apply patch
COPY ./vllm_phiv.patch ./vllm_phiv.patch
RUN git clone https://github.com/vllm-project/vllm.git && \
    cd vllm && \
    git checkout f8e7adda21810104382bdf3febe3ea02c72f7348 && \
    git apply ../vllm_phiv.patch && \
    pip install -r requirements-cuda.txt && \
    python setup.py bdist_wheel && \
    pip install dist/*.whl
# Address pillow vulnerability - torchvision above installs 10.2.0
RUN pip install pillow==10.3.0

# clean conda and pip caches
RUN rm -rf ~/.cache/pip

ADD /api_server /var/runit/api_server
RUN sed -i 's/\r$//g' /var/runit/api_server/run
RUN chmod +x /var/runit/api_server/run

ENV SVDIR=/var/runit
ENV WORKER_TIMEOUT=3600
EXPOSE 5001
CMD [ "runsvdir", "/var/runit" ]