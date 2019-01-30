FROM centos

LABEL maintainer="Alex Guedes <aguedes@pivotal.io>"

COPY nginx.repo /etc/yum.repos.d/nginx.repo

RUN set -x \
	&& yum update -y \
	&& yum install -y nginx

RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]