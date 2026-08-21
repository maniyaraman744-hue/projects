FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install --no-install-recommends -y apache2 \
    && rm -rf /var/lib/apt/lists/* /var/www/html/*

COPY index.html /var/www/html/index.html

WORKDIR /var/www/html

EXPOSE 80

CMD ["apachectl", "-D", "FOREGROUND"]
