# ====================== 构建阶段优化 ======================
FROM alpine:3.16 AS builder
LABEL maintainer="metacubex.official@gmail.com"

# 构建参数优化：默认线程数自动适配CPU，可覆盖
ARG THREADS=$(nproc)
ARG SHA=""
# 编译优化参数：启用O3优化、LTO链接优化、减小二进制体积
ARG CFLAGS="-O3 -flto -ffunction-sections -fdata-sections -march=native"
ARG CXXFLAGS="-O3 -flto -ffunction-sections -fdata-sections -std=c++17 -march=native"
ARG LDFLAGS="-Wl,--gc-sections -Wl,--strip-all -flto"

# 设置环境变量，确保编译工具链使用优化参数
ENV CFLAGS=${CFLAGS} \
    CXXFLAGS=${CXXFLAGS} \
    LDFLAGS=${LDFLAGS} \
    MAKEFLAGS="-j ${THREADS}" \
    TERM=xterm

# 安装依赖并清理缓存，分层安装减少重建
RUN set -xe && \
    # 更新源并安装构建依赖，使用国内源加速（可选）
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk update && \
    # 构建工具：基础编译环境
    apk add --no-cache --virtual .build-tools \
        git g++ build-base linux-headers cmake python3 py3-pip ninja && \
    # 依赖库：开发版
    apk add --no-cache --virtual .build-deps \
        curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev && \
    # 清理apk缓存，减小层体积
    rm -rf /var/cache/apk/*

# 构建quickjspp：单独层，利用缓存
WORKDIR /build/quickjspp
RUN set -xe && \
    git clone --depth=1 --single-branch https://github.com/ftk/quickjspp . && \
    git submodule update --init --depth=1 && \
    # 使用ninja构建加速，启用编译优化
    cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_FLAGS="${CFLAGS}" -DCMAKE_CXX_FLAGS="${CXXFLAGS}" . && \
    ninja quickjs && \
    # 安装到系统目录
    install -d /usr/lib/quickjs/ && \
    install -m644 quickjs/libquickjs.a /usr/lib/quickjs/ && \
    install -d /usr/include/quickjs/ && \
    install -m644 quickjs/quickjs.h quickjs/quickjs-libc.h /usr/include/quickjs/ && \
    install -m644 quickjspp.hpp /usr/include && \
    # 清理构建文件
    rm -rf /build/quickjspp/*

# 构建libcron：单独层
WORKDIR /build/libcron
RUN set -xe && \
    git clone --depth=1 --single-branch https://github.com/PerMalmberg/libcron . && \
    git submodule update --init --depth=1 && \
    cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS="${CXXFLAGS}" -DCMAKE_EXE_LINKER_FLAGS="${LDFLAGS}" . && \
    ninja libcron && \
    install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
    install -d /usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
    install -d /usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /usr/include/date/ && \
    rm -rf /build/libcron/*

# 构建toml11：单独层，版本锁定
WORKDIR /build/toml11
RUN set -xe && \
    git clone --depth=1 --single-branch --branch="v4.3.0" https://github.com/ToruNiina/toml11 . && \
    cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_STANDARD=11 -DCMAKE_CXX_FLAGS="${CXXFLAGS}" . && \
    ninja install && \
    rm -rf /build/toml11/*

# 构建subconverter：核心构建层
WORKDIR /build/subconverter
RUN set -xe && \
    git clone --depth=1 --single-branch https://github.com/metacubex/subconverter . && \
    # 可选：指定SHA版本
    [ -n "$SHA" ] && sed -i 's/\(v[0-9]\.[0-9]\.[0-9]\)/\1-'"$SHA"'/' src/version.h || true && \
    # 安装python依赖，使用国内源
    pip3 install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple gitpython && \
    # 更新规则
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    # 构建subconverter，启用所有优化
    cmake -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_C_FLAGS="${CFLAGS}" \
        -DCMAKE_CXX_FLAGS="${CXXFLAGS}" \
        -DCMAKE_EXE_LINKER_FLAGS="${LDFLAGS}" . && \
    ninja && \
    # 压缩二进制文件（进一步减小体积）
    strip --strip-all subconverter && \
    upx --best --lzma subconverter || true

# ====================== 最终镜像优化 ======================
FROM alpine:3.16 AS final
LABEL maintainer="metacubex.official@gmail.com"

# 设置时区，默认UTC（可通过-e TZ=Asia/Shanghai覆盖）
ARG TZ=UTC
ENV TZ=${TZ} \
    # 非root用户运行
    PUID=1000 \
    PGID=1000

# 创建非root用户，增强安全性
RUN set -xe && \
    # 更新源并安装运行时依赖
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk update && \
    apk add --no-cache \
        pcre2 libcurl yaml-cpp tzdata su-exec upx && \
    # 创建用户组和用户
    addgroup -g ${PGID} -S subconverter && \
    adduser -u ${PUID} -S subconverter -G subconverter -h /base -s /sbin/nologin && \
    # 设置时区
    ln -sf /usr/share/zoneinfo/${TZ} /etc/localtime && \
    echo ${TZ} > /etc/timezone && \
    # 清理缓存
    rm -rf /var/cache/apk/*

# 从构建阶段复制二进制和配置文件
COPY --from=builder --chown=subconverter:subconverter /build/subconverter/subconverter /usr/bin/
COPY --from=builder --chown=subconverter:subconverter /build/subconverter/base /base/

# 设置工作目录权限
WORKDIR /base
RUN chown -R subconverter:subconverter /base

# 健康检查：确保服务正常运行
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -q -O - http://127.0.0.1:25500/version || exit 1

# 暴露端口
EXPOSE 25500/tcp

# 使用非root用户启动，支持PUID/PGID动态调整
ENTRYPOINT ["/bin/sh", "-c"]
CMD ["su-exec ${PUID}:${PGID} subconverter"]
