# syntax=docker/dockerfile:1

# 可选：通过 --build-arg BASE_IMAGE=... 覆盖默认基础镜像
ARG BASE_IMAGE=dqzboy/mirror-hub:latest
FROM ${BASE_IMAGE}

LABEL maintainer="longshu"
LABEL description="Docker Hub proxy/cache registry"

COPY ./config.yml /etc/distribution/config.yml

EXPOSE 5000
VOLUME ["/var/lib/registry"]

