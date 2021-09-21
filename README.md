
Setup `docker buildx`: https://docs.docker.com/buildx/working-with-buildx/#install

Enable emulation:

    docker run --privileged --rm tonistiigi/binfmt --install all

Run the build:

    docker buildx build . -f container/kura_debian/Dockerfile --platform linux/amd64,linux/arm64,linux/arm/v7

You can then push it:

    docker buildx build . -f container/kura_debian/Dockerfile --platform linux/amd64,linux/arm64,linux/arm/v7 --tag my-repo/my-org/mytag:latest --push
