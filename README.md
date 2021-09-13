
Setup docker buildx.

Enable emulation:

    docker run --privileged --rm tonistiigi/binfmt --install all

Run the build:

    docker buildx build . -f container/kura_alpine/Dockerfile --platform linux/amd64,linux/arm64,linux/arm/v7 --build-arg MAVEN_PROPS="-DskipTests -Dcheckstyle.skip -Dgwt.extraJvmArgs=-Xmx1536m"

