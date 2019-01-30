FROM  debian

RUN apt-get -y update
RUN export DEBIAN_FRONTEND=noninteractive && apt-get install -y slapd ldap-utils supervisor

ADD volume /opt/ldap
EXPOSE 389 636