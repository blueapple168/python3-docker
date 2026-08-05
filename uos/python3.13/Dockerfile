# ==============================================================================
# 多阶段构建：UOS Server 20 Python 运行时镜像
# 构建阶段：builder 编译 Python3.13.14；运行阶段：交付最终业务镜像
# 基础镜像：registry.uniontech.com/uos-server-base/uos-server-20-1070a:latest
# 内置组件：Python3.13.14
# 支持架构：x86_64(aarch64) 通过 buildx+qemu-user 仿真编译
# ==============================================================================
ARG BASE_IMAGE_TAG=uos-server-20-1070a:latest
FROM ghcr.io/blueapple168/uos-server-20-1070a:latest AS builder

# buildx内置变量，必须在FROM之后第一行，仅做日志输出，不用于configure
ARG TARGETARCH

# OCI标准容器标签
LABEL org.opencontainers.image.authors="blueapple" \
      org.opencontainers.image.version="python 3.13.14" \
      org.opencontainers.image.description="UOS Server 20 | Python3.13 Runtime" \
      org.opencontainers.image.licenses="Internal" \
      org.opencontainers.image.vendor="Enterprise" \
      os="UnionTechOS Server 20" \
      os.version="${BASE_IMAGE_TAG}" \
      component="python3-runtime"

ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PATH=/usr/local/bin:$PATH

# 安装编译依赖
RUN set -eux; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=False \
        gcc gcc-c++ make wget tar xz gzip findutils which \
        openssl-devel bzip2-devel libffi-devel zlib-devel \
        readline-devel sqlite-devel tk-devel xz-devel ncurses-devel \
        gdbm-devel libnsl2-devel bluez-libs-devel libuuid-devel; \
    yum clean all; \
    rm -rf /var/cache/yum/* /tmp/* /var/tmp/*

# Python源码版本与校验和
ENV GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305 \
    PYTHON_VERSION=3.13.14 \
    PYTHON_SHA256=639e43243c620a308f968213df9e00f2f8f62332f7adbaa7a7eeb9783057c690

# 下载解压源码
RUN set -eux; \
    cd /tmp; \
    wget -O python.tar.xz "https://mirrors.huaweicloud.com/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
    echo "$PYTHON_SHA256 *python.tar.xz" | sha256sum -c -; \
    mkdir -p /usr/src/python; \
    tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
    rm -f python.tar.xz

WORKDIR /usr/src/python

# QEMU‑user仿真环境自动探测架构编译，删除--build/--host
RUN set -eux; \
    echo ">>> buildx TARGETARCH=${TARGETARCH}"; \
    echo ">>> container uname -m=$(uname -m)"; \
    ./configure \
        --enable-loadable-sqlite-extensions \
        --enable-optimizations \
        --enable-option-checking=fatal \
        --enable-shared \
        --with-lto \
        --prefix=/usr/local \
        --without-ensurepip \
        LDFLAGS="-Wl,-rpath=/usr/local/lib"; \
    nproc="$(nproc)"; \
    make -j "$nproc" EXTRA_CFLAGS="-DNDEBUG -g -fwrapv -O3 -Wall"; \
    make -j "$nproc" altinstall; \
    cd /; \
    rm -rf /usr/src/python; \
    echo "/usr/local/lib" > /etc/ld.so.conf.d/python3.13.conf; \
    ldconfig; \
    /usr/local/bin/python3.13 --version; \
    find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec strip '{}' + || true; \
    find /usr/local -type f \( -name '*.a' -o -name '*.pyc' -o -name '*.pyo' \) -delete; \
    ldconfig

# ===================== 运行阶段 =====================
FROM ghcr.io/blueapple168/uos-server-20-1070a:latest

LABEL org.opencontainers.image.authors="blueapple" \
      org.opencontainers.image.version="python 3.13.14" \
      org.opencontainers.image.description="UOS Server 20 Python3.13 runtime image"

ENV LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PATH=/usr/local/bin:$PATH \
    PYTHON_VERSION=3.13.14 \
    PYTHONDONTWRITEBYTECODE=1 \
    TZ=Asia/Shanghai \
    APP_USER=app

# 运行期系统依赖
RUN set -eux; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=False \
        ca-certificates openssl libffi zlib bzip2 xz sqlite tk ncurses gdbm libnsl2 bluez-libs libuuid findutils which tzdata; \
    yum clean all; \
    rm -rf /var/cache/yum/* /tmp/* /var/tmp/* \
        /usr/share/man/* /usr/share/doc/* \
        /usr/share/locale/[a-df-z]* /usr/share/locale/en@* /usr/share/locale/zh_*; \
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime; echo $TZ > /etc/timezone

# 从builder复制编译产物
COPY --from=builder /usr/local /usr/local
COPY --from=builder /etc/ld.so.conf.d/python3.13.conf /etc/ld.so.conf.d/python3.13.conf

RUN set -eux; \
    ldconfig; \
    # 确保共享库存在
    ls -la /usr/local/lib/libpython* || echo "Checking Python libraries"; \
    # 验证Python安装
    python3.13 --version; \
    # 创建必要的符号链接
    cd /usr/local/bin; \
    ln -sf python3.13 python3; \
    ln -sf python3.13 python; \
    # 确保pip可用
    if [ ! -f /usr/local/bin/pip3.13 ]; then \
        python3 -m ensurepip --upgrade --default-pip; \
    fi; \
    ln -sf pip3.13 pip3; \
    ln -sf pip3.13 pip; \
    # 验证pip
    pip3 --version; \
    find /usr/local -depth '(' \
      '(' -type d -a '(' -name test -o -name tests -o -name idle_test ')' ')' \
      -o \
      '(' -type f -a '(' -name '*.pyc' -o -name '*.pyo' ')' ')' \
      -o \
      '(' -type f -a -name '*.a' -a -not -name 'libpython*.a' ')' \
    ')' -exec rm -rf {} + ; \
    # 优化pip配置
    python3 -m pip install --no-cache-dir --upgrade \
        pip \
        setuptools \
        wheel; \
    # 创建应用目录并设置权限
    mkdir -p /${APP_USER}; \
    # 创建非root用户
    groupadd -g 1000 ${APP_USER}; \
    useradd -r -u 1000 -g ${APP_USER} -d /${APP_USER} -s /bin/bash ${APP_USER} ; \
    chown -R ${APP_USER}:${APP_USER} /${APP_USER}; \
    # 修复setuptools版本信息
    sed -i 's/Version: 39\.2\.0/Version: 2026.08.02/g' /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA 2>/dev/null || true ; \
    # 添加用户环境变量
    echo '. ~/.bashrc' > /${APP_USER}/.bash_profile ; \
    echo '. /etc/profile' > /${APP_USER}/.bashrc

USER ${APP_USER}
WORKDIR /home/${APP_USER}
CMD ["python3"]
