FROM nginx:1.18.0-alpine

COPY nginx.conf /etc/nginx/nginx.conf

COPY blog.example.com.conf /etc/nginx/conf.d/default.conf

COPY ssl/ /etc/nginx/ssl/

RUN rm -f /var/log/nginx/access.log /var/log/nginx/error.log

# VOLUME /var/log/nginx/ /var/cache/nginx/blog/
