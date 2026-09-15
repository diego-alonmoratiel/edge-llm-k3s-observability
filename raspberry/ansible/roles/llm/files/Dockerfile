# Multi-stage build that compiles llama-server from source with -DGGML_NATIVE=ON,
# so the binary is optimised for the Raspberry Pi's CPU (Cortex-A76).
# The result is imported into k3s' containerd on the worker, so no registry is needed.
ARG DEBIAN_VERSION=bookworm

FROM debian:${DEBIAN_VERSION}-slim AS build
ARG LLAMA_CPP_REF=master
# Lower the job count if the build runs out of memory on the 4 GB Pi.
ARG BUILD_JOBS=2

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates git cmake build-essential libgomp1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN git clone --depth 1 --branch "${LLAMA_CPP_REF}" https://github.com/ggml-org/llama.cpp .

RUN cmake -B build \
        -DCMAKE_BUILD_TYPE=Release \
        -DGGML_NATIVE=ON \
        -DLLAMA_CURL=OFF \
        -DLLAMA_BUILD_TESTS=OFF \
        -DLLAMA_BUILD_EXAMPLES=OFF \
    && cmake --build build --target llama-server -j"${BUILD_JOBS}"

FROM debian:${DEBIAN_VERSION}-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends libgomp1 curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=build /src/build/bin/llama-server /usr/local/bin/llama-server

ENV LLAMA_ARG_HOST=0.0.0.0
EXPOSE 8080
HEALTHCHECK CMD ["curl", "-f", "http://localhost:8080/health"]
ENTRYPOINT ["/usr/local/bin/llama-server"]
