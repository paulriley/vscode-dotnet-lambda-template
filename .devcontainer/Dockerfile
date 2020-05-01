FROM buildpack-deps:focal-curl AS basics

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG en_GB

# Do not exclude man pages & other documentation
RUN rm /etc/dpkg/dpkg.cfg.d/excludes

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        apt-utils \
        locales \
        lsb-release \
        software-properties-common \
        sudo \
    && locale-gen ${LANG} \
    && dpkg-reconfigure --frontend noninteractive locales \
    && rm "/etc/locale.gen"

ENV LANGUAGE en_GB:en
ENV LC_ALL ${LANG}

FROM basics AS add-sudo-user

ARG USERNAME
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID $USERNAME \
    && mkdir /home/$USERNAME \
    && chown -R $USER_UID:$USER_GID /home/$USERNAME \
    && usermod -a -G sudo ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers

FROM add-sudo-user AS dotnet-essentials

# Enable detection of running in a container
ENV DOTNET_RUNNING_IN_CONTAINER=true
# Enable correct mode for dotnet watch (only mode supported in a container)
ENV DOTNET_USE_POLLING_FILE_WATCHER=true
# Skip extraction of XML docs - generally not useful within an image/container - helps performance
ENV NUGET_XMLDOC_MODE=skip

# Install .NET CLI dependencies 
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        # .NET Core requirements
        libc6 \
        libgcc1 \
        libgcc-s1 \
        libgssapi-krb5-2 \
        libssl1.1 \
        libstdc++6 \
        libicu66 \
        zlib1g \
        -y --no-install-recommends \
    # Cleanup, to keep interim container small
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

FROM dotnet-essentials AS dotnet

ENV UNPACKFOLDER=/tmp/unpack

# Install .NET Core SDK
RUN dotnet_sdk_version=3.1.102 \
    && curl -SL --output dotnet.tar.gz https://dotnetcli.azureedge.net/dotnet/Sdk/$dotnet_sdk_version/dotnet-sdk-$dotnet_sdk_version-linux-x64.tar.gz \
    && dotnet_sha512='9cacdc9700468a915e6fa51a3e5539b3519dd35b13e7f9d6c4dd0077e298baac0e50ad1880181df6781ef1dc64a232e9f78ad8e4494022987d12812c4ca15f29' \
    && echo "$dotnet_sha512 dotnet.tar.gz" | sha512sum -c - \
    && mkdir -p ${UNPACKFOLDER} \
    && tar -ozxf dotnet.tar.gz -C ${UNPACKFOLDER} \
    && ls -la ${UNPACKFOLDER}

FROM dotnet-essentials AS required

# RUN curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | (OUT=$(apt-key add - 2>&1) || echo $OUT) \
    # && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" \
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        # zsh/docker/git requirements
        apt-transport-https \
        apt-utils \
        dialog \
        dnsutils \
        docker.io \
        gcc \
        git \
        git-man \
        gnupg-agent \
        groff \
        iproute2 \
        less \
        man-db \
        net-tools \
        procps \
        python3-pip \
        python3-setuptools \
        unzip \
        zsh \
        2>&1 \
    # Cleanup
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

FROM required AS man-install

# Reinstall all currently installed packages in order to get the man pages back
RUN apt-get update && \
    dpkg -l | grep ^ii | cut -d' ' -f3 | xargs apt-get install -y --reinstall && \
    rm -r /var/lib/apt/lists/*

FROM required AS terraform

RUN UNPACKFOLDER=/tmp/unpack \
    && TERRAFORM_VERSION=0.12.24 \
    && TERRAFORM_LSP_VERSION=0.0.11-beta1 \
    && TERRAFORM_AWS_PROVIDER_VERSION=2.59.0 \
    && mkdir -p ${UNPACKFOLDER}/lsp ${UNPACKFOLDER}/bin \
    && wget -c https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
    && unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d ${UNPACKFOLDER}/bin \
    && wget -c https://releases.hashicorp.com/terraform-provider-aws/${TERRAFORM_AWS_PROVIDER_VERSION}/terraform-provider-aws_${TERRAFORM_AWS_PROVIDER_VERSION}_linux_amd64.zip \
    && unzip terraform-provider-aws_${TERRAFORM_AWS_PROVIDER_VERSION}_linux_amd64.zip -d ${UNPACKFOLDER}/lsp \
    && echo "provider \"aws\" {}" >> ${UNPACKFOLDER}/lsp/providers.tf \
    && wget -c https://github.com/juliosueiras/terraform-lsp/releases/download/v${TERRAFORM_LSP_VERSION}/terraform-lsp_${TERRAFORM_LSP_VERSION}_linux_amd64.tar.gz -O - | tar -zx -C ${UNPACKFOLDER}/lsp/

FROM man-install as final

COPY --from=terraform /tmp/unpack/ /usr/share/terraform/
COPY --from=dotnet /tmp/unpack/ /usr/share/dotnet/

# Install Docker Compose CLI
RUN pip3 install wheel \
    && pip3 install docker-compose \
    # Install AWS CLI & Templates
    && pip3 install awscli \
    && pip3 install awscli-local \
    && ln -s /usr/share/terraform/bin/terraform /usr/bin/terraform \
    && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet \
    && dotnet new -i Amazon.Lambda.Templates

ENV DEBIAN_FRONTEND=
ENV CONTAINER_IMAGE_VER=1.0.0

ENTRYPOINT [ "sleep", "infinity" ]