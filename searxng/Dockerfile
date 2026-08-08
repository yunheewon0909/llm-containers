FROM ghcr.io/searxng/searxng:latest

USER root

COPY settings.yml /etc/searxng/settings.yml

RUN SECRET="$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 64)" \
    && sed -i "s/__GENERATE_AT_BUILD__/${SECRET}/" /etc/searxng/settings.yml