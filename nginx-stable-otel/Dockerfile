# syntax=docker/dockerfile:1
# =====================================================================
# UOS + nginx (stable) + otel/njs/image-filter/xslt 动态模块 多架构镜像
# linux/amd64 + linux/arm64，由 TARGETARCH/$TARGETPLATFORM 区分架构，
# 由 GitHub Actions 在 nginx 官方发布 stable 新版本时自动构建。
# =====================================================================
# 基础镜像: ghcr.io/blueapple168/uos-server-20-1070a (UOS Server 20, RHEL 8 兼容, glibc 2.28)
# 注意: 该基础镜像需同时包含 linux/amd64 与 linux/arm64 架构
ARG OS_VERSION=20-1070a
FROM --platform=$TARGETPLATFORM ghcr.io/blueapple168/uos-server-20-1070a:latest
ARG OS_VERSION

LABEL org.opencontainers.image.authors="blueapple" \
      org.opencontainers.image.version="1.0" \
      org.opencontainers.image.description="nginx-stable-otel (UOS, multi-arch)" \
      os="UOS Linux" \
      os.version="${OS_VERSION}" \
      component="base-image"

# BuildKit 内置目标架构变量（FROM 之后才能用）
ARG TARGETARCH

ENV NGINX_VERSION=stable \
    TZ=Asia/Shanghai \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

COPY --chmod=644 default.conf /etc/nginx/conf.d/
COPY --chmod=644 nginx.conf /etc/nginx/nginx.conf

RUN set -eux; \
    # 目标架构 -> nginx 仓库路径（TARGETARCH 缺失时回退 uname）
    case "${TARGETARCH:-$(uname -m)}" in \
        arm64|aarch64) BASE_ARCH=aarch64 ;; \
        amd64|x86_64)  BASE_ARCH=x86_64  ;; \
        *) echo "Unsupported arch: TARGETARCH=${TARGETARCH:-unset}, uname=$(uname -m)" >&2; exit 1 ;; \
    esac; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo ; \
    echo '[nginx-stable]' > /etc/yum.repos.d/nginx.repo; \
    echo 'name=nginx stable repo' >> /etc/yum.repos.d/nginx.repo; \
    echo "baseurl=https://nginx.org/packages/centos/8/${BASE_ARCH}/" >> /etc/yum.repos.d/nginx.repo; \
    echo 'gpgcheck=0' >> /etc/yum.repos.d/nginx.repo; \
    echo 'enabled=1' >> /etc/yum.repos.d/nginx.repo; \
    echo 'module_hotfixes=true' >> /etc/yum.repos.d/nginx.repo; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=false --nogpgcheck --nodocs \
        c-ares \
        libedit \
        gd \
        libxslt \
        nginx \
        nginx-module-otel \
        nginx-module-njs \
        nginx-module-image-filter \
        nginx-module-xslt ; \
    yum clean all; \
    rm -rf /var/cache/yum/* /var/tmp/* /tmp/*; \
    mkdir -p \
        /var/cache/nginx/client_temp \
        /var/run/nginx \
        /usr/share/nginx/html; \
    ln -sf /dev/stdout /var/log/nginx/access.log; \
    ln -sf /dev/stderr /var/log/nginx/error.log; \
    if [ -f /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA ]; then \
        sed -i 's/Version: 39\.2\.0/Version: 2026.10.23/g' \
            /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA; \
    fi; \
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone; \
    if ! getent group nginx >/dev/null; then \
        groupadd -r nginx; \
    fi; \
    if ! getent passwd nginx >/dev/null; then \
        useradd -r -g nginx -s /sbin/nologin -d /var/cache/nginx nginx; \
    fi; \
    chown -R nginx:nginx /var/cache/nginx; \
    chmod -R 755 /var/cache/nginx; \
    chmod 644 /etc/nginx/conf.d/default.conf /etc/nginx/nginx.conf; \
    rpm -e --nodeps \
           python3-pip-wheel \
           python3-libcomps \
           python3-gpg \
           python3-hawkey \
           python3-rpm \
           python3-libdnf \
           python3-dbus \
           python3-dnf \
           platform-python \
           platform-python-setuptools \
           python3-setuptools-wheel \
           python3-libs \
           python3-libselinux \
           chkconfig \
           dbus-daemon \
           dbus-tools \
           dbus-common \
           device-mapper \
           device-mapper-libs \
           libblockdev-utils \
           libblockdev-fs \
           libblockdev-part \
           libblockdev-crypto \
           libblockdev-loop \
           libblockdev-swap \
           libblockdev-mdraid \
           cryptsetup-libs \
           volume_key-libs \
           qt5-qtbase \
           qt5-qtbase-common \
           info \
           libuosdevicec-libs \
           libuosdevicex-libs \
           libuosdeviceh-libs \
           ima-evm-utils \
           tpm2-tss; \
    rm -rf /etc/nginx/nginx.conf.rpmnew \
           /etc/nginx/conf.d/default.conf.rpmnew \
           /var/lib/rpm/__db* \
           /tmp/* \
           /var/tmp/* \
           /usr/share/man/* \
           /usr/share/doc/* \
           /usr/share/info/* \
           /usr/share/locale/[a-df-z]* \
           /usr/share/locale/e[a-rt-z]* \
           /usr/share/locale/en@* \
           /usr/share/locale/en_* \
           /var/log/*.log \
           /var/log/*.log.* \
           /var/cache/yum/*

WORKDIR /usr/share/nginx/html

EXPOSE 8080 443

CMD ["nginx", "-g", "daemon off;"]
