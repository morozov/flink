###############################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
###############################################################################

# This Dockerfile is created based on https://github.com/apache/flink-docker/blob/187238c6b444457d9b57279715fa16b37fe59e45/1.13/scala_2.12-java11-debian/Dockerfile
# Updated the base image from openjdk:11-jre to openjdk:16-jdk-buster
FROM openjdk:16-jdk-buster

# Install dependencies
RUN set -ex; \
  apt-get update; \
  apt-get -y install wget gpg libsnappy1v5 gettext-base libjemalloc-dev zip python3-pip; \
  rm -rf /var/lib/apt/lists/*

# Setup AWS CLI (for heapdump uploader)
RUN pip3 install awscli --upgrade
ENV PATH=$PATH:/usr/local/bin/

# Grab gosu for easy step-down from root
ENV GOSU_VERSION 1.11
RUN set -ex; \
  wget -nv -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)"; \
  wget -nv -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc"; \
  export GNUPGHOME="$(mktemp -d)"; \
  for server in ha.pool.sks-keyservers.net $(shuf -e \
                          hkp://p80.pool.sks-keyservers.net:80 \
                          keyserver.ubuntu.com \
                          hkp://keyserver.ubuntu.com:80 \
                          pgp.mit.edu) ; do \
      gpg --batch --keyserver "$server" --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && break || : ; \
  done && \
  gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
  gpgconf --kill all; \
  rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
  chmod +x /usr/local/bin/gosu; \
  gosu nobody true

# Prepare environment
ENV FLINK_HOME=/opt/flink
ENV PATH=$FLINK_HOME/bin:$PATH
RUN groupadd --system --gid=9999 flink && \
    useradd --system --home-dir $FLINK_HOME --uid=9999 --gid=flink flink
WORKDIR $FLINK_HOME

# Install Flink
ARG FLINK_DIST_DIR
COPY --chown=flink:flink $FLINK_DIST_DIR $FLINK_HOME/

# remove web frontend files from lib flink dist
#RUN zip -d $FLINK_HOME/lib/flink-dist*.jar org/apache/flink/runtime/webmonitor/\*

# remove unused plugins
RUN mv $FLINK_HOME/plugins $FLINK_HOME/vendor-plugins ; mkdir $FLINK_HOME/plugins
RUN mv $FLINK_HOME/vendor-plugins/metrics-statsd $FLINK_HOME/plugins/
RUN mv $FLINK_HOME/vendor-plugins/metrics-influx $FLINK_HOME/plugins/
RUN mv $FLINK_HOME/vendor-plugins/metrics-prometheus $FLINK_HOME/plugins/
RUN rm -r $FLINK_HOME/vendor-plugins


# Decodable uses logback so removing default log4j libs.
# See https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/deployment/advanced/logging/#configuring-logback
RUN rm $FLINK_HOME/lib/log4j-*

# needed to avoid java.lang.ClassNotFoundException: org.apache.flink.table.planner.codegen.CodeGenException
RUN rm $FLINK_HOME/lib/flink-table-planner-loader*
RUN mv $FLINK_HOME/opt/flink-table-planner* $FLINK_HOME/lib/

# Copy Flink plugin jars (within the docker image, not from host)
# See: https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/deployment/filesystems/plugins/
RUN mkdir \
  ./plugins/s3-fs-hadoop
RUN cp \
  ./opt/flink-s3-fs-hadoop-*.jar \
  ./plugins/s3-fs-hadoop/
