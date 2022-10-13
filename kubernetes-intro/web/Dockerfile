FROM nginx

COPY default.conf /etc/nginx/conf.d/default.conf 

ARG nginx_uid=1001
ARG nginx_gid=1001

RUN usermod -u $nginx_uid nginx && groupmod -g $nginx_gid nginx

