# Copyright (c) 2016-present Sonatype, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM alpine:3.8 as loader

ARG NEXUS_VERSION=3.13.0-01
ARG NEXUS_DOWNLOAD_URL=https://download.sonatype.com/nexus/3/nexus-${NEXUS_VERSI                                            ON}-unix.tar.gz
ARG NEXUS_DOWNLOAD_SHA256_HASH=5d1890f45e95e2ca74e62247be6b439482d2fe4562a7ec8ae                                            905c4bdba6954ce

RUN wget -q -O nexus.tgz $NEXUS_DOWNLOAD_URL
RUN tar -xzvf nexus.tgz
RUN mv nexus-$NEXUS_VERSION nexus

FROM dmaj/java:8

MAINTAINER Sonatype <cloud-ops@sonatype.com>

LABEL vendor=Sonatype \
      com.sonatype.license="Apache License, Version 2.0" \
      com.sonatype.name="Nexus Repository Manager base image"


# configure nexus runtime
ENV SONATYPE_DIR=/opt/sonatype
ENV NEXUS_HOME=${SONATYPE_DIR}/nexus \
    NEXUS_DATA=/nexus-data \
    NEXUS_CONTEXT='' \
    SONATYPE_WORK=${SONATYPE_DIR}/sonatype-work \
    DOCKER_TYPE='docker'


RUN mkdir -p ${SONATYPE_DIR} ${NEXUS_HOME} ${NEXUS_DATA} ${SONATYPE_WORK}

RUN addgroup -g 1000 nexus && \
    adduser nexus -S -h /home/x -u 1000 -G nexus


RUN printf "#!/bin/bash\n\
\n\
cd /opt/sonatype/nexus\n\
exec ./bin/nexus run\n"\
 > ${SONATYPE_DIR}/start-nexus-repository-manager.sh


COPY --from=loader /nexus ${NEXUS_HOME}
COPY --from=loader /sonatype-work ${SONATYPE_WORK}

RUN chown -R nexus:nexus ${SONATYPE_DIR} \
 && chown -R nexus:nexus ${NEXUS_DATA} \
 && chmod 766 ${SONATYPE_DIR}/*.sh

VOLUME ${NEXUS_DATA}

EXPOSE 8081
USER nexus

ENV INSTALL4J_ADD_VM_PARAMS="-Xms1000m -Xmx1000m -XX:MaxDirectMemorySize=800m -D                                            java.util.prefs.userRoot=${NEXUS_DATA}/javaprefs"

CMD ["sh", "-c", "${SONATYPE_DIR}/start-nexus-repository-manager.sh"]
