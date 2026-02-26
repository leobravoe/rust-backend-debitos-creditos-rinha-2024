# ======================================================================================
# DOCKERFILE MULTI-STAGE — GUIA PASSO A PASSO PARA INICIANTES
# --------------------------------------------------------------------------------------
# (conteúdo inalterado)
# ======================================================================================

FROM rust:1.91.1-alpine3.22 AS builder

WORKDIR /usr/src/app

RUN apk add --no-cache \
    build-base=0.5-r3 \
    musl-dev=1.2.5-r10 \
    openssl-dev=3.5.5-r0

RUN mkdir src

COPY app/Cargo.toml app/Cargo.lock* ./

COPY app/main.rs ./src/main.rs

RUN cargo build --release

# ======================================================================================
# Runtime
# ======================================================================================

FROM alpine:3.22.3 AS runtime

ENV BIN_NAME=rust_api
ENV RUST_LOG=off

RUN apk add --no-cache \
    ca-certificates=20250911-r0 \
    curl=8.14.1-r2 \
    pg_activity=3.5.1-r0

COPY --from=builder /usr/src/app/target/release/${BIN_NAME} /usr/local/bin/

RUN chmod +x /usr/local/bin/${BIN_NAME}

USER nobody

CMD /usr/local/bin/${BIN_NAME}