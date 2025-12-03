# ======================================================================================
# DOCKERFILE MULTI-STAGE — GUIA PASSO A PASSO PARA INICIANTES
# --------------------------------------------------------------------------------------
# Este arquivo descreve, linha por linha, como empacotar uma aplicação Rust em duas fases:
#   1) Fase de CONSTRUÇÃO (“builder”): usa uma imagem completa com as ferramentas
#      necessárias para compilar o código em modo otimizado (release).
#   2) Fase de EXECUÇÃO (“runtime”): usa uma imagem mínima, contendo apenas o binário
#      já compilado e o essencial para rodar. Isso deixa a imagem final pequena, rápida
#      para baixar e com superfície de ataque reduzida (mais segura).
#
# A ideia do “multi-stage” é: compilar em um ambiente completo e, depois, copiar
# só o resultado (o binário final) para uma imagem enxuta. Assim, você não carrega
# ferramentas de desenvolvimento desnecessárias para produção.
# ======================================================================================

# Escolhe a imagem base para a FASE DE CONSTRUÇÃO e dá a ela o nome “builder”.
# Aqui usamos uma imagem oficial Rust sobre Alpine com tag totalmente fixada
# (versão específica de Rust e de Alpine), garantindo reprodutibilidade.
FROM rust:1.91.1-alpine3.22 AS builder

# Define o diretório de trabalho dentro do contêiner durante a construção.
# Pense nele como o “cd /usr/src/app”; tudo a seguir acontece a partir daqui.
WORKDIR /usr/src/app

# Instala os pacotes do sistema necessários para COMPILAR dependências nativas
# que algumas crates de Rust exigem. As versões dos pacotes estão fixadas para
# builds determinísticas:
#   - build-base=0.5-r3      → toolchain C completo (gcc, make, etc.).
#   - musl-dev=1.2.5-r10     → headers e libs de desenvolvimento da libc musl.
#   - openssl-dev=3.5.4-r0   → arquivos de desenvolvimento do OpenSSL.
# O uso de “pkg=versão” evita que futuras atualizações do repositório mudem o
# ambiente de compilação de forma inesperada.
RUN apk add --no-cache \
    build-base=0.5-r3 \
    musl-dev=1.2.5-r10 \
    openssl-dev=3.5.4-r0

# Cria o diretório-fonte padrão do Cargo. O compilador espera encontrar
# o arquivo principal em "src/main.rs".
RUN mkdir src

# Primeiro copiamos manifestos do projeto (Cargo.toml e Cargo.lock) para aproveitar
# o cache do Docker: se as dependências não mudarem, as próximas builds reutilizam
# a camada de “download/compilação” das crates, acelerando bastante a compilação.
COPY app/Cargo.toml app/Cargo.lock* ./

# Agora copiamos o código-fonte de fato (o main). Separar esta etapa da anterior
# garante melhor reuso de cache quando apenas o código muda, mas as dependências não.
COPY app/main.rs ./src/main.rs

# Compila o projeto em modo release (otimizado). O resultado ficará em
# /usr/src/app/target/release/. Esta etapa é a que mais consome tempo em builds frias.
RUN cargo build --release

# ======================================================================================
# A partir daqui começa a FASE DE EXECUÇÃO (“runtime”): imagem limpa e mínima.
# Nada do ambiente de compilação vem junto — só o binário final entrará aqui.
# ======================================================================================

# Usa uma imagem Alpine mínima com versão totalmente fixada (3.22.2) para rodar
# o binário. Isso garante que o ambiente de runtime seja estável e reproduzível.
FROM alpine:3.22.2 AS runtime

# Define uma variável de ambiente com o nome do binário. Centralizar o nome em
# uma variável torna mais fácil alterar o nome no futuro sem caçar em várias linhas.
ENV BIN_NAME=rust_api

# Desativa logs baseados em RUST_LOG (se sua aplicação/stack usar esse padrão).
# Útil em cenários de benchmark onde silêncio e previsibilidade interessam.
ENV RUST_LOG=off

# Instala certificados de autoridade (CA), curl e pg_activity com versões fixas:
#   - ca-certificates=20250911-r0  → certificados raiz para validação de HTTPS.
#   - curl=8.14.1-r2               → ferramenta de linha de comando para HTTP/HTTPS.
#   - pg_activity=3.5.1-r0         → monitor de atividade para PostgreSQL
#                                   (útil para debugging/observabilidade dentro do contêiner).
# As versões travadas evitam mudanças inesperadas de comportamento em ambientes
# de produção e de benchmark.
RUN apk add --no-cache \
    ca-certificates=20250911-r0 \
    curl=8.14.1-r2 \
    pg_activity=3.5.1-r0

# Copia APENAS o binário final da fase “builder” para esta imagem minimalista.
# Repare no "--from=builder": é assim que “puxamos” artefatos do estágio anterior.
# O destino /usr/local/bin é um local padrão para executáveis instalados manualmente.
COPY --from=builder /usr/src/app/target/release/${BIN_NAME} /usr/local/bin/

# Garante permissão de execução ao arquivo do binário. Em Linux, o bit “x” precisa
# estar setado para que o sistema reconheça o arquivo como programa executável.
RUN chmod +x /usr/local/bin/${BIN_NAME}

# Troca o usuário efetivo para “nobody” (sem privilégios). Boa prática de segurança:
# mesmo que ocorra uma falha no app, o atacante não terá poderes administrativos.
USER nobody

# Define o comando padrão ao iniciar o contêiner em produção. Diferente de RUN (que
# executa na construção), o CMD roda quando o contêiner “sobe”. Aqui, executamos
# diretamente o binário instalado anteriormente.
CMD /usr/local/bin/${BIN_NAME}
