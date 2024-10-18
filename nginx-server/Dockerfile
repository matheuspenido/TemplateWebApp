#Official image for NGINX
FROM nginx:alpine

#Expose ports
EXPOSE 80
EXPOSE 443

CMD ["/bin/sh", "-c", "\
    envsubst '${SERVER_NAME} ${API_ADDRESS} ${CERT_NAME} ${CERT_KEY}' \
    < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && \ 
    nginx -g 'daemon off;'"]