ARG BASE_IMAGE=nvidia/cuda:11.3.0-cudnn8-runtime-ubuntu18.04

FROM ${BASE_IMAGE} AS compile-image
ARG BASE_IMAGE=nvidia/cuda:11.3.0-cudnn8-runtime-ubuntu18.04
ENV PYTHONUNBUFFERED TRUE

RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    #apt --fix-broken -y install && \
    apt-get update && \
    apt remove python-pip  python3-pip && \
    DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
    ca-certificates \
    g++ \
    python3.8 \
    python3.8-dev \
    python3.8-distutils \
    python3.8-venv \
    python3-venv \
    openjdk-17-jdk \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && cd /tmp \
    && curl -O https://bootstrap.pypa.io/get-pip.py \
    && python3.8 get-pip.py


RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.8 1 \
    && update-alternatives --install /usr/local/bin/pip pip /usr/local/bin/pip3.8 1

RUN python3.8 -m venv /home/venv

ENV PATH="/home/venv/bin:$PATH"

RUN python -m pip install -U pip setuptools

# This is only useful for cuda env
RUN export USE_CUDA=1

ARG CUDA_VERSION="cu113"

RUN TORCH_VER=1.11.0 && \
    TORCH_VISION_VER=0.12.0 && \
    if echo "$BASE_IMAGE" | grep -q "cuda:"; then \
        # Install CUDA version specific binary when CUDA version is specified as a build arg
        if [ "$CUDA_VERSION" ]; then \
            python -m pip install --no-cache-dir torch==$TORCH_VER+$CUDA_VERSION torchvision==$TORCH_VISION_VER+$CUDA_VERSION -f https://download.pytorch.org/whl/torch_stable.html; \
        # Install the binary with the latest CUDA version support
        else \
            python -m pip install --no-cache-dir torch torchvision; \
        fi; \
        python -m pip install --no-cache-dir -r https://raw.githubusercontent.com/pytorch/serve/master/requirements/common.txt; \
    # Install the CPU binary
    else \
        python -m pip install --no-cache-dir torch==$TORCH_VER+cpu torchvision==$TORCH_VISION_VER+cpu -f https://download.pytorch.org/whl/torch_stable.html; \
    fi
RUN python -m pip install -U setuptools && python -m pip install --no-cache-dir captum torchtext torchserve torch-model-archiver

# Final image for production
FROM ${BASE_IMAGE} AS runtime-image

ENV PYTHONUNBUFFERED TRUE

RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
    python3.8 \
    python3.8-distutils \
    python3.8-dev \
    # using openjdk-17-jdk due to circular dependency(ca-certificates) bug in openjdk-17-jre-headless debian package
    # https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1009905
    openjdk-17-jdk \
    build-essential \
    && rm -rf /var/lib/apt/lists/* \
    && cd /tmp

RUN useradd -m model-server \
    && mkdir -p /home/model-server/tmp


ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

COPY --chown=model-server --from=compile-image /home/venv /home/venv

ENV PATH="/home/venv/bin:$PATH"

COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh

RUN chmod +x /usr/local/bin/dockerd-entrypoint.sh \
    && chown -R model-server /home/model-server

COPY config.properties /home/model-server/config.properties
RUN mkdir /home/model-server/model-store && chown -R model-server /home/model-server/model-store

## this is torchserve-kfs build
RUN pip install --upgrade pip
COPY requirements.txt /home/model-server/requirements.txt
RUN pip install -r /home/model-server/requirements.txt
COPY kserve_wrapper /home/model-server/kserve_wrapper

EXPOSE 8080 8081 8082 7070 7071

USER model-server
WORKDIR /home/model-server
ENV TEMP=/home/model-server/tmp
ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]
CMD ["serve"]
