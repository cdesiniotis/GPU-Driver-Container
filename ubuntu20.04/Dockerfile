ARG CUDA_VERSION
FROM nvidia/cuda:${CUDA_VERSION}-base-ubuntu20.04 as build

ARG TARGETARCH

# Args needed for builds using private runfiles
ARG DRIVER_VERSION
ARG PRIVATE_RUNFILE=false

SHELL ["/bin/bash", "-c"]

RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

# Remove cuda repository to avoid GPG errors
RUN rm /etc/apt/sources.list.d/cuda.list

RUN apt-get update && apt-get install -y --no-install-recommends \
        apt-utils \
        build-essential \
        ca-certificates \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*

ENV GOLANG_VERSION=1.20

# download appropriate binary based on the target architecture for multi-arch builds
RUN OS_ARCH=${TARGETARCH/x86_64/amd64} && OS_ARCH=${OS_ARCH/aarch64/arm64} && \
    curl https://storage.googleapis.com/golang/go${GOLANG_VERSION}.linux-${OS_ARCH}.tar.gz \
    | tar -C /usr/local -xz

ENV PATH /usr/local/go/bin:$PATH

WORKDIR /work

# Add any private runfiles
ADD NVIDIA-Linux-*-${DRIVER_VERSION}.run .

# Only include private runfile for the TARGETARCH
RUN if [ "$PRIVATE_RUNFILE" != "true" ]; then \
        rm *.run || true; \
    elif [ "$TARGETARCH" == "amd64" ]; then \
        chmod +x NVIDIA-Linux-x86_64-${DRIVER_VERSION}.run; \
        rm NVIDIA-Linux-aarch64-${DRIVER_VERSION}.run || true; \
    else \
        chmod +x NVIDIA-Linux-aarch64-${DRIVER_VERSION}.run; \
        rm NVIDIA-Linux-x86_64-${DRIVER_VERSION}.run || true; fi

RUN git clone https://gitlab.com/nvidia/container-images/driver && \
    cd driver/vgpu/src && \
    go build -o vgpu-util && \
    mv vgpu-util /work

FROM nvidia/cuda:${CUDA_VERSION}-base-ubuntu20.04

SHELL ["/bin/bash", "-c"]

ARG BASE_URL=https://us.download.nvidia.com/tesla
ARG TARGETARCH
ENV TARGETARCH=$TARGETARCH
ARG DRIVER_VERSION
ENV DRIVER_VERSION=$DRIVER_VERSION
ENV DEBIAN_FRONTEND=noninteractive

# Arg to indicate if driver type is either of passthrough(baremetal) or vgpu
ARG DRIVER_TYPE=passthrough
ENV DRIVER_TYPE=$DRIVER_TYPE
ARG DRIVER_BRANCH=525
ENV DRIVER_BRANCH=$DRIVER_BRANCH
ARG VGPU_LICENSE_SERVER_TYPE=NLS
ENV VGPU_LICENSE_SERVER_TYPE=$VGPU_LICENSE_SERVER_TYPE
# Enable vGPU version compability check by default
ARG DISABLE_VGPU_VERSION_CHECK=true
ENV DISABLE_VGPU_VERSION_CHECK=$DISABLE_VGPU_VERSION_CHECK
ENV NVIDIA_VISIBLE_DEVICES=void

# Args needed for builds using private runfiles
ARG PRIVATE_RUNFILE=false

RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections

RUN echo "TARGETARCH=$TARGETARCH"

ADD install.sh /tmp

# Fetch GPG keys for CUDA repo
RUN apt-key del 7fa2af80 && OS_ARCH=${TARGETARCH/amd64/x86_64} && OS_ARCH=${OS_ARCH/arm64/sbsa} && \
    apt-key adv --fetch-keys "https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/${OS_ARCH}/3bf863cc.pub"

RUN /tmp/install.sh reposetup && /tmp/install.sh depinstall && \
    curl -fsSL -o /usr/local/bin/donkey https://github.com/3XX0/donkey/releases/download/v1.1.0/donkey && \
    chmod +x /usr/local/bin/donkey

COPY nvidia-driver /usr/local/bin

COPY --from=build /work/vgpu-util /usr/local/bin
COPY --from=build /work/NVIDIA-Linux-*-${DRIVER_VERSION}.run drivers/

ADD drivers drivers/

# Fetch the installer automatically for passthrough/baremetal types
RUN if [ "$PRIVATE_RUNFILE" != "true" ] && [ "$DRIVER_TYPE" != "vgpu" ]; then \
    cd drivers && \
    /tmp/install.sh download_installer; fi

# Fabric manager packages are not available for arm64
RUN if [ "$PRIVATE_RUNFILE" != "true" ] && [ "$DRIVER_TYPE" != "vgpu" ] && [ "$TARGETARCH" != "arm64" ]; then \
    apt-get update && \
    apt-get install -y --no-install-recommends nvidia-fabricmanager-${DRIVER_BRANCH}=${DRIVER_VERSION}-1 \
    libnvidia-nscq-${DRIVER_BRANCH}=${DRIVER_VERSION}-1; fi

WORKDIR  /drivers

ARG PUBLIC_KEY=empty
COPY ${PUBLIC_KEY} kernel/pubkey.x509

# Install / upgrade packages here that are required to resolve CVEs
ARG CVE_UPDATES
RUN if [ -n "${CVE_UPDATES}" ]; then \
        apt-get update && apt-get upgrade -y ${CVE_UPDATES} && \
        rm -rf /var/lib/apt/lists/*; \
    fi

# Remove cuda repository to avoid GPG errors
RUN rm /etc/apt/sources.list.d/cuda.list

# Add NGC DL license from the CUDA image
RUN mkdir /licenses && mv /NGC-DL-CONTAINER-LICENSE /licenses/NGC-DL-CONTAINER-LICENSE

ENTRYPOINT ["nvidia-driver", "init"]
