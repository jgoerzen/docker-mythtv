FROM jgoerzen/mythtv-backend
MAINTAINER John Goerzen <jgoerzen@complete.org>
RUN mv /usr/sbin/policy-rc.d.disabled /usr/sbin/policy-rc.d
RUN apt-get update && \
    apt-get -y --no-install-recommends install mysql-server && \
    apt-get -y -u dist-upgrade && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN mv /usr/sbin/policy-rc.d /usr/sbin/policy-rc.d.disabled

# MySQL MUST be running before we install mythtv-database.

RUN /etc/init.d/mysql start && \
    while [ ! -e /var/run/mysqld/mysqld.sock ]; do sleep 1; done; sleep 1 && \
    apt-get update && \
    echo "" | DEBIAN_FRONTEND=noninteractive DEBIAN_PRIORITY=critical \
      apt-get -y --no-install-recommends install mythtv-database && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    /etc/init.d/mysql stop
RUN /usr/local/bin/docker-wipelogs

EXPOSE 80 443 81
CMD ["/usr/local/bin/boot-debian-base"]

