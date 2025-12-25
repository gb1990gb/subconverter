# 阶段1：基础构建环境（缓存层，仅当alpine版本或build-tools变更时重建）
FROM alpine:3.16 AS builder-base
LABEL maintainer="metacubex.official@gmail.com"
ARG THREADS="4"
ENV THREADS=${THREADS}

# 安装基础构建工具（仅安装一次，缓存）
RUN set -xe && \
    apk add --no-cache --virtual .build-tools \
        git g++ build-base linux-headers cmake python3 && \
    apk add --no-cache --virtual .build-deps \
        curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

# 阶段2：编译quickjspp（独立层，仅当quickjspp源码变更时重建）
FROM builder-base AS builder-quickjspp
WORKDIR /build/quickjspp
RUN set -xe && \
    git clone https://github.com/ftk/quickjspp --depth=1 . && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make quickjs -j $THREADS && \
    # 安装到临时目录，便于后续拷贝
    install -d /install/usr/lib/quickjs/ && \
    install -m644 quickjs/libquickjs.a /install/usr/lib/quickjs/ && \
    install -d /install/usr/include/quickjs/ && \
    install -m644 quickjs/quickjs.h quickjs/quickjs-libc.h /install/usr/include/quickjs/ && \
    install -m644 quickjspp.hpp /install/usr/include

# 阶段3：编译libcron（独立层，仅当libcron源码变更时重建）
FROM builder-base AS builder-libcron
WORKDIR /build/libcron
RUN set -xe && \
    git clone https://github.com/PerMalmberg/libcron --depth=1 . && \
    mkdir -p /install/usr/lib && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j $THREADS && \
    # 安装到临时目录
    install -m644 libcron/out/Release/liblibcron.a /install/usr/lib/ && \
    install -d /install/usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /install/usr/include/libcron/ && \
    install -d /install/usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /install/usr/include/date/

# 阶段4：编译toml11（独立层，仅当toml11版本变更时重建）
FROM builder-base AS builder-toml11
WORKDIR /build/toml11
RUN set -xe && \
    git clone https://github.com/ToruNiina/toml11 --branch="v4.3.0" --depth=1 . && \
    cmake -DCMAKE_CXX_STANDARD=11 . && \
    make install -j $THREADS DESTDIR=/install

# 阶段5：编译subconverter（核心层，仅当subconverter源码/SHA变更时重建）
FROM builder-base AS builder-subconverter
ARG SHA=""
WORKDIR /build/subconverter

# 先拷贝所有依赖的编译产物（从各独立层拷贝）
COPY --from=builder-quickjspp /install/ /usr/
COPY --from=builder-libcron /install/ /usr/
COPY --from=builder-toml11 /install/ /

# 编译subconverter
RUN set -xe && \
    git clone https://github.com/metacubex/subconverter --depth=1 . && \
    # 处理SHA参数（保持原有逻辑）
    [ -n "$SHA" ] && sed -i 's/\(v[0-9]\.[0-9]\.[0-9]\)/\1-'"$SHA"'/' src/version.h || true && \
    # 安装python依赖并更新规则
    python3 -m ensurepip && \
    python3 -m pip install gitpython && \
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    # 编译subconverter
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j $THREADS && \
    # 准备最终产物
    install -d /install/usr/bin/ && \
    install -m755 subconverter /install/usr/bin/ && \
    cp -r base /install/base

# 阶段6：最终镜像（仅拷贝运行时产物，最小化）
FROM alpine:3.16
LABEL maintainer="metacubex.official@gmail.com"

# 安装运行时依赖（保持原依赖不变）
RUN apk add --no-cache --virtual subconverter-deps \
    pcre2 libcurl yaml-cpp

# 从编译层拷贝产物（仅运行时需要的文件）
COPY --from=builder-subconverter /install/usr/bin/subconverter /usr/bin/
COPY --from=builder-subconverter /install/base /base/

# 时区设置（保持原有逻辑）
ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone

# 工作目录和入口（保持原有逻辑）
WORKDIR /base
CMD ["subconverter"]

EXPOSE 25500/tcp
