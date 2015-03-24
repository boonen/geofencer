FROM ubuntu
MAINTAINER Jan Boonen <janboonen5@gmail.com>

RUN \
  apt-get update && apt-get -y --no-install-recommends install \
    curl \
    ca-certificates \
    supervisor \
    nginx-light

WORKDIR /opt
RUN \
  curl -s -o grafana_latest.tar.gz http://grafanarel.s3.amazonaws.com/grafana-1.9.1.tar.gz && \
  curl -s -o influxdb_latest_amd64.deb http://get.influxdb.org/influxdb_latest_amd64.deb && \
  mkdir grafana && \
  tar -xzf grafana_latest.tar.gz --directory grafana --strip-components=1 && \
  dpkg -i influxdb_latest_amd64.deb

ADD grafana/config.js /opt/grafana/config.js
ADD nginx/nginx.conf /etc/nginx/nginx.conf
ADD supervisor/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
ADD influxdb/config.toml /opt/influxdb/current/config.toml
ADD influxdb/types.db /opt/influxdb/current/types.db

VOLUME ["/opt/influxdb/shared/data"]

# Optionally add port 2003 for a Graphite listener
EXPOSE 80 8083 8086

CMD ["supervisord", "-n"]
