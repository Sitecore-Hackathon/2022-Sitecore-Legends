﻿# escape=`

# This Dockerfile will build the Sitecore solution and save the build artifacts for use in
# other images, such as 'cm' and 'rendering'. It does not produce a runnable image itself.
ARG BASE_IMAGE
ARG BUILD_IMAGE

# In a separate image (as to not affect layer cache), gather all NuGet-related solution assets, so that
# we have what we need to run a cached NuGet restore in the next layer:
# https://stackoverflow.com/questions/51372791/is-there-a-more-elegant-way-to-copy-specific-files-using-docker-copy-to-the-work/61332002#61332002
# This technique is described here:
# https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/building-net-docker-images?view=aspnetcore-3.1#the-dockerfile-1
FROM ${BUILD_IMAGE} AS nuget-prep
COPY *.sln nuget.config Directory.Build.targets Packages.props /nuget/
COPY src/ /temp/
RUN Invoke-Expression 'robocopy C:/temp C:/nuget/src /s /ndl /njh /njs *.csproj *.scproj packages.config'

FROM ${BUILD_IMAGE} AS builder
ARG BUILD_CONFIGURATION

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
WORKDIR /build

# Copy prepped NuGet artifacts, and restore as distinct layer to take advantage of caching.
COPY --from=nuget-prep ./nuget ./

# Restore NuGet packages
RUN nuget restore -Verbosity quiet

# Copy remaining source code
COPY src/ ./src/

# Ensure deploy folders exist
RUN mkdir ./docker/deploy/platform; mkdir ./docker/deploy/rendering; mkdir ./docker/deploy/identity

# Build the solution to generate build artifacts
## assumes that the msbuild property <SitecoreRoleType>platform|rendering</SitecoreRoleType> is used to target different deploy folders,
RUN Get-ChildItem *.sln | %{  msbuild $_.FullName /p:Configuration=$env:BUILD_CONFIGURATION /m /p:DeployOnBuild=true /p:IsLocalDockerDeploy=true }

# Final solution assets image
FROM ${BASE_IMAGE}
WORKDIR /artifacts

# Copy in all artifacts from build
COPY --from=builder /build/docker/deploy/ ./