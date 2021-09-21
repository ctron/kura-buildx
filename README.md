
Setup docker buildx.

Enable emulation:

    docker run --privileged --rm tonistiigi/binfmt --install all

Run the build:

    docker buildx build . -f container/kura_debian/Dockerfile --platform linux/amd64,linux/arm64,linux/arm/v7
