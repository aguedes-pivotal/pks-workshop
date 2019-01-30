FROM debian:stretch

LABEL maintainer="Alex Guedes <aguedes@pivotal.io>"

RUN set -x \
	&& apt-get update \
	&& apt-get install -y nginx \
	&& sed -i '/[::]/d' /etc/nginx/sites-available/default

RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]