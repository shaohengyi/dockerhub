# ============================================================
# Stage 1: Build Hadoop 2.7.1
# ============================================================

FROM ubuntu:16.04 AS builder

LABEL stage="hadoop-builder"
LABEL description="Builds Hadoop 2.7.1 from source on a compatible Ubuntu 16.04."

ARG HADOOP_VERSION=2.7.1
ARG PROTOBUF_VERSION=2.5.0
ARG MAVEN_VERSION=3.3.9

ARG MAVEN_URL=https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz
ARG HADOOP_URL=https://archive.apache.org/dist/hadoop/core/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}-src.tar.gz
ARG PROTOBUF_URL=https://github.com/google/protobuf/releases/download/v${PROTOBUF_VERSION}/protobuf-${PROTOBUF_VERSION}.tar.gz

ENV DEBIAN_FRONTEND=noninteractive
ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
ENV M2_HOME=/opt/maven
ENV PATH=$M2_HOME/bin:$JAVA_HOME/bin:$PATH
ENV HADOOP_SRC_HOME=/usr/src/hadoop-${HADOOP_VERSION}-src

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    wget tar gzip \
    openjdk-8-jdk \
    build-essential autoconf automake libtool cmake \
    fuse libfuse-dev \
    zlib1g-dev libssl-dev && \
    rm -rf /var/lib/apt/lists/*

RUN ln -s $JAVA_HOME/jre/lib/amd64/server/libjvm.so \
    $JAVA_HOME/jre/lib/amd64/libjvm.so

RUN cd /tmp && \
    wget -q ${PROTOBUF_URL} && \
    tar -xzvf protobuf-${PROTOBUF_VERSION}.tar.gz && \
    cd protobuf-${PROTOBUF_VERSION} && \
    ./configure --prefix=/usr && \
    make && \
    make install && \
    ldconfig && \
    cd / && \
    rm -rf /tmp/* && \
    protoc --version

RUN wget -q ${MAVEN_URL} -P /tmp && \
    tar -xzvf /tmp/apache-maven-${MAVEN_VERSION}-bin.tar.gz -C /opt && \
    ln -s /opt/apache-maven-${MAVEN_VERSION} ${M2_HOME} && \
    rm -f /tmp/apache-maven-${MAVEN_VERSION}-bin.tar.gz

RUN mkdir -p ${HADOOP_SRC_HOME} && \
    wget -q ${HADOOP_URL} -P ${HADOOP_SRC_HOME} && \
    tar -xzvf \
    ${HADOOP_SRC_HOME}/hadoop-${HADOOP_VERSION}-src.tar.gz \
    -C ${HADOOP_SRC_HOME} \
    --strip-components=1 && \
    rm -f ${HADOOP_SRC_HOME}/hadoop-${HADOOP_VERSION}-src.tar.gz

WORKDIR ${HADOOP_SRC_HOME}

RUN mvn package -Pdist,native -DskipTests -Dtar


# ============================================================
# Stage 2: TensorFlow Serving 2.18.0 GPU
# ============================================================

FROM tensorflow/serving:2.18.0-gpu

ARG HADOOP_VERSION=2.7.1
ARG PYTHON_VERSION=3.10

ENV DEBIAN_FRONTEND=noninteractive \
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 \
    HADOOP_HOME=/opt/hadoop \
    HADOOP_CONF_DIR=/etc/hadoop \
    PATH=$PATH:/opt/hadoop/bin:/opt/hadoop/sbin

USER root

RUN apt-get update && \
    apt-get install -y --no-install-recommends software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
    openjdk-8-jdk \
    python${PYTHON_VERSION} \
    python3-distutils \
    git \
    curl \
    vim \
    fuse \
    libssl1.1 \
    libssl-dev \
    zlib1g-dev && \
    update-alternatives --install \
    /usr/bin/python python /usr/bin/python${PYTHON_VERSION} 1 && \
    update-alternatives --install \
    /usr/bin/python3 python3 /usr/bin/python${PYTHON_VERSION} 1 && \
    curl -sS https://bootstrap.pypa.io/get-pip.py | python${PYTHON_VERSION} && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /serving

RUN pip install --no-cache-dir \
    pyarrow \
    filelock \
    schedule \
    pyyaml \
    dacite \
    protobuf==4.25.3 \
    grpcio==1.64.1 \
    grpcio-tools \
    wheel

COPY setup.py test.py build_config_api.sh /serving/

RUN chmod +x build_config_api.sh && ./build_config_api.sh

RUN mkdir -p /opt/hadoop-libs

COPY --from=builder \
    /lib/x86_64-linux-gnu/libssl.so.1.0.0 \
    /opt/hadoop-libs/

COPY --from=builder \
    /lib/x86_64-linux-gnu/libcrypto.so.1.0.0 \
    /opt/hadoop-libs/

COPY --from=builder \
    /usr/src/hadoop-2.7.1-src/hadoop-dist/target/hadoop-2.7.1 \
    /opt/hadoop

ENV HADOOP_HOME=/opt/hadoop
ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
ENV PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

ENV LD_LIBRARY_PATH=/opt/hadoop-libs:$HADOOP_HOME/lib/native

RUN echo "export JAVA_HOME=${JAVA_HOME}" \
    >> ${HADOOP_HOME}/etc/hadoop/hadoop-env.sh && \
    ln -s \
    ${HADOOP_HOME}/lib/native/libhdfs.so.0.0.0 \
    /usr/lib/libhdfs.so

EXPOSE 8500 8501
