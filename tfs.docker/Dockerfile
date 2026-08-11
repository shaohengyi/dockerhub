# ============================================================
# TensorFlow Serving 2.18.0 GPU Runtime
# ============================================================

FROM tensorflow/serving:2.18.0-gpu

ARG PYTHON_VERSION=3.10.12

ENV DEBIAN_FRONTEND=noninteractive

USER root


# ============================================================
# 安装基础工具、tini 和 Python 编译依赖
# ============================================================

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        curl \
        wget \
        vim \
        tini \
        build-essential \
        libssl-dev \
        zlib1g-dev \
        libffi-dev \
        libbz2-dev \
        libreadline-dev \
        libsqlite3-dev \
        libncursesw5-dev \
        liblzma-dev \
        libgdbm-dev \
        libnss3-dev \
        uuid-dev \
        tk-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*


# ============================================================
# 源码安装 Python 3.10.12
# ============================================================

RUN cd /tmp && \
    wget -q https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tgz && \
    tar -xzf Python-${PYTHON_VERSION}.tgz && \
    cd Python-${PYTHON_VERSION} && \
    ./configure --enable-optimizations && \
    make -j"$(nproc)" && \
    make altinstall && \
    ln -sf /usr/local/bin/python3.10 /usr/bin/python && \
    ln -sf /usr/local/bin/python3.10 /usr/bin/python3 && \
    curl -sS https://bootstrap.pypa.io/get-pip.py | /usr/local/bin/python3.10 && \
    ln -sf /usr/local/bin/pip3.10 /usr/local/bin/pip && \
    cd / && \
    rm -rf /tmp/Python-${PYTHON_VERSION}*


# ============================================================
# Python / Jupyter 环境
# ============================================================

WORKDIR /serving

RUN python --version && \
    pip --version

RUN pip install --no-cache-dir \
    jupyter \
    jupyterlab \
    notebook \
    pyarrow \
    filelock \
    schedule \
    pyyaml \
    dacite \
    protobuf==4.25.3 \
    grpcio==1.64.1 \
    grpcio-tools \
    wheel


# ============================================================
# 项目 Python 包
# ============================================================

COPY setup.py test.py build_config_api.sh /serving/

RUN chmod +x build_config_api.sh && \
    ./build_config_api.sh


# ============================================================
# 构建阶段验证
# ============================================================

RUN python --version && \
    pip --version && \
    jupyter --version && \
    which tini && \
    tensorflow_model_server --version


# ============================================================
# TensorFlow Serving
# ============================================================

EXPOSE 8500 8501

# 继承 tensorflow/serving:2.18.0-gpu 原有 ENTRYPOINT / CMD
