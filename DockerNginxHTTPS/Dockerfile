# take default nginx docker 
FROM nginx:1.17.1-alpine

# copy app to nginx/html
COPY /src/ /usr/share/nginx/html

# copy certs and key to docker
COPY /app.crt /etc/ssl/certs/app.crt
COPY /app.key /etc/ssl/private/app.key

# copy custom nginx config to docker 
COPY /default.conf /etc/nginx/conf.d/default.conf

# restart nginx in docker after config update
CMD ["/bin/sh", "-c", "nginx -g 'daemon off;'; nginx -s reload;"]