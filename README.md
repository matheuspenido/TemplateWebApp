Backend with NGINX and Docker:

mkdir MyApp

Create the backend app:
docker run --rm -it -v $(pwd)/MyApp:/app mcr.microsoft.com/dotnet/sdk:8.0 bash

inside the container, do:

cd app
dotnet new sln -n MyApp
dotnet new webapi -n MyApp.API
dotnet sln MyApp.sln add MyApp.API/MyApp.API.csproj
dotnet new gitignore
exit

exit command will return to host.
change the files permission, to do that, go to the root folder and do:
sudo chown -R Matheus:Matheus ./MyApp

Verify if the aspnet runtime is installed
dotnet watch run --project ./MyApp/MyApp.API/

---------------------------------------
Nginx conf:

mkdir nginx-server

create inside nginx-conf the files the default.conf file with the content:

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri; #To redirect to https.
}

# Handle HTTPS for the backend
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/localhost.crt;
    ssl_certificate_key /etc/nginx/ssl/localhost.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Create the Dockerfile inside nginx-server folder:

#Official image for NGINX
FROM nginx:alpine

#Expose ports
EXPOSE 80
EXPOSE 443

CMD ["/bin/sh", "-c", "nginx -g 'daemon off;'"]

Create the certificate for the nginx-server on localhost:

cd nginx-server
mkdir certs
cd certs

openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -subj "/CN=localhost"

openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt -passout pass:mypapp

Now we can load the cert, inside the nginx server using docker.
Inside nginx-server folder:
docker build -t nginx-server .
docker run -v $(pwd)/certs/localhost.crt:/etc/nginx/ssl/localhost.crt -v $(pwd)/cert/localhost.key:/etc/nginx/ssl/localhost.key -p 443:443 -p 80:80 nginx-server

stop the container.

Create the Dockerfile file inside MyApp folder:

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

----------------------------------------------

Test configuration:
Back to the root folder, where we have the MyApp and nginx-server folders and execute:

Create a network to allow the communication between the nginx server (as a Reverse Proxy) and MyApp:

docker network create test-network

Create the backend api container:
docker build -f ./MyApp/Dockerfile --build-arg PROJECT_NAME=MyApp.API -t webapp ./MyApp
docker run -p 8080:8080 --name webapp-api --network test-network  -e PROJECT_NAME=MyApp.API -e ASPNETCORE_ENVIRONMENT=Development webapp

Create the nginx-server container:
docker build -f ./nginx-server/Dockerfile -t nginx-server ./nginx-server
docker run --network test-network -v $(pwd)/nginx-server/certs/localhost.crt:/etc/nginx/ssl/localhost.crt -v $(pwd)/nginx-server/certs/localhost.key:/etc/nginx/ssl/localhost.key -v $(pwd)/nginx-server/default.conf.template:/etc/nginx/conf.d/default.conf.template -p 443:443 -p 80:80 nginx-server

try to navigate http://localhost and notice that you should be redirect to https://localhost and a warning message should appear saying that there went something wrong with the certificate. That is happening because we are using a self signed certificate... Accept the risk, dont worry, this is our app.
Try to navigate now to https://localhost/swagger and you'll be able to see the swagger page running properly.

--------------------------------------------

Creating an configurable environment:

If you look at default.conf for nginx, you will see many hardcoded configuration:

server_name localhost; #hardcoded to localhost

proxy_pass http://webapp-api:8080; #hardcoded to webapp-api:8080

It is important to be set up because each environment must have their own specifications.
For purpose of test, this config is fine, but for prod, the servername should reflect the prod env and the proxy_pass will need to address to the real backend api ip.

first, lets pass the variables using the docker cli, but lets be prepared to change to docker-compose approach, to avoid overhead on cli.

first, replace the hardcoded default.conf by:
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name $SERVER_NAME;
    return 301 https://$host$request_uri; #To redirect to https.
}

# Handle HTTPS for the backend
server {
    listen 443 ssl;
    server_name $SERVER_NAME;

    ssl_certificate /etc/nginx/ssl/$CERT_NAME;
    ssl_certificate_key /etc/nginx/ssl/$CERT_KEY;

    location / {
        proxy_pass http://$API_ADDRESS:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Rename the file from default.conf to default.conf.template

Added command envsubs to replace the env variable that we have defined on default.conf.template throught tha Dockerfile. Is important to replace only the values that we want, because if they are not defined on envsubs, all env variables placeholders will be replaced, including the nginx ones, like $host for example. Cause that, envsubs has been receiving the variables that should be updated.

#Official image for NGINX
FROM nginx:alpine

#Expose ports
EXPOSE 80
EXPOSE 443

CMD ["/bin/sh", "-c", "\
    envsubst '${SERVER_NAME} ${API_ADDRESS} ${CERT_NAME} ${CERT_KEY}' \
    < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && \ 
    nginx -g 'daemon off;'"]

To test, do:
docker network create test-network

docker build -f ./MyApp/Dockerfile --build-arg PROJECT_NAME=MyApp.API -t webapp ./MyApp

docker run -p 8080:8080 --name webapp-api --network test-network -e PROJECT_NAME=MyApp.API -e ASPNETCORE_ENVIRONMENT=Development webapp

docker build -f ./nginx-server/Dockerfile -t nginx-server ./nginx-server

Lets pass the variables by parameter:

docker run --network test-network -v $(pwd)/nginx-server/certs/localhost.crt:/etc/nginx/ssl/localhost.crt -v $(pwd)/nginx-server/certs/localhost.key:/etc/nginx/ssl/localhost.key -v $(pwd)/nginx-server/default.conf.template:/etc/nginx/conf.d/default.conf.template -e SERVER_NAME=localhost -e API_ADDRESS=webapp-api -e CERT_NAME=localhost.crt -e CERT_KEY=localhost.key -p 443:443 -p 80:80 nginx-server

----------------------------------------

Lets create the docker compose file to allow the process became easier.

To configure that, lets create a .env.test to create environment variables to specify the config for test env and we will create placeholders to be replaced by our environment variable during the execution of our container.

Create the .env.test file at the same level of MyApp or nginx-server:
SERVER_NAME=localhost
PROJECT_NAME=MyApp.API
API_ADDRESS=webapp-api
CERT_NAME=localhost.crt
CERT_KEY=localhost.key

---------------------------

Creating the docker compose file to make the process easy to use.

On the root folder:
mkdir docker.

inside the docker folder, create the file:

docker-compose.test.yml:

services:
  myapp:
    image: my-app-img
    build:
      context: ../MyApp
      dockerfile: Dockerfile
      args:
        PROJECT_NAME: MyApp.API
    ports:
      - "8080:8080"
    env_file:
      - ../.env.test
    networks:
      - rp-network

  nginx:
    container_name: myapp-webservice-rp-img
    build:
      context: ../nginx-server
      dockerfile: Dockerfile
    ports:
      - "80:80"
      - "443:443"
    env_file:
      - ../.env.test
    volumes:
      - ../nginx-server/certs/localhost.crt:/etc/nginx/ssl/localhost.crt
      - ../nginx-server/certs/localhost.key:/etc/nginx/ssl/localhost.key
      - ../nginx-server/default.conf.template:/etc/nginx/conf.d/default.conf.template
    networks:
      - rp-network

networks:
  rp-network:
    driver: bridge

It could be tested executing:
docker-compose -f docker/docker-compose.test.yml up --build

---------------------------
Dev environment:
The purpose to have a dev environment is to allow the developer create the solution without the concern about the host environment and the required packages to have the app up and running.

The objective here is to allow the code update the image developed whenever it is changed, doing that automatically.

Create the Dockerfile.dev, this docker file will be responsible only to build the image using the volumes to map the code in development be transported to image wheneve something changed.

Dockerfile.dev:

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS dev

WORKDIR /app

EXPOSE 8080

CMD dotnet watch run --project "/app/$PROJECT_NAME/" --urls "${ASPNETCORE_URLS}" --no-hot-reload

To test it, we can run:
docker build -f ./MyApp/Dockerfile.dev -t my-app-dev ./MyApp/

docker run --rm -v $(pwd)/MyApp/:/app -e PROJECT_NAME=MyApp.API -e ASPNETCORE_URLS="http://+:8080" -e ASPNETCORE_ENVIRONMENT=Development my-app-dev

After that, when the code is changed, it should be reflected on the running container.


-------------------------

Lets try to include our app with code reload behind nginx server:

docker network create test-network

docker build -f ./MyApp/Dockerfile.dev --build-arg PROJECT_NAME=MyApp.API -t webapp ./MyApp

docker run --rm --network test-network --name webapp-api -v $(pwd)/MyApp/:/app -e PROJECT_NAME=MyApp.API -e ASPNETCORE_URLS="http://+:8080" -e ASPNETCORE_ENVIRONMENT=Development my-app-dev

docker build -f ./nginx-server/Dockerfile -t nginx-server ./nginx-server

docker run --network test-network -v $(pwd)/nginx-server/certs/localhost.crt:/etc/nginx/ssl/localhost.crt -v $(pwd)/nginx-server/certs/localhost.key:/etc/nginx/ssl/localhost.key -v $(pwd)/nginx-server/default.conf.template:/etc/nginx/conf.d/default.conf.template -e SERVER_NAME=localhost -e API_ADDRESS=webapp-api -e CERT_NAME=localhost.crt -e CERT_KEY=localhost.key -p 443:443 -p 80:80 nginx-server

---------------------------------------

Lets create the docker-compose to be responsible to automatically create the development environment without all the overhead of docker cli build and run:

Create the .env.dev file at the same level of MyApp or nginx-server:
SERVER_NAME=localhost
PROJECT_NAME=MyApp.API
API_ADDRESS=webapp-api
CERT_NAME=localhost.crt
CERT_KEY=localhost.key

docker-compose.dev.yml:

services:
  myapp:
    image: my-app-img
    build:
      context: ../MyApp
      dockerfile: Dockerfile.dev
      args:
        PROJECT_NAME: MyApp.API
    env_file:
      - ../.env.dev
    volumes:
      - ../MyApp:/app
    networks:
      - rp-network

  nginx:
    container_name: myapp-webservice-rp-img
    build:
      context: ../nginx-server
      dockerfile: Dockerfile
    ports:
      - "80:80"
      - "443:443"
    env_file:
      - ../.env.dev
    volumes:
      - ../nginx-server/certs/localhost.crt:/etc/nginx/ssl/localhost.crt
      - ../nginx-server/certs/localhost.key:/etc/nginx/ssl/localhost.key
      - ../nginx-server/default.conf.template:/etc/nginx/conf.d/default.conf.template
    networks:
      - rp-network

networks:
  rp-network:
    driver: bridge