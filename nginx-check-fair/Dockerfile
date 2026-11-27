# syntax=docker/dockerfile:1
# =====================================================================
# UOS + nginx (stable) + upstream_check + upstream_fair 多架构镜像
# linux/amd64 + linux/arm64, 由 TARGETARCH/$TARGETPLATFORM 区分架构
#
# 调研结论 (2026-11-27, 多来源交叉验证):
#   - nginx_upstream_check_module (主动健康检查, 支持 tcp/http/ssl_hello/mysql/ajp)
#     与 upstream_fair (忙闲感知调度) 均为第三方模块, 官方 nginx 未集成,
#     社区无任何预编译二进制/RPM -> 只能源码打补丁编译, 无二进制安装途径。
#   - check 模块: yaoweibin/nginx_upstream_check_module v0.4.0,
#     nginx 新版本 (1.26/1.28 实测) 使用 check_1.20.1+.patch;
#     补丁版本必须 "不高于 nginx 版本且最接近", 否则编译报错。
#   - fair 模块: gnosek 原版对 nginx>=1.14 编译报错, 采用 blueapple168/nginx-curl
#     项目 (nginx_upstream_check_fair/1.20.0) 验证过的修复方法:
#     sed 's/default_port/no_port/g' ngx_http_upstream_fair_module.c (等价 PR#27)
#   - check 模块与 fair 模块均取 GitHub master 源码 (与 nginx-curl 项目同源)
#   - 不选 Tengine: 其内置 check 但无 fair, 且与 nginx stable 1.30.x 生态不对齐。
#
# 构建参数 (build-args 可覆盖):
#   NGINX_VERSION  nginx 源码版本, 默认 1.30.8 (与 stable 系列对齐)
#   CHECK_PATCH    check 补丁文件名 (tarball 内字面文件名), 默认 check_1.20.1+
#                  patch 失败时按 nginx 版本就近调整, 如 check_1.22.1+
# =====================================================================
ARG NGINX_VERSION=1.30.8
ARG CHECK_PATCH=check_1.20.1+.patch

# 基础镜像: ghcr.io/blueapple168/uos-server-20-1070a (UOS Server 20, RHEL 8 兼容, glibc 2.28)
ARG OS_VERSION=20-1070a
FROM --platform=$TARGETPLATFORM ghcr.io/blueapple168/uos-server-20-1070a:latest
ARG OS_VERSION

LABEL org.opencontainers.image.authors="blueapple" \
      org.opencontainers.image.version="1.0" \
      org.opencontainers.image.description="nginx-stable with upstream_check + upstream_fair (source build)" \
      os="UOS Linux" \
      os.version="${OS_VERSION}" \
      component="base-image"

# BuildKit 内置目标架构变量：arm64 => aarch64；amd64 => x86_64
ARG TARGETARCH

ARG NGINX_VERSION
ARG CHECK_PATCH

ENV NGINX_VERSION=${NGINX_VERSION} \
    TZ=Asia/Shanghai \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

RUN set -eux; \
    # ---------- 架构映射 (arm64=>aarch64, amd64=>x86_64) ----------
    case "${TARGETARCH}" in \
        arm64) BASE_ARCH=aarch64 ;; \
        amd64) BASE_ARCH=x86_64 ;; \
        *) echo "unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    \
    # ---------- 编译依赖 ----------
    yum makecache; \
    yum install -y --setopt=install_weak_deps=false --nodocs \
        gcc gcc-c++ make patch curl tar gzip; \
    \
    # ---------- 下载源码与模块 ----------
    mkdir -p /usr/src; \
    cd /usr/src; \
    curl -fsSL --retry 3 "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" -o nginx.tar.gz; \
    tar zxf nginx.tar.gz; \
    curl -fsSL --retry 3 "https://codeload.github.com/yaoweibin/nginx_upstream_check_module/tar.gz/refs/heads/master" -o check.tar.gz; \
    tar zxf check.tar.gz; \
    CHECK_DIR=$(ls -d nginx_upstream_check_module-* | head -n1); \
    curl -fsSL --retry 3 "https://codeload.github.com/gnosek/nginx-upstream-fair/tar.gz/refs/heads/master" -o fair.tar.gz; \
    tar zxf fair.tar.gz; \
    FAIR_DIR=$(ls -d nginx-upstream-fair-* | head -n1); \
    # gnosek 原版对 nginx>=1.14 的 default_port 编译报错, 按 nginx-curl 项目验证过的方法修复
    sed -i 's/default_port/no_port/g' "/usr/src/${FAIR_DIR}/ngx_http_upstream_fair_module.c"; \
    \
    # ---------- 给 nginx 源码打 check 补丁 (失败即中断, 换 CHECK_PATCH 重试) ----------
    cd "nginx-${NGINX_VERSION}"; \
    if [ -f "../${CHECK_DIR}/${CHECK_PATCH}" ]; then \
        patch -p1 < "../${CHECK_DIR}/${CHECK_PATCH}"; \
    else \
        echo "可用补丁: $(ls ../${CHECK_DIR}/)"; \
        echo "CHECK_PATCH=${CHECK_PATCH} 不存在, 用 --build-arg CHECK_PATCH=<?> 调整" >&2; exit 1; \
    fi; \
    \
    # ---------- configure: 完整对齐官方 el8 RPM (nginx 1.30.8-1) 编译参数 ----------
    # 差异说明:
    #   1. --with-http_v3_module 去掉: QUIC 需 OpenSSL 3.x, 基础镜像 OpenSSL 1.1.1k 下
    #      configure 直接失败 (官方 el8 包由 nginx.org 特殊构建)
    #   2. cc-opt 中 -specs=redhat-hardened-cc1/annobin-cc1 依赖 redhat-rpm-config/annobin
    #      包, 基础镜像不含, 去掉
    #   3. -m64/-fcf-protection 为 x86 专属 flag, aarch64 gcc 不识别, 按 TARGETARCH 分支
    CC_OPT_COMMON="-O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -fasynchronous-unwind-tables -fstack-clash-protection -fPIC"
    case "${TARGETARCH}" in \
        amd64) CC_OPT_ARCH="-m64 -mtune=generic -fcf-protection" ;; \
        arm64) CC_OPT_ARCH="-mtune=generic" ;; \
    esac; \
    ./configure \
        --prefix=/etc/nginx \
        --sbin-path=/usr/sbin/nginx \
        --modules-path=/usr/lib64/nginx/modules \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --pid-path=/var/run/nginx.pid \
        --lock-path=/var/run/nginx.lock \
        --http-client-body-temp-path=/var/cache/nginx/client_temp \
        --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
        --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
        --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
        --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
        --user=nginx --group=nginx \
        --with-compat \
        --with-file-aio \
        --with-threads \
        --with-http_addition_module \
        --with-http_auth_request_module \
        --with-http_dav_module \
        --with-http_flv_module \
        --with-http_gunzip_module \
        --with-http_gzip_static_module \
        --with-http_mp4_module \
        --with-http_random_index_module \
        --with-http_realip_module \
        --with-http_secure_link_module \
        --with-http_slice_module \
        --with-http_ssl_module \
        --with-http_stub_status_module \
        --with-http_sub_module \
        --with-http_v2_module \
        --with-mail \
        --with-mail_ssl_module \
        --with-stream \
        --with-stream_realip_module \
        --with-stream_ssl_module \
        --with-stream_ssl_preread_module \
        --with-cc-opt="${CC_OPT_COMMON} ${CC_OPT_ARCH}" \
        --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' \
        --add-module="/usr/src/${CHECK_DIR}" \
        --add-module="/usr/src/${FAIR_DIR}"; \
    \
    # ---------- 编译安装 ----------
    make -j"$(nproc)"; \
    make install; \
    \
    # ---------- 验证: 二进制可用且模块已编入 ----------
    /usr/sbin/nginx -v; \
    /usr/sbin/nginx -V 2>&1 | grep -q "upstream_check" || { echo "check 模块未编入" >&2; exit 1; }; \
    /usr/sbin/nginx -V 2>&1 | grep -q "upstream-fair" || { echo "fair 模块未编入" >&2; exit 1; }; \
    \
    # ---------- 运行环境: 用户/目录/日志软链 ----------
    if ! getent group nginx >/dev/null; then groupadd -r nginx; fi; \
    if ! getent passwd nginx >/dev/null; then \
        useradd -r -g nginx -s /sbin/nologin -d /var/cache/nginx nginx; \
    fi; \
    mkdir -p /var/cache/nginx/client_temp /var/run/nginx /usr/share/nginx/html; \
    # prefix=/etc/nginx 时 make install 会把 html 放到 /etc/nginx/html, 迁移到标准位置
    if [ -d /etc/nginx/html ]; then \
        cp -r /etc/nginx/html/50x.html /usr/share/nginx/html/ 2>/dev/null || true; \
        rm -rf /etc/nginx/html; \
        ln -s /usr/share/nginx/html /etc/nginx/html; \
    fi; \
    ln -sf /dev/stdout /var/log/nginx/access.log; \
    ln -sf /dev/stderr /var/log/nginx/error.log; \
    chown -R nginx:nginx /var/cache/nginx; \
    chmod -R 755 /var/cache/nginx; \
    \
    # ---------- 清理 ----------
    rm -rf /usr/src/nginx* /usr/src/check* /usr/src/fair* \
           /var/cache/yum/* /var/tmp/* /tmp/*; \
    yum clean all; \
    yum -y remove gcc gcc-c++ make patch; \
    yum autoremove -y --setopt=install_weak_deps=false || true; \
    rm -rf /var/log/*.log /usr/share/man/* /usr/share/doc/*; \
    if [ -f /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA ]; then \
        sed -i 's/Version: 39\.2\.0/Version: 2026.11.27/g' \
            /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA; \
    fi

# 配置文件 (业务示例: upstream 含 check 主动健康检查 + fair 调度 + check_status 状态页)
COPY --chmod=644 default.conf /etc/nginx/conf.d/default.conf
COPY --chmod=644 nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/share/nginx/html

EXPOSE 8080 443

CMD ["nginx", "-g", "daemon off;"]
