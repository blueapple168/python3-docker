# ==============================================================================
# 多阶段构建：UOS Server 20 Python 运行时镜像
# 构建阶段：builder 编译 Python3.13；运行阶段：交付最终业务镜像
# 基础镜像：ghcr.io/blueapple168/uos-server-20-1070a:latest (amd64+arm64 多架构)
# 内置组件：Python3.13.14
# 支持架构：x86_64 aarch64
# ==============================================================================
ARG BASE_IMAGE_TAG=uos-server-20-1070a:latest
FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG} AS builder
# buildx注入变量，必须紧跟FROM之后
ARG TARGETARCH

# OCI标准容器标签
LABEL org.opencontainers.image.authors="blueapple" \
      org.opencontainers.image.version="python 3.13.14" \
      org.opencontainers.image.description="UOS Server 20 | Python3.13 Runtime" \
      org.opencontainers.image.licenses="Internal" \
      org.opencontainers.image.vendor="Enterprise" \
      os="UnionTechOS Server 20" \
      os.version="${BASE_IMAGE_TAG}" \
      component="python-runtime"

# 构建阶段环境
ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PATH=/usr/local/bin:$PATH

# 安装编译依赖
RUN set -eux; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=False \
        gcc \
        gcc-c++ \
        make \
        wget \
        tar \
        xz \
        bzip2 \
        ca-certificates \
        openssl-devel \
        libffi-devel \
        zlib-devel \
        bzip2-devel \
        xz-devel \
        sqlite-devel \
        tk-devel \
        ncurses-devel \
        gdbm-devel \
        libnsl2-devel \
        bluez-libs-devel \
        libuuid-devel \
        findutils \
        which \
        tzdata \
    ; \
    yum clean all; \
    rm -rf /var/cache/yum/* /tmp/* /var/tmp/*

# Python编译参数
ENV PYTHON_VERSION=3.13.14 \
    PYTHON_SHA256=639e43243c620a308f968213df9e00f2f8f62332f7adbaa7a7eeb9783057c690

# 下载源码
RUN set -eux; \
    cd /tmp; \
    wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tar.xz"; \
    echo "${PYTHON_SHA256}  python.tar.xz" | sha256sum -c -; \
    mkdir -p /usr/src/python; \
    tar --extract --file=python.tar.xz --directory=/usr/src/python --strip-components=1; \
    rm python.tar.xz

WORKDIR /usr/src/python

# 编译Python，QEMU仿真下自动识别架构，增加调试输出
RUN set -eux; \
    echo "==== TARGETARCH=${TARGETARCH} ===="; \
    echo "==== uname -m=$(uname -m) ===="; \
    gnuArch="$(uname -m)"; \
    echo "==== gnuArch=${gnuArch} ===="; \
    ./configure \
        --build="${gnuArch}" \
        --enable-loadable-sqlite-extensions \
        --enable-optimizations \
        --enable-option-checking=fatal \
        --enable-shared \
        --with-lto \
        --prefix=/usr/local \
        --without-ensurepip \
        LDFLAGS="-Wl,-rpath=/usr/local/lib" \
    ; \
    nproc="$(nproc)"; \
    make -j "${nproc}" EXTRA_CFLAGS="-DNDEBUG -g -fwrapv -O3 -Wall"; \
    make altinstall; \
    cd /; \
    rm -rf /tmp/*; \
    echo "/usr/local/lib" > /etc/ld.so.conf.d/python3.conf; \
    ldconfig; \
    /usr/local/bin/python3.13 --version; \
    find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec strip '{}' + || true; \
    find /usr/local -type f \( -name '*.a' -o -name '*.pyc' -o -name '*.pyo' \) -delete

# ------------------------------
# 运行时阶段
# ------------------------------
FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG} AS runtime
ARG BASE_IMAGE_TAG

ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PYTHON_VERSION=3.13.14 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=UTF-8 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PATH=/usr/local/bin:$PATH \
    TZ=Asia/Shanghai \
    APP_USER=app

# 运行时系统依赖
RUN set -eux; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=False \
        ca-certificates \
        openssl \
        libffi \
        zlib \
        bzip2 \
        xz \
        sqlite \
        tk \
        ncurses \
        gdbm \
        libnsl2 \
        bluez-libs \
        libuuid \
        findutils \
        which \
        tzdata \
    ; \
    yum clean all; \
    rm -rf \
        /var/cache/yum/* \
        /tmp/* \
        /var/tmp/* \
        /usr/share/man/* \
        /usr/share/doc/* \
        /usr/share/locale/[a-df-z]* \
        /usr/share/locale/e[!n]*; \
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 从builder复制编译产物
COPY --from=builder /usr/local /usr/local
COPY --from=builder /etc/ld.so.conf.d/python3.conf /etc/ld.so.conf.d/python3.conf
RUN ldconfig

# 初始化pip，清理冗余文件，创建普通业务用户
RUN set -eux; \
    python3.13 --version; \
    ln -sf python3.13 /usr/local/bin/python3; \
    ln -sf python3.13 /usr/local/bin/python; \
    ln -sf pip3.13 /usr/local/bin/pip3; \
    ln -sf pip3.13 /usr/local/bin/pip; \
    python3 -m ensurepip --upgrade; \
    pip3 install --no-cache --upgrade pip setuptools wheel; \
    find /usr/local -depth \
        \( \
            \( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
            -o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
            -o \( -type f -a -name '*.a' -not -name 'libpython*.a' \) \
        \) -exec rm -rf '{}' +; \
    mkdir -p /${APP_USER}; \
    groupadd -g 1000 ${APP_USER}; \
    useradd -r -u 1000 -g ${APP_USER} -d /${APP_USER} -s /bin/bash ${APP_USER}; \
    chown -R ${APP_USER}:${APP_USER} /${APP_USER}; \
    sed -i 's/Version: 39\.2\.0/Version: 2026.08.02/g' /usr/lib/python3.6/site-packages/setuptools-*.dist-info/METADATA 2>/dev/null || true; \
    echo '. /etc/profile' > /${APP_USER}/.bashrc

USER ${APP_USER}
WORKDIR /${APP_USER}

CMD ["python3"]
