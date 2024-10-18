#Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /source

# Argument to accept the project path (Looking for a way to be automated soon)
ARG PROJECT_NAME
ENV PROJECT_NAME=${PROJECT_NAME}

# Copy the files
COPY . .

# Restore dependencies
RUN dotnet restore

# Publish the project defined by PROJECT_PATH arg
RUN dotnet publish "/source/${PROJECT_NAME}" -c Release -o /app/publish

# Stage 2: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

# Copy the published files from the build stage
COPY --from=build /app/publish .

# Expose the default ports for .NET 8 (8080) the SSL will be managed by NGINX.
EXPOSE 8080

ENTRYPOINT ["sh", "-c", "dotnet /app/${PROJECT_NAME}.dll"]