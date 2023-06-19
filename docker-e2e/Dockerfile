ARG base_image=alpine:latest
FROM golang:alpine as build

ARG TARGET_ARCH
ENV GO_ARCH=${TARGET_ARCH}

COPY ./src/terraform-resource /build/src
RUN cd /build/src && \
  go build -o "/build/bin/check" cmd/check/main.go && \
  go build -o "/build/bin/in" cmd/in/main.go && \
  go build -o "/build/bin/out" cmd/out/main.go

FROM ${base_image}

ARG TARGET_ARCH
ARG TERRAFORM_VERSION

RUN apk update && \
  apk add bash ca-certificates curl git gnupg openssh-client unzip

SHELL [ "/bin/bash", "-c" ]
RUN mkdir -p $HOME/.ssh && \
  echo "StrictHostKeyChecking no" >> $HOME/.ssh/config && \
  echo "LogLevel quiet" >> $HOME/.ssh/config && \
  chmod 0600 $HOME/.ssh/config

# this glibc compatibility module is needed for some downloaded binaries,
# such as aws cli, to run in provisioners.
# Steps to build glibc for ARM64:
# https://gist.github.com/ljfranklin/f6a7dfcfb2c573d4a276dad59e58db82
# TODO: Switch back to upstream for ARM64 once supported:
# https://github.com/sgerrand/alpine-pkg-glibc/issues/126
# TODO: Remove --force-overwrite once
# https://github.com/sgerrand/alpine-pkg-glibc/issues/185 is fixed
RUN if [ "${TARGET_ARCH}" = "amd64" ]; then \
  wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
  wget -q -O /tmp/glibc.apk https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.32-r0/glibc-2.32-r0.apk && \
  wget -q -O /tmp/glibc-bin.apk https://github.com/sgerrand/alpine-pkg-glibc/releases/download/2.32-r0/glibc-bin-2.32-r0.apk; \
  elif [ "${TARGET_ARCH}" = "arm64" ]; then \
  wget -q -O /etc/apk/keys/ljfranklin-glibc.pub https://github.com/ljfranklin/alpine-pkg-glibc/releases/download/2.32-r0-arm64/ljfranklin-glibc.pub && \
  wget -q -O /tmp/glibc.apk https://github.com/ljfranklin/alpine-pkg-glibc/releases/download/2.32-r0-arm64/glibc-2.32-r0.apk && \
  wget -q -O /tmp/glibc-bin.apk https://github.com/ljfranklin/alpine-pkg-glibc/releases/download/2.32-r0-arm64/glibc-bin-2.32-r0.apk; \
  fi; \
  apk add --force-overwrite /tmp/glibc.apk && \
  apk add /tmp/glibc-bin.apk && \
  rm -rf /tmp/glibc.apk && \
  rm -rf /tmp/glibc-bin.apk

RUN mkdir -p /tmp/terraform-fetch && \
  pushd /tmp/terraform-fetch > /dev/null && \
  \
  wget -O terraform_${TERRAFORM_VERSION}_linux_${TARGET_ARCH}.zip https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_${TARGET_ARCH}.zip && \
  wget -O terraform_SHA256SUMS https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_SHA256SUMS && \
  wget -O terraform_SHA256SUMS.sig https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_SHA256SUMS.sig && \
  \ 
  # verify checksum file with Hashicorp public key
  # https://www.hashicorp.com/security.html
  gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 72D7468F && \
  gpg --verify terraform_SHA256SUMS.sig terraform_SHA256SUMS && \
  grep terraform_${TERRAFORM_VERSION}_linux_${TARGET_ARCH}.zip terraform_SHA256SUMS | sha256sum -c - && \
  \
  unzip terraform_${TERRAFORM_VERSION}_linux_${TARGET_ARCH}.zip -d "/usr/local/bin" && \
  terraform --version && \
  \
  popd > /dev/null && \
  rm -rf /tmp/terraform-fetch

RUN mkdir -p /usr/local/share/terraform/plugins/github.com/ashald/stateful/1.2.0/linux_${TARGET_ARCH}/ && \
  wget -O /usr/local/share/terraform/plugins/github.com/ashald/stateful/1.2.0/linux_${TARGET_ARCH}/terraform-provider-stateful_v1.2.0 "https://github.com/ashald/terraform-provider-stateful/releases/download/v1.2.0/terraform-provider-stateful_v1.2.0-linux-${TARGET_ARCH}" && \
  chmod +x /usr/local/share/terraform/plugins/github.com/ashald/stateful/1.2.0/linux_${TARGET_ARCH}/terraform-provider-stateful_v1.2.0

COPY --from=build /build/bin/check /build/bin/in /build/bin/out /opt/resource/
