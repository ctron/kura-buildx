FROM --platform=$BUILDPLATFORM docker.io/library/alpine:3.11 AS builder

MAINTAINER Eclipse Kura Developers <kura-dev@eclipse.org>
LABEL maintainer="Eclipse Kura Developers <kura-dev@eclipse.org>" \
      io.k8s.description="Containerized version of the Eclipse Kura™ IoT gateway"

ARG BUILDPLATFORM

ARG GIT_REPO=""
ARG GIT_BRANCH=""
ARG KURA_COMMIT=""
ARG PROFILE=intel-up2-centos-7-nn,raspberry-pi-2-3-nn

ENV \
  GIT_REPO=${GIT_REPO:-https://github.com/eclipse/kura.git} \
  GIT_BRANCH=${GIT_BRANCH:-develop} \
  JAVA_HOME=/usr/lib/jvm/default-jvm \
  KURA_DIR=/opt/eclipse/kura \
  LAUNCHER_VERSION="1.5.800.v20200727-1323"

# required for JDK tools like "jar"
ENV \
  PATH=${PATH}:/usr/lib/jvm/default-jvm/bin

RUN echo "$GIT_REPO / $GIT_BRANCH / $KURA_COMMIT"

RUN chmod a+x -R /usr/local/bin

RUN apk update
RUN apk --no-cache add git openjdk8 maven bash zip curl
RUN which jar

RUN true && \
    if [ -d /context/.git ]; then \
      mv /context /kura; \
      mv /kura/kura/container/kura_alpine /context; \
    else \
      git clone "$GIT_REPO" -b "$GIT_BRANCH"; \
      cp -a kura/kura/container/kura_alpine /context; \
    fi

RUN if [ -n "$KURA_COMMIT" ]; then git checkout "$KURA_COMMIT"; fi

WORKDIR /kura

RUN git log -1

# Replace broken 'nn' script
RUN cp kura/distrib/src/main/sh/extract.sh kura/distrib/src/main/sh/extract_nn.sh

ARG MAVEN_PROPS="-DskipTests -Dcheckstyle.skip"
ARG MAVEN_OPTS=""

RUN mvn -B -f target-platform/pom.xml clean install -Pno-mirror $MAVEN_PROPS
RUN mvn -B -f kura/pom.xml clean install $MAVEN_PROPS -Pspeedup
RUN mvn -B -f kura/distrib/pom.xml clean install $MAVEN_PROPS -P${PROFILE} -nsu

RUN ls -la /kura/kura/distrib/target

RUN mkdir -p /installers/linux/amd64
RUN mkdir -p /installers/linux/arm64
RUN mkdir -p /installers/linux/arm/7

RUN cp /kura/kura/distrib/target/kura_*_intel-up2-centos-7-nn_installer.sh /installers/linux/amd64/installer.sh
RUN cp /kura/kura/distrib/target/kura_*_raspberry-pi-2-nn_installer.sh /installers/linux/arm64/installer.sh
RUN cp /kura/kura/distrib/target/kura_*_raspberry-pi-2-nn_installer.sh /installers/linux/arm/v7/installer.sh

RUN find /installers

FROM docker.io/library/alpine:3.11

ARG TARGETPLATFORM

COPY --from=builder /installers/${TARGETPLATFORM}/installer.sh /installer.sh

ARG PACKED=false

ENV \
  KURA_DIR=/opt/eclipse/kura \
  JAVA_HOME=/usr/lib/jvm/jre-1.8.0 \
  SKIP_JAVA_VERSION_CHECK=true \
  LAUNCHER_VERSION="1.5.800.v20200727-1323"

ENV \
  PATH=$PATH:/usr/lib/jvm/java-1.8-openjdk/bin

RUN mkdir -p /usr/local/bin
RUN apk update
RUN apk --no-cache add openjdk8 eudev glib bash zip curl procps zip unzip gzip tar psmisc socat dos2unix openssl net-tools which busybox-extras
RUN ln -s /bin/bash /usr/bin/bash

COPY --from=builder /context/bin/* /usr/local/bin

RUN true && \
    /installer.sh && \
    echo "Kura installer completed" && \
    rm installer.sh && \
    chmod a+rw -R /opt/eclipse && \
    find /opt/eclipse -type d | xargs chmod a+x && \
    chmod a+rwx /var/log && \
    `# Test for the existence of the entry point` \
    test -x "${KURA_DIR}/bin/start_kura.sh" && \
    chmod a+x /usr/local/bin/* && \
    install -m 0777 -d "${KURA_DIR}/data" && \
    sed -i 's/kura.primary.network.interface/#kura.primary.network.interface/g' ${KURA_DIR}/framework/kura.properties && \
    mkdir -p ${KURA_DIR}/packages && \
    if [ "$PACKED" == "true" ]; then touch /kura.packed && pack-kura; fi && \
    \
    unpack-kura && \
    sed -i "s/\-printf \'\%P.n\'//g" /usr/local/bin/dp-install && \
    dp-install "https://repo1.maven.org/maven2/de/dentrassi/kura/addons/de.dentrassi.kura.addons.utils.fileinstall/0.6.0/de.dentrassi.kura.addons.utils.fileinstall-0.6.0.dp" && \
    add-config-ini "felix.fileinstall.disableNio2=true" && \
    add-config-ini "felix.fileinstall.dir=/load" && \
    pack-kura

EXPOSE 443

VOLUME ["/load"]

ENTRYPOINT ["/usr/local/bin/kura-entry-point"]
