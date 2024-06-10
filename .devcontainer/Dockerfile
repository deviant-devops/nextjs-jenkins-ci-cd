FROM debian:stable

RUN apt-get update \
    && apt-get install -y curl \
    && apt-get -y autoclean

ENV NVM_VERSION v0.39.7
ENV NODE_VERSION v20.14.0
ENV NVM_DIR /usr/local/nvm

RUN mkdir $NVM_DIR
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

ENV NODE_PATH $NVM_DIR/$NODE_VERSION/lib/node_modules
ENV PATH $NVM_DIR/versions/node/$NODE_VERSION/bin:$PATH

RUN echo "source $NVM_DIR/nvm.sh && \
    nvm install $NODE_VERSION && \
    nvm alias default $NODE_VERSION && \
    nvm use default" | bash

# confirm installation
RUN node -v
RUN npm -v