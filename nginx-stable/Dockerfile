# syntax=docker/dockerfile:1
# =====================================================================
# UOS + nginx (stable) 多架构镜像：linux/amd64 + linux/arm64
# 由同一 Dockerfile 通过 TARGETARCH/$TARGETPLATFORM 区分架构，
# 由 GitHub Actions 在 nginx 官方发布 stable 新版本时自动构建。
# =====================================================================
# 基础镜像: ghcr.io/blueapple168/uos-server-20-1070a (UOS Server 20, RHEL 8 兼容, glibc 2.28)
# 注意: 该基础镜像需同时包含 linux/amd64 与 linux/arm64 架构
ARG OS_VERSION=20-1070a
FROM --platform=$TARGETPLATFORM ghcr.io/blueapple168/uos-server-20-1070a:latest
ARG OS_VERSION

LABEL org.opencontainers.image.authors="baidongying@cnpc.com.cn" \
      org.opencontainers.image.version="1.0" \
      org.opencontainers.image.description="nginx-stable (UOS, multi-arch)" \
      os="UOS Linux" \
      os.version="${OS_VERSION}" \
      component="base-image"

# BuildKit 内置目标架构变量：arm64 => aarch64；amd64 => x86_64
ARG TARGETARCH
ENV NGINX_VERSION=stable

# 配置 nginx 官方 stable yum 仓库（按架构映射仓库路径）
RUN rm -rf /etc/yum.repos.d/*.repo && \
    case "${TARGETARCH}" in \
        arm64) BASE_ARCH=aarch64 ;; \
        amd64) BASE_ARCH=x86_64 ;; \
        *) echo "unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac && \
    echo '[nginx-stable]' > /etc/yum.repos.d/nginx.repo && \
    echo 'name=nginx stable repo' >> /etc/yum.repos.d/nginx.repo && \
    echo "baseurl=http://nginx.org/packages/centos/8/${BASE_ARCH}/" >> /etc/yum.repos.d/nginx.repo && \
    echo 'gpgcheck=1' >> /etc/yum.repos.d/nginx.repo && \
    echo 'enabled=1' >> /etc/yum.repos.d/nginx.repo && \
    echo 'gpgkey=https://nginx.org/keys/nginx_signing.key' >> /etc/yum.repos.d/nginx.repo && \
    rpm --import https://nginx.org/keys/nginx_signing.key

# 安装 nginx 并做基础配置
RUN yum makecache && \
    yum install -y --setopt=install_weak_deps=false --nogpgcheck --nodocs nginx && \
    yum clean all && \
    rm -rf /var/cache/yum/* /var/tmp/* /tmp/* && \
    mkdir -p /var/cache/nginx/client_temp \
             /var/run/nginx \
             /usr/share/nginx/html && \
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    if [ -f /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA ]; then \
        sed -i 's/Version: 39\.2\.0/Version: 2026.10.23/g' \
            /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA; \
    fi

COPY --chmod=644 default.conf /etc/nginx/conf.d/default.conf

WORKDIR /usr/share/nginx/html

EXPOSE 8080 443

CMD ["nginx", "-g", "daemon off;"]
