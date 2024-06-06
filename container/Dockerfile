FROM emscripten/emsdk:3.1.59
LABEL maintainer "tom wilson <tom@hyper.io>"

# The working directory used by the base image is /src, so we can mount volumes to there
# to expose files on the host to the ao container
# 
# https://github.com/emscripten-core/emsdk/blob/9b0db91883452051aca8deddc932363aab29060b/docker/Dockerfile#L120

RUN apt-get update -qq -y
RUN apt-get install -y curl vim make gcc libreadline6-dev libssl-dev zlib1g-dev zip unzip

ENV LUA_VERSION 5.3.4
ENV LUAROCKS_VERSION 2.4.4
ENV PYTHON_VERSION 3.6.6

# install deno
# RUN curl -fsSL https://deno.land/x/install/install.sh | sh
# RUN echo 'export DENO_INSTALL="/root/.deno"' >> $HOME/.bashrc
# RUN echo 'export PATH="$DENO_INSTALL/bin:$PATH"' >> $HOME/.bashrc

# Intall yaml
RUN pip3 install pyyaml

# Install lua runtime
RUN cd / && \
  curl -L http://www.lua.org/ftp/lua-${LUA_VERSION}.tar.gz | tar xzf - && \
  cd /lua-${LUA_VERSION} && \
  make linux test && \
  make install

# Install luarocks
RUN cd / && \
  curl -L https://luarocks.org/releases/luarocks-${LUAROCKS_VERSION}.tar.gz | tar xzf - && \
  cd /luarocks-${LUAROCKS_VERSION} && \
  ./configure && \
  make build && \
  make install

# Install NodeJS
ENV NODE_VERSION=18.17.0
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
ENV NVM_DIR=/root/.nvm
RUN . "$NVM_DIR/nvm.sh" && nvm install ${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm use v${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm alias default v${NODE_VERSION}
ENV PATH="/root/.nvm/versions/node/v${NODE_VERSION}/bin/:${PATH}"
RUN node --version
RUN npm --version

# And, re-compile lua with "generic WASM"
RUN cd /lua-${LUA_VERSION} && \
  make clean && \
  make generic CC='emcc -s WASM=1 -s MEMORY64=1 -s SUPPORT_LONGJMP=1' 

#############################
##### Install Commands ######
#############################

# NOTE: Some commands do not require additional work
# when building the image, but are still commented
# here for posterity

###############
#### Init #####
###############
# The init command currently does not require running
# within the ao container, and so requires no additional setup

###############
#### Repl #####
###############
# The repl command simply fires up lua repl, which is installed
# as part of lua runtime above, and so requires no additional setup

###############
#### Build ####
###############
COPY ./src/emcc-lua /usr/local/bin/emcc-lua
COPY ./src/emcc_lua_lib /usr/local/emcc-lua/emcc_lua_lib

COPY ./src/pre.js /opt/pre.js
COPY ./src/definition.yml /opt/definition.yml
COPY ./src/loader.lua /opt/loader.lua

RUN mkdir -p /opt/src
COPY ./src/json.lua /opt/src/json.lua
COPY ./src/ao.lua /opt/src/ao.lua
# COPY ./src/pack.lua /opt/pack.lua
COPY ./src/main.c /opt/main.c
# COPY ./src/linit.c /opt/linit.c
COPY ./src/lsqlite3.c /opt/lsqlite3.c
COPY ./src/lsqlite3.h /opt/lsqlite3.h
COPY ./src/sqlite3.c /opt/sqlite3.c
COPY ./src/sqlite3.h /opt/sqlite3.h
COPY ./src/main.lua /opt/main.lua

###################################
# BUILD WeaveDrive Extension Helper
###################################
COPY ./src/aolibc /opt/aolibc
RUN cd /opt/aolibc && make CC="emcc -s WASM=1 -s MEMORY64=1 -s SUPPORT_LONGJMP=1"


RUN chmod +x /usr/local/bin/emcc-lua

ENV CC 'emcc -s WASM=1'
ENV NM 'llvm-nm'

COPY ./src/ort.c /opt/ort.c
COPY ./src/Makefile /opt/Makefile
COPY ./src/model.onnx /opt/model.onnx

########################
# BUILD ONNXRUNTIME WASM
########################
# https://onnxruntime.ai/docs/build/web.html#build-onnx-runtime-webassembly-artifacts
# Download and install CMake 3.26
RUN wget https://github.com/Kitware/CMake/releases/download/v3.26.4/cmake-3.26.4-linux-x86_64.sh && \
    mkdir /opt/cmake && \
    sh cmake-3.26.4-linux-x86_64.sh --prefix=/opt/cmake --skip-license && \
    ln -s /opt/cmake/bin/cmake /usr/local/bin/cmake && \
    rm cmake-3.26.4-linux-x86_64.sh

# Clone the ONNX Runtime repository
RUN git clone --recursive https://github.com/microsoft/onnxruntime /onnxruntime
# Set the working directory
WORKDIR /onnxruntime

# Install Python dependencies
RUN pip3 install -r requirements.txt.in

# Build ONNX Runtime WebAssembly artifacts
RUN /bin/bash -c "source /emsdk/emsdk_env.sh && \
    ./build.sh --build_dir /onnxruntime/build --config Release --build_wasm --skip_tests --disable_wasm_exception_catching --disable_rtti --allow_running_as_root --parallel"

# Copy the build artifacts to the desired location
RUN mkdir -p /onnxruntime/wasm_artifacts && \
    cp -r /onnxruntime/build/Release/* /onnxruntime/wasm_artifacts/

# Set the working directory to the artifacts folder
WORKDIR /onnxruntime/wasm_artifacts

## Build lort
RUN cd /opt && \
  make emscripten

###########################################
### Publish, Bundler, Process, Contract ###
###########################################
# We first create a directory for the node impls to be placed
# and dependencies installed
# 
# By running npm link, we allow any commands exposed by
# the node module to be ran globally within the container
RUN mkdir -p /opt/node
COPY ./src/node /opt/node
RUN cd /opt/node && \
  npm install --omit="dev" && \
  npm link