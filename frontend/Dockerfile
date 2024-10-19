# Stage 1: Build Angular Application
FROM node:20.11.1 AS build
WORKDIR /app

# Copy package.json and package-lock.json to install dependencies
COPY ./package.json ./package-lock.json ./
RUN npm install

# Copy the rest of the application code
COPY . .

# Build the Angular app for production
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine AS publish

ARG ANGULAR_PROJECT_NAME

# Copy the built Angular app from the build stage
COPY --from=build /app/dist/${ANGULAR_PROJECT_NAME}/browser /usr/share/nginx/html

# Expose port 80 for HTTP and HTTPS traffic
EXPOSE 80
EXPOSE 443

# Start replace the env variable inside the template and nginx server
CMD ["/bin/sh", "-c", "\
    envsubst '${SERVER_DOMAIN} ${BACKEND_API_URL} ${CERT_KEY} ${CERT_NAME}' \
    < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && \
    nginx -g 'daemon off;'"]