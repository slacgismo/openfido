# syntax=docker/dockerfile:1.0.0-experimental
FROM python:3.8-slim as base

# S6 for multiprocess management

ADD https://github.com/just-containers/s6-overlay/releases/download/v2.1.0.2/s6-overlay-amd64-installer /tmp/
RUN chmod u+x /tmp/s6-overlay-amd64-installer && \
  /tmp/s6-overlay-amd64-installer / && \
  rm /tmp/s6-overlay-amd64-installer

# All project dependencies

ADD https://download.docker.com/linux/debian/gpg /tmp/
ADD https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc /tmp/
ADD https://www.postgresql.org/media/keys/ACCC4CF8.asc /tmp/
RUN apt-get update && apt-get install --no-install-recommends -y \
  gnupg ssh git apt-transport-https ca-certificates curl && \
  (cat /tmp/gpg | apt-key add -) && \
  (cat /tmp/rabbitmq-release-signing-key.asc | apt-key add -) && \
  (cat /tmp/ACCC4CF8.asc | apt-key add -) && \
  rm /tmp/gpg && \
  rm /tmp/rabbitmq-release-signing-key.asc && \
  rm /tmp/ACCC4CF8.asc && \
  echo "deb https://dl.bintray.com/rabbitmq-erlang/debian buster erlang-22.x" > /etc/apt/sources.list.d/bintray.erlang.list && \
  echo "deb https://dl.bintray.com/rabbitmq/debian buster main" > /etc/apt/sources.list.d/bintray.rabbitmq.list && \
  echo "deb http://apt.postgresql.org/pub/repos/apt buster-pgdg main" > /etc/apt/sources.list.d/bintray.postgres.list && \
  echo "deb [arch=amd64] https://download.docker.com/linux/debian buster stable" > /etc/apt/sources.list.d/bintray.docker.list && \
  mkdir /docker-entrypoint-initdb.d && \
  apt-get clean && apt-get update && apt-get install --no-install-recommends -y \
  docker-ce \
  gosu tzdata postgresql-common postgresql-12 postgresql-contrib-12 \
  rabbitmq-server && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

# AUTH, WORKFLOW, APP DEPS

FROM base as pipenv-deps

ARG SSH_PRIVATE_KEY
RUN mkdir /root/.ssh/ && \
  echo "${SSH_PRIVATE_KEY}" > /root/.ssh/id_rsa && \
  chmod 600 /root/.ssh/id_rsa && \
  touch /root/.ssh/known_hosts && \
  ssh-keyscan github.com >> /root/.ssh/known_hosts

RUN mkdir /opt/openfido-auth-service
ADD openfido-auth-service/Pipfile /opt/openfido-auth-service
ADD openfido-auth-service/Pipfile.lock /opt/openfido-auth-service
RUN pip install pipenv
RUN --mount=type=ssh cd /opt/openfido-auth-service && PIPENV_VENV_IN_PROJECT="enabled" pipenv install --dev --deploy

## WORKFLOW DEPS

RUN mkdir /opt/openfido-workflow-service
ADD openfido-workflow-service/Pipfile /opt/openfido-workflow-service
ADD openfido-workflow-service/Pipfile.lock /opt/openfido-workflow-service
RUN pip install pipenv
RUN --mount=type=ssh cd /opt/openfido-workflow-service && PIPENV_VENV_IN_PROJECT="enabled" pipenv install --dev --deploy

## APP SERVER

RUN mkdir /opt/openfido-app-service
ADD openfido-app-service/Pipfile /opt/openfido-app-service
ADD openfido-app-service/Pipfile.lock /opt/openfido-app-service
RUN pip install pipenv
RUN --mount=type=ssh cd /opt/openfido-app-service && PIPENV_VENV_IN_PROJECT="enabled" pipenv install --dev --deploy

# All services

FROM base as services

RUN mkdir /etc/services.d/0-rabbitmq \
  /etc/services.d/1-minio \
  /etc/services.d/2-postgres \
  /etc/services.d/3-openfido-auth-service \
  /etc/services.d/4-openfido-workflow-service \
  /etc/services.d/5-openfido-app-service \
  /etc/services.d/6-openfido-workflow-worker \
  /etc/services.d/7-openfido-client \
  /opt/app-keys

## AUTH

COPY openfido-auth-service /opt/openfido-auth-service
COPY --from=pipenv-deps /opt/openfido-auth-service/.venv /opt/openfido-auth-service/.venv
ADD files/auth-env /opt/openfido-auth-service/env
ADD files/run-auth-service.sh /etc/services.d/3-openfido-auth-service/run

EXPOSE 5001/tcp

## WORKFLOW

COPY openfido-workflow-service /opt/openfido-workflow-service
COPY --from=pipenv-deps /opt/openfido-workflow-service/.venv /opt/openfido-workflow-service/.venv
ADD files/workflow-env /opt/openfido-workflow-service/env
ADD files/run-workflow-service.sh /etc/services.d/4-openfido-workflow-service/run

EXPOSE 5002/tcp

## APP

COPY openfido-app-service /opt/openfido-app-service
COPY --from=pipenv-deps /opt/openfido-app-service/.venv /opt/openfido-app-service/.venv
ADD files/app-env /opt/openfido-app-service/env
ADD files/run-app-service.sh /etc/services.d/5-openfido-app-service/run

EXPOSE 5003/tcp

## WORKER

ADD files/worker-env /opt/openfido-workflow-service/worker-env
ADD files/run-workflow-worker.sh /etc/services.d/6-openfido-workflow-worker/run

## REACT CLIENT

COPY openfido-client /opt/openfido-client
ENV NVM_DIR /opt/openfido-client/.nvm
ADD https://raw.githubusercontent.com/nvm-sh/nvm/v0.37.2/install.sh /tmp/
RUN (mkdir $NVM_DIR && cat /tmp/install.sh | bash) && \
  rm /tmp/install.sh && \
  . $NVM_DIR/nvm.sh && cd /opt/openfido-client && \
  nvm install && \
  npm install && \
  npm install -g serve
ADD files/run-client.sh /etc/services.d/7-openfido-client/run
ADD files/client-config.js /opt/openfido-client/src/config/index.js

EXPOSE 3000/tcp

# MINIO

ADD https://dl.min.io/server/minio/release/linux-ppc64le/minio /tmp/
RUN chmod +x /tmp/minio && \
  mkdir /opt/minio && \
  mv /tmp/minio /opt/minio && \
  mkdir /var/opt/minio /var/opt/minio/app-data /var/opt/minio/auth-data /var/opt/minio/workflow-data
ADD files/run-minio /etc/services.d/1-minio/run
ADD files/minio-env /opt/minio/env

VOLUME /var/opt/minio
EXPOSE 9000/tcp

# RABBITMQ

ADD files/run-rabbitmq /etc/services.d/0-rabbitmq/run

# POSTGRES

RUN mkdir /opt/postgres && \
  localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ADD files/run-postgres /etc/services.d/2-postgres/run
ADD files/postgres-env /opt/postgres/env

VOLUME /var/lib/postgresql/data
EXPOSE 5432/tcp

# SEED SCRIPT

ADD files/seed.sh /etc/cont-init.d/1-seed.sh

ENTRYPOINT ["/init"]
