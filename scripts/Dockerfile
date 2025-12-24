# ==========================================
# 阶段 1: 基础构建环境
# ==========================================
FROM alpine:3.16 AS base
RUN apk add --no-cache git g++ build-base linux-headers cmake python3 curl-dev rapidjson-dev pcre2-dev yaml-cpp-dev

# ==========================================
# 阶段 2: 编译 QuickJS (耗时较长，缓存)
# ==========================================
FROM base AS quickjs_builder
RUN git clone github.com --depth=1 && \
    cd quickjspp && git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make quickjs -j $(nproc) && \
    install -d /usr/lib/quickjs/ && \
    install -m644 quickjs/libquickjs.a /usr/lib/quickjs/ && \
    install -d /usr/include/quickjs/ && \
    install -m644 quickjs/quickjs.h quickjs/quickjs-libc.h /usr/include/quickjs/ && \
    install -m644 quickjspp.hpp /usr/include

# ==========================================
# 阶段 3: 编译 libcron (缓存)
# ==========================================
FROM base AS libcron_builder
RUN git clone github.com --depth=1 && \
    cd libcron && git submodule update --init && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make libcron -j $(nproc) && \
    install -m644 libcron/out/Release/liblibcron.a /usr/lib/ && \
    install -d /usr/include/libcron/ && \
    install -m644 libcron/include/libcron/* /usr/include/libcron/ && \
    install -d /usr/include/date/ && \
    install -m644 libcron/externals/date/include/date/* /usr/include/date/

# ==========================================
# 阶段 4: 编译 subconverter (频繁变动层)
# ==========================================
FROM base AS builder
# 拷贝前序阶段编译好的静态库和头文件
COPY --from=quickjs_builder /usr/lib/quickjs/ /usr/lib/quickjs/
COPY --from=quickjs_builder /usr/include/quickjs/ /usr/include/quickjs/
COPY --from=quickjs_builder /usr/include/quickjspp.hpp /usr/include/
COPY --from=libcron_builder /usr/lib/liblibcron.a /usr/lib/
COPY --from=libcron_builder /usr/include/libcron/ /usr/include/libcron/
COPY --from=libcron_builder /usr/include/date/ /usr/include/date/

# 编译 toml11 (较快，也可独立，此处放在 builder 顶层)
RUN git clone github.com --branch="v4.3.0" --depth=1 && \
    cd toml11 && cmake -DCMAKE_CXX_STANDARD=11 . && make install -j $(nproc)

# 拷贝本地源码 (如果你是在本地修改源码编译，请确保 Dockerfile 在项目根目录)
WORKDIR /subconverter
COPY . .

# 更新规则并编译
RUN python3 -m ensurepip && \
    python3 -m pip install gitpython && \
    python3 scripts/update_rules.py -c scripts/rules_config.conf && \
    cmake -DCMAKE_BUILD_TYPE=Release . && \
    make -j $(nproc)

# ==========================================
# 阶段 5: 最终运行镜像 (最小化)
# ==========================================
FROM alpine:3.16
LABEL maintainer="metacubex.official@gmail.com"

# 仅安装运行必需的动态库
RUN apk add --no-cache pcre2 libcurl yaml-cpp tzdata

# 从 builder 拷贝二进制文件和基础配置
COPY --from=builder /subconverter/subconverter /usr/bin/
COPY --from=builder /subconverter/base /base/

ENV TZ=Asia/Shanghai
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /base
EXPOSE 25500/tcp

# 启动命令
CMD ["subconverter"]
