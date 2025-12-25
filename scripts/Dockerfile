# --- Stage 1: Base Environment ---
FROM alpine:3.16 AS base
RUN apk add --no-cache git g++ build-base linux-headers cmake python3 \
    curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

# --- Stage 2: Build QuickJS++ ---
FROM base AS quickjs-builder
ARG THREADS="4"
WORKDIR /src
RUN git clone https://github.com/ftk/quickjspp --depth=1 && \
    cd quickjspp && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make quickjs -j $THREADS && \
    install -d /usr/lib/quickjs/ && \
    install -m644 quickjs/libquickjs.a /usr/lib/quickjs/ && \
    install -d /usr/include/quickjs/ && \
    install -m644 quickjs/quickjs.h quickjs/quickjs-libc.h /usr/include/quickjs/ && \
    install -m644 quickjspp.hpp /usr/include

# --- Stage 3: Build libcron ---
FROM base AS libcron-builder
ARG THREADS="4"
WORKDIR /src
RUN git clone https://github.com/PerMalmberg/libcron --depth=1 && \
    cd libcron && \
    git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j $THREADS && \
    install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
    install -d /usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
    install -d /usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /usr/include/date/

# --- Stage 4: Build toml11 ---
FROM base AS toml11-builder
ARG THREADS="4"
WORKDIR /src
RUN git clone https://github.com/ToruNiina/toml11 --branch="v4.3.0" --depth=1 && \
    cd toml11 && \
    cmake -DCMAKE_CXX_STANDARD=11 . && \
    make install -j $THREADS

# --- Stage 5: Final Compilation (Subconverter) ---
FROM base AS builder
ARG THREADS="4"
ARG SHA=""

# 从之前的阶段拷贝已编译好的库和头文件
COPY --from=quickjs-builder /usr/lib/quickjs/ /usr/lib/quickjs/
COPY --from=quickjs-builder /usr/include/quickjs/ /usr/include/quickjs/
COPY --from=quickjs-builder /usr/include/quickjspp.hpp /usr/include/

COPY --from=libcron-builder /usr/lib/liblibcron.a /usr/lib/
COPY --from=libcron-builder /usr/include/libcron/ /usr/include/libcron/
COPY --from=libcron-builder /usr/include/date/ /usr/include/date/

COPY --from=toml11-builder /usr/local/include/toml11 /usr/local/include/toml11
COPY --from=toml11-builder /usr/local/lib/cmake/toml11 /usr/local/lib/cmake/toml11

WORKDIR /subconverter
RUN git clone https://github.com/metacubex/subconverter . --depth=1 && \
    [ -n "$SHA" ] && sed -i 's/\(v[0-9]\.[0-9]\.[0-9]\)/\1-'"$SHA"'/' src/version.h || true && \
    python3 -m ensurepip && \
    python3 -m pip install --no-cache-dir gitpython && \
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j $THREADS

# --- Stage 6: Runtime Image ---
FROM alpine:3.16
LABEL maintainer="metacubex.official@gmail.com"

# 只安装运行时依赖
RUN apk add --no-cache pcre2 libcurl yaml-cpp tzdata

# 从编译阶段拷贝二进制文件和基础配置
# 注意：这里使用了显式路径以确保安全性
COPY --from=builder /subconverter/subconverter /usr/bin/subconverter
COPY --from=builder /subconverter/base /base/

ENV TZ=Africa/Abidjan
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone

WORKDIR /base
EXPOSE 25500/tcp

CMD ["subconverter"]
