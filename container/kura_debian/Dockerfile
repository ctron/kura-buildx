FROM --platform=$BUILDPLATFORM docker.io/library/debian:buster AS builder

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
  LAUNCHER_VERSION="1.5.800.v20200727-1323" \
  DEBIAN_FRONTEND=noninteractive

RUN echo "$GIT_REPO / $GIT_BRANCH / $KURA_COMMIT"

RUN chmod a+x -R /usr/local/bin

RUN apt-get -y update
RUN apt-get -y install openjdk-11-jdk git maven
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
RUN mkdir -p /installers/linux/arm/v7

RUN cp /kura/kura/distrib/target/kura_*_intel-up2-centos-7-nn_installer.sh /installers/linux/amd64/installer.sh
RUN cp /kura/kura/distrib/target/kura_*_raspberry-pi-2-nn_installer.sh /installers/linux/arm64/installer.sh
RUN cp /kura/kura/distrib/target/kura_*_raspberry-pi-2-nn_installer.sh /installers/linux/arm/v7/installer.sh

RUN find /installers

FROM docker.io/library/debian:buster

LABEL org.opencontainers.image.source="https://github.com/ctron/kura-buildx"

ARG TARGETPLATFORM

COPY --from=builder /installers/${TARGETPLATFORM}/installer.sh /installer.sh

ARG PACKED=false

ENV \
  KURA_DIR=/opt/eclipse/kura \
  SKIP_JAVA_VERSION_CHECK=true \
  LAUNCHER_VERSION="1.5.800.v20200727-1323" \
  DEBIAN_FRONTEND=noninteractive

ENV \
  JAVA_HOME=/opt/java/openjdk \
  PATH=${PATH}:/opt/java/openjdk/bin

RUN mkdir -p /usr/local/bin

RUN apt-get -y update
RUN apt-get -y install openjdk-11-jdk zip unzip curl gzip tar socat dos2unix grep procps psmisc libglib2.0-0 openssl net-tools
#RUN apk --no-cache add eudev glib bash zip curl procps zip unzip gzip tar psmisc socat dos2unix openssl net-tools which grep busybox-extras

RUN /bin/sh -c \
    set -eux; \
    case "${TARGETPLATFORM}" in \
       "linux/arm64") \
         BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.12%2B7/OpenJDK11U-jdk_aarch64_linux_hotspot_11.0.12_7.tar.gz'; \
         ;; \
       "linux/arm/v7") \
         BINARY_URL='https://github.com/AdoptOpenJDK/openjdk11-binaries/releases/download/jdk-11.0.11%2B9/OpenJDK11U-jdk_arm_linux_hotspot_11.0.11_9.tar.gz'; \
         ;; \
       "linux/amd64") \
         BINARY_URL='https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.12%2B7/OpenJDK11U-jdk_x64_linux_hotspot_11.0.12_7.tar.gz'; \
         ;; \
       *) \
         echo "Unsupported arch: ${ARCH}"; \
         exit 1; \
         ;; \
    esac; \
    curl -LfsSo /tmp/openjdk.tar.gz ${BINARY_URL}; \
    mkdir -p /opt/java/openjdk;  \
    cd /opt/java/openjdk;  \
    tar -xf /tmp/openjdk.tar.gz --strip-components=1;  \
    rm -rf /tmp/openjdk.tar.gz

COPY --from=builder /context/bin/* /usr/local/bin

RUN true && \
    if ! /installer.sh; then \
      cat /tmp/kura_install_*.log ; false ; \
    fi && \
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
