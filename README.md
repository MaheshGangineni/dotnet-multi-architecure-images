# dotnet-docker-multi-arch-images
Build a container image for a specific architecture (different than your build host).

You might have observed the below warnings(s) while running a docker image build on your local machine/ CICD on any servers/colud environment.

```console
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64) and no specific platform was requested
```

```console
WARNING: The requested image's platform (linux/arm64) does not match the detected host platform (linux/amd64) and no specific platform was requested
```

These warning are due to the difference in platofrms of image and host (on which is container is running). In some scenarios the container might get exited automatically.

Running an amd64 (x86_64) image on an arm64 (ARM) host is not directly possible because these architectures are not compatible. Docker images are platform-specific, and they are built to run on a specific architecture.

However, there are some solutions to consider:

- **QEMU Emulation:** You can use QEMU to emulate a different architecture. This can be achieved by enabling QEMU in the Docker image for arm64 and then running an amd64 image inside it.
*Keep in mind that emulation may have some performance overhead.*

- **Multi-Architecture Images:** 
  - Build a container image for a specific architecture (different than your machine).
  - Build multiple container images at once, for multiple architectures.

We are focusing on the first way mentioned in multi-architecture images.

# Mutli-platform build

I am using a windows machine on which docker desktop is installed and Os/Arch are as follows:
```console
   Docker client arch : windows/amd64
   Docker engine arch : linux/amd64
```

You can verify yours using below cmd.

```console 
    docker version
```
Docker has a ```--platform``` switch that you can use to control the output of your images.

```console 
    docker build --platform platfro-arch -t your-image-name:tag .
```
where ```platfro-arch``` can be ```linux/amd64```, ```linux/arm4```, ```darwin/amd64``` etc.

But when you try to build the .NET images using above command and provide the platform as different from your build host you might face issues in building image.

**Ex:**

```console 
    docker buildx build --platform linux/arm64 -t docker-arch-v8:alpine -f docker-build-arch/Dockerfile .
```
I ran the above cmd to build an arm64 based image on my machine, build failed with below error.

```
 => ERROR [build 4/7] RUN dotnet restore "docker-build-arch/docker-build-arch.csproj" -a arm64  22.9s
------
 > [build 4/7] RUN dotnet restore "docker-build-arch/docker-build-arch.csproj" -a arm64:
#15 22.21 /usr/share/dotnet/sdk/8.0.201/Sdks/Microsoft.NET.Sdk.Web/Sdk/Sdk.props(18,50): error MSB4184: The expression "[MSBuild]::GetTargetPlatformIdentifier('')" cannot be evaluated. Exception has been thrown by the target of an invocation. [/src/docker-build-arch/docker-build-arch.csproj]
------
error: failed to solve: executor failed running [/bin/sh -c dotnet restore "docker-build-arch/docker-build-arch.csproj" -a $TARGETARCH]: exit code: 1
```
Issue is due to the mismatch of architecures in build stage.Since the platform is provided as linux/arm64, by default all the dotnet base images used in Dockerfile will pull the image with respect to the plaform specified. In our case it pulled arm64 where as our build host is amd64.

```
FROM  mcr.microsoft.com/dotnet/sdk:8.0-alphine AS build
```

So we need to explictly get the sdk which matches with the host (on which image is being built). In our case it is ```amd64```.

Here hardcoding the platform for given stage is a bad idea for so many reasons like all devs may not work machines which has same architecture or CICD runner architecures may change.

Docker out of box provides build arugmnets like ```BUILDPLATFORM``` ,```
TARGETPLATFORM```, ```TARGETARCH``` etc.

we use these build arugmnets to get the native sdk and build binaries based on target architecure.
After modification of dockerfile build stage looks as below-

```
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
ARG TARGETARCH
WORKDIR /src
COPY ["docker-build-arch/docker-build-arch.csproj", "docker-build-arch/"]
RUN dotnet restore "docker-build-arch/docker-build-arch.csproj" -a $TARGETARCH
COPY . .
WORKDIR "/src/docker-build-arch"
RUN dotnet build "docker-build-arch.csproj" -c Release -a $TARGETARCH  -o /app/build

FROM build AS publish
RUN dotnet publish "docker-build-arch.csproj" -c Release -a $TARGETARCH --no-restore -o /app/publish
```

**Note:**  dotnet sdk(s) prior to 8 may not support  architecure flag 
```-a``` for  ```restore```,```buold``` and ```publish``` cmds. In those scenarios you can use [rids](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog)

On doing the above mentioned changes you can successfully build images targeted to different architecures.

you can verify the architecure in container as below

```bash
/app $ dotnet --info
Host:
  Version:      8.0.2
  Architecture: arm64
  Commit:       1381d5ebd2
  RID:          linux-musl-arm64
```
```bash
/app $ arch
aarch64
```

Architecture can be infered from image manifest using ```docker inspect <image-id>```. But it may not be accurate due some to the mistakes we do in writing dockerfile, but may not affect the build. check more on this [improving-multiplatform-container-support](https://devblogs.microsoft.com/dotnet/improving-multiplatform-container-support/)

#Related & useful links
- [docker-multi-platform-builds](https://docs.docker.com/build/building/multi-platform/)
- [improving-multiplatform-container-support](https://devblogs.microsoft.com/dotnet/improving-multiplatform-container-support/)
- [securing-containers-with-rootless](https://devblogs.microsoft.com/dotnet/securing-containers-with-rootless/)
- [optimise-dotnet-docker-images](https://www.hanselman.com/blog/optimizing-aspnet-core-docker-image-sizes)
- [dotnet-rids](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog)