FROM ubuntu:22.04 as build

SHELL ["/bin/bash", "-c"]

ARG GOPROXY=direct
ARG DEBIAN_FRONTEND=noninteractive

WORKDIR /tmp

RUN apt-get update && \
    apt-get -y install \
    build-essential \
    gnupg \
    ca-certificates \
    ninja-build \
    git \
    openjdk-11-jdk-headless \
    default-jdk \
    libssl-dev \
    liblz4-dev \
    curl \
    zlib1g-dev \
    cmake \
    autoconf \
    libtool \
    unzip \
    libiberty-dev \
    python3-distutils \
    libpython3-dev \
    libjemalloc-dev \
    distcc \
    valgrind \
    xsltproc

RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
RUN echo "deb https://download.mono-project.com/repo/ubuntu stable-focal main" | tee /etc/apt/sources.list.d/mono-official-stable.list
RUN apt update && apt -y install mono-devel

# build/install liburing
RUN curl -Ls https://github.com/axboe/liburing/archive/refs/tags/liburing-2.1.tar.gz -o liburing.tar.gz && \
    echo "f1e0500cb3934b0b61c5020c3999a973c9c93b618faff1eba75aadc95bb03e07  liburing.tar.gz" > liburing-sha.txt && \
    sha256sum --quiet -c liburing-sha.txt && \
    mkdir liburing && \
    tar --strip-components 1 --no-same-owner --directory liburing -xf liburing.tar.gz && \
    cd liburing && \
    ./configure && \
    make -j$(nproc) -C src V=1 CPPFLAGS="-Werror" CFLAGS="-g -O2 -Wall -Wextra -Werror" CXXFLAGS="-g -O2 -Wall -Wextra -Werror" && \
    make install && \
    cd ../ && \
    rm -rf /tmp/*


# build/install LLVM
# compiler-rt, libcxx and libcxxabi can't be built with gcc<11
#     ref: https://libcxx.llvm.org/#platform-and-compiler-support)
# so build and install clang first, then build other components and with clang
# build clang a second time to pass component check
RUN curl -Ls https://github.com/llvm/llvm-project/releases/download/llvmorg-13.0.0/llvm-project-13.0.0.src.tar.xz -o llvm.tar.xz && \
    echo "6075ad30f1ac0e15f07c1bf062c1e1268c241d674f11bd32cdf0e040c71f2bf3  llvm.tar.xz" > llvm-sha.txt && \
    sha256sum --quiet -c llvm-sha.txt && \
    mkdir llvm-project && \
    tar --strip-components 1 --no-same-owner --directory llvm-project -xf llvm.tar.xz && \
    cd llvm-project && \
    mkdir -p build && \
    cd build && \
    cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -G Ninja \
    -Wno-dev \
    -DLLVM_INCLUDE_EXAMPLES=OFF \
    -DLLVM_INCLUDE_TESTS=OFF \
    -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;lldb" \
    -DLLVM_STATIC_LINK_CXX_STDLIB=ON \
    ../llvm && \
    cmake --build . && \
    cmake --build . --target install && \
    cd .. && \
    rm -rf build && \
    mkdir build && \
    cd build && \
    cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -G Ninja \
    -Wno-dev \
    -DLLVM_INCLUDE_EXAMPLES=OFF \
    -DLLVM_INCLUDE_TESTS=OFF \
    -DLLVM_ENABLE_PROJECTS="clang;compiler-rt;libcxx;libcxxabi;libunwind" \
    -DLLVM_STATIC_LINK_CXX_STDLIB=ON \
    -DCMAKE_C_COMPILER=/usr/local/bin/clang \
    -DCMAKE_CXX_COMPILER=/usr/local/bin/clang++ \
    ../llvm && \
    cmake --build . && \
    cmake --build . --target install && \
    cd ../.. && \
    rm -rf /tmp/*

# install golang 1.18
RUN if [ "$(uname -m)" == "aarch64" ]; then \
    GOLANG_ARCH="arm64"; \
    GOLANG_SHA256="beacbe1441bee4d7978b900136d1d6a71d150f0a9bb77e9d50c822065623a35a"; \
    else \
    GOLANG_ARCH="amd64"; \
    GOLANG_SHA256="956f8507b302ab0bb747613695cdae10af99bbd39a90cae522b7c0302cc27245"; \
    fi && \
    curl -Ls https://go.dev/dl/go1.18.3.linux-${GOLANG_ARCH}.tar.gz -o golang.tar.gz && \
    echo "${GOLANG_SHA256}  golang.tar.gz" > golang-sha.txt && \
    sha256sum --quiet -c golang-sha.txt && \
    tar --directory /usr/local -xf golang.tar.gz && \
    echo '[ -x /usr/local/go/bin/go ] && export GOROOT=/usr/local/go && export GOPATH=$HOME/go && export PATH=$GOPATH/bin:$GOROOT/bin:$PATH' >> /etc/profile.d/golang.sh && \
    echo 'source /etc/profile.d/golang.sh' >> /root/.bashrc && \
    rm -rf /tmp/*

# build/install boringssl
RUN source /etc/profile.d/golang.sh && \
    mkdir -p /opt/boringssl && \
    cd /opt/boringssl && \
    git clone https://boringssl.googlesource.com/boringssl . && \
    git checkout e796cc65025982ed1fb9ef41b3f74e8115092816 && \
    for file in crypto/fipsmodule/rand/fork_detect_test.cc include/openssl/bn.h ssl/test/bssl_shim.cc; do \
    perl -p -i -e 's/#include <inttypes.h>/#define __STDC_FORMAT_MACROS 1\n#include <inttypes.h>/g;' $file; \
    done && \
    perl -p -i -e 's/-Werror/-Werror -fPIC/' CMakeLists.txt && \
    git diff && \
    mkdir build && \
    cd build && \
    CC=clang CXX=clang++ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release .. && \
    ninja && \
    ./ssl/ssl_test && \
    mkdir -p /opt/boringssl/lib && \
    cp crypto/libcrypto.a ssl/libssl.a /opt/boringssl/lib/

# build/install wolfssl
RUN mkdir wolfssl && \
    pushd  wolfssl && \
    git init && \
    git remote add origin https://github.com/wolfSSL/wolfssl.git && \
    # e1829e6: Add wolfSSL_EVP_PKEY_paramgen to the compatibility layer. \
    git fetch --depth 1 origin e1829e614d35ad2dd1d6cc06d493158f16cca505 && \
    git checkout FETCH_HEAD && \
    # add new macro \
    sed -i "$(grep -nE '^#define ASN1_IA5STRING.*' wolfssl/openssl/ssl.h | awk -F':' '{ print $1 }' | head -n1) i #define ASN1_IA5STRING_new              wolfSSL_ASN1_STRING_new" wolfssl/openssl/ssl.h && \
    # fix naming collision \
    sed -i 's/internal_error/wolfssl_internal_error/g' src/internal.c src/ssl.c wolfssl/ssl.h && \
    # fix to allow our version of autoconf \
    sed -i 's/1\.14\.1/1.13.4/g' configure.ac && \
    ./autogen.sh && \
    ./configure CFLAGS="-O3 -fPIC" --prefix=/opt/wolfSSL  \
    --enable-aesgcm  --enable-aesctr  --enable-armasm \
    --enable-sp --enable-sp-asm --enable-sp-math-all \
    --enable-opensslextra --enable-opensslall --enable-asio \
    --enable-static  --enable-pwdbased --enable-sessioncerts && \
    make -j "$(nproc)" && \
    make install && \
    popd && \
    rm -rf /tmp/*

# install gradle
RUN curl -Ls https://services.gradle.org/distributions/gradle-7.5.1-bin.zip -o gradle.zip && \
    echo "f6b8596b10cce501591e92f229816aa4046424f3b24d771751b06779d58c8ec4  gradle.zip" > gradle-sha.txt && \
    sha256sum --quiet -c gradle-sha.txt && \
    unzip -qq gradle.zip && \
    mv gradle-7.5.1 /opt/gradle && \
    echo '[ -x /opt/gradle/bin/gradle ] && export PATH=/opt/gradle/bin/:$PATH' >> /etc/profile.d/gradle.sh && \
    rm -rf /tmp/*

# install maven
RUN curl -Ls https://archive.apache.org/dist/maven/maven-3/3.8.3/binaries/apache-maven-3.8.3-bin.zip -o maven.zip && \
    echo "f28cd38f620d76423c4543d5b443cdbdd5cfac2c511626cb92be3d5d273a6959  maven.zip" > maven-sha.txt && \
    sha256sum --quiet -c maven-sha.txt && \
    unzip -qq maven.zip && \
    mv apache-maven-3.8.3 /opt/maven && \
    echo '[ -x /opt/maven/bin/mvn ] && export PATH=/opt/maven/bin/:$PATH' >> /etc/profile.d/maven.sh && \
    rm -rf /tmp/*

# install rocksdb to /opt
RUN curl -Ls https://github.com/facebook/rocksdb/archive/refs/tags/v6.27.3.tar.gz -o rocksdb.tar.gz && \
    echo "ee29901749b9132692b26f0a6c1d693f47d1a9ed8e3771e60556afe80282bf58  rocksdb.tar.gz" > rocksdb-sha.txt && \
    sha256sum --quiet -c rocksdb-sha.txt && \
    tar --directory /opt -xf rocksdb.tar.gz && \
    rm -rf /tmp/*

# install Boost::context, Boost::filesystem & Boost::iostreams 1.78 to /opt
RUN curl -Ls https://boostorg.jfrog.io/artifactory/main/release/1.78.0/source/boost_1_78_0.tar.bz2 -o boost_1_78_0.tar.bz2 && \
    echo "8681f175d4bdb26c52222665793eef08490d7758529330f98d3b29dd0735bccc  boost_1_78_0.tar.bz2" > boost-sha.txt && \
    sha256sum --quiet -c boost-sha.txt && \
    mkdir -p /opt/boost_1_78_0 && \
    tar --strip-components 1 --no-same-owner --directory /opt/boost_1_78_0 -xjf boost_1_78_0.tar.bz2 && \
    cd /opt/boost_1_78_0 && \
    ./bootstrap.sh --with-libraries=context,filesystem,iostreams &&\
    ./b2 link=static cxxflags="-std=c++14 -fPIC" --prefix=/opt/boost_1_78_0 install &&\
    rm -rf /opt/boost_1_78_0/libs && \
    rm -rf /tmp/*

# Install Boost::context, Boost::filesystem & Boost::iostreams 1.78 to /opt, using clang to compile the library
# Boost::context depends on some C++11 features, e.g. std::call_once; however,
# gcc and clang are using different ABIs, thus a gcc-built Boost::context is
# not linkable to clang objects.
RUN curl -Ls https://boostorg.jfrog.io/artifactory/main/release/1.78.0/source/boost_1_78_0.tar.bz2 -o boost_1_78_0.tar.bz2 && \
    echo "8681f175d4bdb26c52222665793eef08490d7758529330f98d3b29dd0735bccc  boost_1_78_0.tar.bz2" > boost-sha.txt && \
    sha256sum --quiet -c boost-sha.txt && \
    mkdir -p /opt/boost_1_78_0_clang && \
    tar --strip-components 1 --no-same-owner --directory /opt/boost_1_78_0_clang -xjf boost_1_78_0.tar.bz2 && \
    cd /opt/boost_1_78_0_clang && \
    ./bootstrap.sh --with-toolset=clang --with-libraries=context,filesystem,iostreams && \
    ./b2 link=static cxxflags="-std=c++14 -stdlib=libc++ -nostdlib++ -fPIC" linkflags="-stdlib=libc++ -nostdlib++ -static-libgcc -lc++ -lc++abi" --prefix=/opt/boost_1_78_0_clang install && \
    rm -rf /opt/boost_1_78_0_clang/libs && \
    rm -rf /tmp/*

# build/install toml
RUN curl -Ls https://github.com/ToruNiina/toml11/archive/v3.4.0.tar.gz -o toml.tar.gz && \
    echo "bc6d733efd9216af8c119d8ac64a805578c79cc82b813e4d1d880ca128bd154d  toml.tar.gz" > toml-sha256.txt && \
    sha256sum --quiet -c toml-sha256.txt && \
    mkdir toml && \
    tar --strip-components 1 --no-same-owner --directory toml -xf toml.tar.gz && \
    cd toml && \
    mkdir build && \
    cd build && \
    CC=clang CXX=clang++ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -Dtoml11_BUILD_TEST=OFF ../ && \
    cmake --build . --target install && \
    cd .. && \
    rm -rf /tmp/*

RUN curl -Ls https://github.com/manticoresoftware/manticoresearch/raw/master/misc/junit/ctest2junit.xsl -o /opt/ctest2junit.xsl

# Download Rust binaries
ENV RUSTUP_VERSION=1.24.3 \
    RUST_VERSION=1.59.0
RUN if [ "$(uname -m)" == "aarch64" ]; then \
    RUST_ARCH="aarch64-unknown-linux-gnu"; \
    RUST_SHA256="32a1532f7cef072a667bac53f1a5542c99666c4071af0c9549795bbdb2069ec1"; \
    else \
    RUST_ARCH="x86_64-unknown-linux-gnu"; \
    RUST_SHA256="3dc5ef50861ee18657f9db2eeb7392f9c2a6c95c90ab41e45ab4ca71476b4338"; \
    fi && \
    curl -LsO "https://static.rust-lang.org/rustup/archive/${RUSTUP_VERSION}/${RUST_ARCH}/rustup-init" && \
    echo "${RUST_SHA256}  rustup-init" > rustup-sha.txt && \
    sha256sum --quiet -c rustup-sha.txt && \
    chmod +x rustup-init && \
    ./rustup-init -y --no-modify-path --profile minimal --default-toolchain ${RUST_VERSION} --default-host ${RUST_ARCH} && \
    rm -rf /tmp/*


# build/install Ruby
RUN git clone https://github.com/rbenv/rbenv.git ~/.rbenv && \
    echo 'eval "$(~/.rbenv/bin/rbenv init - bash)"' >> ~/.bashrc && \
    eval "$(~/.rbenv/bin/rbenv init - bash)" && \
    git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build && \
    rbenv install 2.7.6 && \
    rbenv global 2.7.6 && \
    rm -f ruby-build.20221102170639.81.log


FROM build as devel