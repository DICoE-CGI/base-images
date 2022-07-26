FROM ubuntu:20.04 as builder

ARG user=indy
ENV HOME="/home/$user"
WORKDIR $HOME
RUN mkdir -p .local/bin .local/etc .local/lib


ENV TZ=UTC
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone


# Install environment
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        automake \
        bison \
        build-essential \
        ca-certificates \
        cmake \
        curl \
        flex \
        git \
        libbz2-dev \
        libffi-dev \
        libgmp-dev \
        liblzma-dev \
        libncurses5-dev \
        libncursesw5-dev \
        libreadline-gplv2-dev \
        libsecp256k1-dev \
        libsqlite3-dev \
        libssl-dev \
        libtool \
        libzmq3-dev \
        pkg-config \
        zlib1g-dev && \
    rm -rf /var/lib/apt/lists/*

ARG nacl_lib_ver=1.0.18

# Build and install libsodium library
RUN curl -o libsodium-${nacl_lib_ver}.tar.gz \
        "https://download.libsodium.org/libsodium/releases/libsodium-${nacl_lib_ver}.tar.gz" && \
    tar xzvf libsodium-${nacl_lib_ver}.tar.gz && \
    cd libsodium-${nacl_lib_ver} && \
    CFLAGS="-Os" ./configure && \
    make install && \
    cd .. && \
    cp -a /usr/local/lib/libsodium.so* "$HOME/.local/lib" && \
    rm -rf libsodium-${nacl_lib_ver}*

# 3.5.3+ required
ARG python_version=3.10.5

# Install pyenv and python to user home directory
ENV PYENV_ROOT="$HOME/.pyenv"
ENV PATH="$PYENV_ROOT/shims:$PYENV_ROOT/bin:$PATH"
RUN git clone https://github.com/pyenv/pyenv.git "$PYENV_ROOT" && \
    CFLAGS="-O2" pyenv install $python_version && \
    pyenv global $python_version && \
    find "$PYENV_ROOT" -depth \
		\( \
			\( -type d -a \( -name test -o -name tests \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' + && \
	pip install --no-cache-dir --upgrade pip

# set to --release for smaller, optimized library
ARG indy_build_flags=--release

# overridden by the build script
# version="b4b330ef326958d593ab42e25679c2dcd655494c"
ARG sdk_version
ENV SDK_VER=$sdk_version
RUN echo $SDK_VER
ARG indy_sdk_url=https://codeload.github.com/hyperledger/indy-sdk/tar.gz/$SDK_VER
RUN echo $indy_sdk_url

# ARG indy_sdk_url=https://codeload.github.com/hyperledger/indy-sdk/tar.gz/b4b330ef326958d593ab42e25679c2dcd655494c

ARG rust_version=1.62.1

ENV LIBRARY_PATH="$HOME/.local/lib:$LIBRARY_PATH"

# Install rust toolchain
RUN curl -o rustup https://sh.rustup.rs && \
    chmod +x rustup && \
    ./rustup --default-toolchain "${rust_version}" -y
ENV PATH="$HOME/.cargo/bin:$PATH"

# Download and extract indy-sdk
RUN mkdir indy-sdk && \
    curl "${indy_sdk_url}" | tar -xz -C indy-sdk

# Build and install indy-sdk
WORKDIR $HOME/indy-sdk
RUN cd indy-sdk*/libindy && \
    cargo build ${indy_build_flags} && \
    cp target/*/libindy.so "$HOME/.local/lib" && \
    cargo clean

# Package and install python3-indy
RUN tar czvf python3-indy.tgz -C indy-sdk*/wrappers/python . && \
    pip install --no-cache-dir python3-indy.tgz && \
    rm python3-indy.tgz

# Build and install indy-cli
RUN cd indy-sdk*/cli && \
    cargo build ${indy_build_flags} && \
    cp target/*/indy-cli "$HOME/.local/bin" && \
    cargo clean

# grab the latest sdk code for the postgres plug-in
WORKDIR $HOME
ARG indy_postgres_url=${indy_sdk_url}
RUN mkdir indy-postgres && \
    curl "${indy_postgres_url}" | tar -xz -C indy-postgres

# Build and install postgres_storage plugin
WORKDIR $HOME/indy-postgres
RUN cd indy-sdk*/experimental/plugins/postgres_storage && \
    cargo build ${indy_build_flags} && \
    cp target/*/libindystrgpostgres.so "$HOME/.local/lib" && \
    cargo clean

# Clean up
WORKDIR $HOME
RUN rm -rf .cargo .multirust .profile .rustup rustup indy-sdk indy-postgres

# Install indy python libraries and other dependencies
ADD requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


## Start fresh (new image) ##
FROM ubuntu:20.04


ARG uid=1001
ARG user=indy
ARG python_version=3.10.5
ARG tag_name=von-image
ARG tag_version=1.16.0

ENV HOME="/home/$user" \
    APP_ROOT="$HOME" \
    LC_ALL=C.UTF-8 \
    LANG=C.UTF-8 \
    PIP_NO_CACHE_DIR=off \
    PYTHON_VERSION="$python_version" \
    PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=UTF-8 \
    RUST_LOG=warning \
    SHELL=/bin/bash \
    SUMMARY="von-image including Python $python_version and indy-sdk" \
    DESCRIPTION="von-image provides a consistent base image for running VON python web \
components. Based on Ubuntu bionic, this image includes Python $python_version, \
 indy-sdk, and supporting Python libraries."

LABEL summary="$SUMMARY" \
      description="$DESCRIPTION" \
      io.k8s.description="$DESCRIPTION" \
      io.k8s.display-name="von-image $tag_version" \
      name="$tag_name" \
      version="$tag_version" \
      maintainer=""

# Add indy user
RUN useradd -U -ms /bin/bash -u $uid $user

# Install environment
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        apt-transport-https \
        ca-certificates \
        bzip2 \
        curl \
        git \
        libffi7 \
        libgmp10 \
        liblzma5 \
        libncurses5 \
        libncursesw5 \
        libreadline5 \
        libsecp256k1-0 \
        libzmq5 \
        net-tools \
        openssl \
        sqlite3 \
        vim-tiny \
        zlib1g && \
    rm -rf /var/lib/apt/lists/* /usr/share/doc/*

WORKDIR $HOME

# Copy build results
COPY --from=builder --chown=indy:indy $HOME .

COPY --chown=indy:indy bin/ $HOME/.local/bin/
RUN chmod ug+rwx $HOME/.local/bin/*

# Support standard python paths used in scripts
ENV PYENV_ROOT="$HOME/.pyenv"
RUN ln -s "$PYENV_ROOT/shims/python" /usr/bin/python && \
    ln -s "$PYENV_ROOT/shims/python3" /usr/bin/python3 && \
    ln -s "$PYENV_ROOT/shims/pip" /usr/bin/pip && \
    ln -s "$PYENV_ROOT/shims/pip3" /usr/bin/pip3

# Add selected version of python and local bin directories
ENV PATH="$HOME/.local/bin:$HOME/bin:$PYENV_ROOT/shims:$PYENV_ROOT/bin:$PATH"

# Make libraries resolvable by python
ENV LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
RUN echo "$HOME/.local/lib" > /etc/ld.so.conf.d/local.conf && ldconfig

# - In order to drop the root user, we have to make some directories writable
#   to the root group as OpenShift default security model is to run the container
#   under random UID.
RUN usermod -a -G 0 indy

# Create standard directories to allow volume mounting and set permissions
# Note: PIP_NO_CACHE_DIR environment variable should be cleared to allow caching
RUN mkdir -p \
        $HOME/.cache/pip/http \
        $HOME/.indy-cli/networks \
        $HOME/.indy_client/wallet \
        $HOME/.indy_client/pool \
        $HOME/.indy_client/ledger-cache \
        $HOME/ledger/sandbox/data \
        $HOME/log

# The root group needs access the directories under $HOME/.indy_client for the
# container to function in OpenShift.
RUN chown -R indy:root $HOME/.indy_client \
    && chmod -R ug+rw $HOME/log $HOME/ledger $HOME/.indy-cli $HOME/.indy_client $HOME/.cache

USER $user

CMD ["bash"]
