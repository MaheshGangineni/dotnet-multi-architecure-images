FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETARCH
ARG BUILDARCH
ARG BUILDOS
ARG TARGETOS
ARG BUILDVARIANT
RUN echo "Building on $BUILDPLATFORM, targeting $TARGETPLATFORM"
RUN echo "Building on ${BUILDOS} and ${BUILDARCH}"
RUN echo "Targeting ${TARGETOS} and ${TARGETARCH}"
WORKDIR /app
EXPOSE 8080

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

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
USER $APP_UID
ENTRYPOINT ["dotnet", "docker-build-arch.dll"]