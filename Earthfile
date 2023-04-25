VERSION 0.7
PROJECT applied-knowledge-systems/terraphim-cloud-dependencies
FROM ubuntu:18.04
ARG TARGETARCH
ARG TARGETOS
ARG TARGETPLATFORM
ARG --global tag=$TARGETOS-$TARGETARCH
ARG --global TARGETARCH
IF [ "$TARGETARCH" = amd64 ]
    ARG --global ARCH=x86_64
ELSE
    ARG --global ARCH=$TARGETARCH
END

publish-pipeline:
  PIPELINE --push
  TRIGGER push main
  TRIGGER pr main
  BUILD --allow-privileged +redismod

all:
    BUILD \
        --platform=linux/amd64 \
        --platform=linux/aarch64 \
        --platform=linux/arm/v7 \
        --platform=linux/arm/v6 

build-stack:
    FROM ghcr.io/applied-knowledge-systems/rgcluster:edge
    ENV DEBIAN_FRONTEND noninteractive
    ENV DEBCONF_NONINTERACTIVE_SEEN true
    RUN apt-get update && apt-get install -yqq --no-install-recommends --fix-missing git curl ca-certificates sudo gpg lsb-release
    RUN update-ca-certificates
    RUN curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
    RUN echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
    RUN apt-get update && apt-get install -y redis-stack-server
    SAVE IMAGE --push ghcr.io/applied-knowledge-systems/redis-stack:bionic


fetch-code:
    ENV DEBIAN_FRONTEND noninteractive
    ENV DEBCONF_NONINTERACTIVE_SEEN true
    RUN apt-get update && apt-get install -yqq --no-install-recommends git ca-certificates
    RUN update-ca-certificates
    RUN git clone --branch v2.6.3 --single-branch --recurse-submodules https://github.com/RediSearch/RediSearch RediSearch.git
    RUN git clone --branch v2.4.2 --single-branch --recurse-submodules https://github.com/RedisJSON/RedisJSON.git RedisJSON.git
    RUN git clone --branch v2.10.4 --single-branch --recurse-submodules https://github.com/RedisGraph/RedisGraph.git RedisGraph.git
    SAVE ARTIFACT RediSearch.git AS LOCAL RediSearch.git
    SAVE ARTIFACT RedisJSON.git AS LOCAL RedisJSON.git
    SAVE ARTIFACT RedisGraph.git AS LOCAL RedisGraph.git

build:
    FROM earthly/dind:ubuntu
    RUN apt-get update && apt-get install -yqq --no-install-recommends build-essential
    COPY +fetch-code/RediSearch.git RediSearch
    COPY +fetch-code/RedisJSON.git RedisJSON
    COPY +fetch-code/RedisGraph.git RedisGraph
    WITH DOCKER --pull redisfab/redis:6.2.10-x64-bionic --pull ghcr.io/applied-knowledge-systems/rgcluster:edge
         RUN cd RediSearch && make setup && make docker STATIC=1 OSNICK=bionic ARTIFACTS=1 PACK=1 REDIS_VER=6.2.6 && cd ../RedisJSON/ && make setup && make docker OSNICK=bionic ARTIFACTS=1 PACK=1 REDIS_VER=6.2.6 && cd ../RedisGraph && make docker OSNICK=bionic ARTIFACTS=1 PACK=1 REDIS_VER=6.2.6
    END
    SAVE ARTIFACT /RediSearch/bin/artifacts AS LOCAL RediSearchArtifacts
    SAVE ARTIFACT /RedisJSON/bin/artifacts AS LOCAL RedisJSONArtifacts
    SAVE ARTIFACT /RedisGraph/bin/artifacts AS LOCAL RedisGraphArtifacts

redisai:
    FROM redislabs/redisai:edge-cpu-bionic
    ENV REDIS_MODULES /usr/lib/redis/modules
    ENV LD_LIBRARY_PATH $REDIS_MODULES
    RUN mkdir -p $REDIS_MODULES/
    # COPY /usr/lib/redis/modules/ $REDIS_MODULES/
    # COPY /var/opt/redislabs/artifacts/ /var/opt/redislabs/artifacts
    SAVE ARTIFACT /usr/lib/redis/modules /modules 
    SAVE ARTIFACT /var/opt/redislabs/artifacts /artifacts

redismod:
    FROM ghcr.io/applied-knowledge-systems/redis-stack:bionic
    RUN apt-get update && apt-get install -y build-essential libgomp1 
    ENV LD_LIBRARY_PATH /usr/lib/redis/modules
    ENV REDISGEARS_MODULE_DIR /var/opt/redislabs/lib/modules
    ENV REDISGEARS_PY_DIR /var/opt/redislabs/modules/rg
    # COPY +redisai/artifacts /var/opt/redislabs/artifacts
    # COPY +redisai/modules /usr/lib/redis/modules
    ENV LD_LIBRARY_PATH /usr/lib/redis/modules
    COPY ./conf/redis_with_mods_stack.conf /etc/redis/redis.conf
    COPY ./conf/redis.service /etc/systemd/system/redis.service
    WORKDIR /cluster
    SAVE IMAGE --push ghcr.io/applied-knowledge-systems/redismod:bionic
    