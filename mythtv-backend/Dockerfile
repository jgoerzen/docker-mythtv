FROM jgoerzen/debian-base-vnc:stretch
MAINTAINER John Goerzen <jgoerzen@complete.org>
RUN mv /usr/sbin/policy-rc.d.disabled /usr/sbin/policy-rc.d
COPY setup/ /tmp/setup/
RUN /tmp/setup/setup.sh
RUN apt-get update && \
      apt-get -y --no-install-recommends install mythtv-backend xmltv-util && \
    apt-get -y -u dist-upgrade && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
COPY scripts/ /usr/local/bin/
RUN /usr/local/bin/docker-wipelogs
RUN mv /usr/sbin/policy-rc.d /usr/sbin/policy-rc.d.disabled

EXPOSE 80 443 81
VOLUME ["/var/www/html/ampache/config"]
CMD ["/usr/local/bin/boot-debian-base"]

