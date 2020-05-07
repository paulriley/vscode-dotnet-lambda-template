FROM buildpack-deps:bionic AS basics

ARG UNPACK_FOLDER=/tmp/unpack

ENV UNPACK_FOLDER=${UNPACK_FOLDER}
ENV DEBIAN_FRONTEND=noninteractive
ENV LANG en_GB.UTF-8

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
        libgssapi-krb5-2 \
        libicu60 \
        libssl1.1 \
        libstdc++6 \
        zlib1g \
        -y --no-install-recommends \
    # Cleanup, to keep interim container small
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

FROM dotnet-essentials AS dotnet

# Install .NET Core SDK
RUN DOTNET_SDK_VERSION=3.1.102 \
    && curl -SL --output dotnet.tar.gz https://dotnetcli.azureedge.net/dotnet/Sdk/$DOTNET_SDK_VERSION/dotnet-sdk-$DOTNET_SDK_VERSION-linux-x64.tar.gz \
    && dotnet_sha512='9cacdc9700468a915e6fa51a3e5539b3519dd35b13e7f9d6c4dd0077e298baac0e50ad1880181df6781ef1dc64a232e9f78ad8e4494022987d12812c4ca15f29' \
    && echo "$dotnet_sha512 dotnet.tar.gz" | sha512sum -c - \
    && mkdir -p ${UNPACK_FOLDER} \
    && tar -ozxf dotnet.tar.gz -C ${UNPACK_FOLDER} \
    && ls -la ${UNPACK_FOLDER}

FROM dotnet-essentials AS required

RUN curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | (OUT=$(apt-key add - 2>&1) || echo $OUT) \
    && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        # zsh/docker/git requirements
        apt-transport-https \
        apt-utils \
        ca-certificates \
        curl \
        dialog \
        dnsutils \
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
        # Install Docker CLI
        docker-ce-cli \
    # Cleanup
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

FROM required AS terraform

RUN TERRAFORM_VERSION=0.12.24 \
    && TERRAFORM_LSP_VERSION=0.0.11-beta1 \
    && TERRAFORM_AWS_PROVIDER_VERSION=2.59.0 \
    && mkdir -p ${UNPACK_FOLDER}/lsp ${UNPACK_FOLDER}/bin \
    && wget -c https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
    && unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d ${UNPACK_FOLDER}/bin \
    && wget -c https://releases.hashicorp.com/terraform-provider-aws/${TERRAFORM_AWS_PROVIDER_VERSION}/terraform-provider-aws_${TERRAFORM_AWS_PROVIDER_VERSION}_linux_amd64.zip \
    && unzip terraform-provider-aws_${TERRAFORM_AWS_PROVIDER_VERSION}_linux_amd64.zip -d ${UNPACK_FOLDER}/lsp \
    && echo "provider \"aws\" {}" >> ${UNPACK_FOLDER}/lsp/providers.tf \
    && wget -c https://github.com/juliosueiras/terraform-lsp/releases/download/v${TERRAFORM_LSP_VERSION}/terraform-lsp_${TERRAFORM_LSP_VERSION}_linux_amd64.tar.gz -O - | tar -zx -C ${UNPACK_FOLDER}/lsp/

FROM required AS gh

RUN GH_VERSION=0.7.0 \
    && mkdir -p ${UNPACK_FOLDER} \
    && wget -c https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_linux_amd64.tar.gz -O - | tar -zx --strip-components=1 -C ${UNPACK_FOLDER}/

FROM required as final

COPY --from=terraform ${UNPACK_FOLDER} /usr/share/terraform/
COPY --from=dotnet ${UNPACK_FOLDER} /usr/share/dotnet/
COPY --from=gh ${UNPACK_FOLDER} /usr/share/gh/

# Install Docker Compose CLI
RUN pip3 install wheel \
    && pip3 install docker-compose \
    # Install AWS CLI & Templates
    && pip3 install awscli \
    && pip3 install awscli-local \
    && ln -s /usr/share/terraform/bin/terraform /usr/bin/terraform \
    && ln -s /usr/share/dotnet/dotnet /usr/bin/dotnet \
    && ln -s /usr/share/gh/bin/gh /usr/bin/gh \
    && dotnet new -i Amazon.Lambda.Templates

ENV DEBIAN_FRONTEND=
ENV CONTAINER_IMAGE_VER=1.0.0

ENTRYPOINT [ "sleep", "infinity" ]