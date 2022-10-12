ARG BASE_IMAGE=nvidia/cuda:11.3.0-cudnn8-runtime-ubuntu18.04

FROM ${BASE_IMAGE} AS compile-image
ARG BASE_IMAGE=nvidia/cuda:11.3.0-cudnn8-runtime-ubuntu18.04
ENV PYTHONUNBUFFERED TRUE

RUN apt-get update && \
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

ARG CUDA_VERSION=cu113

RUN python -m pip install --no-cache-dir torch==1.11.0+cu113 torchvision==0.12.0+cu113 -f https://download.pytorch.org/whl/torch_stable.html && \
    python -m pip install --no-cache-dir -r https://raw.githubusercontent.com/pytorch/serve/master/requirements/common.txt


RUN python -m pip install -U setuptools && python -m pip install --no-cache-dir captum torchtext==0.12.0 torchserve torch-model-archiver

# Final image for production
FROM ${BASE_IMAGE} AS runtime-image

ENV PYTHONUNBUFFERED TRUE

RUN apt-get update && \
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


COPY --chown=model-server --from=compile-image /home/venv /home/venv

ENV PATH="/home/venv/bin:$PATH"

# this is torchserve-kfs build
RUN pip install --upgrade pip
COPY requirements.txt /home/model-server/requirements.txt
RUN pip install -r /home/model-server/requirements.txt

COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
COPY kserve_wrapper /home/model-server/kserve_wrapper

RUN chmod +x /usr/local/bin/dockerd-entrypoint.sh \
    && chown -R model-server /home/model-server

COPY config.properties /home/model-server/config.properties
RUN mkdir /home/model-server/model-store && chown -R model-server /home/model-server/model-store



EXPOSE 8080 8081 8082 7070 7071

USER model-server
WORKDIR /home/model-server
ENV TEMP=/home/model-server/tmp
ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]
CMD ["serve"]
