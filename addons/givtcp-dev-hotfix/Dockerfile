# GivTCP-DEV 3.5.47 hotfix — self-contained build (no FROM givtcp.docker.scarf.sh)
#
# Built directly from the app source extracted from the exact upstream 3.5.47 image
# (givtcp.docker.scarf.sh/britkat/giv_tcp-dev:3.5.47), with the HV-stack/read.py fix
# already applied (see README.md) and requirements.txt pinned to the exact package
# versions that image ships, verified with `pip freeze` against the running container.
# This removes the build-time dependency on the scarf.sh gateway entirely (it was
# rate-limiting `FROM` pulls with 429s), while keeping every other dependency
# byte-identical to what's already running successfully.
FROM python:3.14-alpine

RUN apk add --no-cache \
    git \
    mosquitto \
    nginx \
    redis \
    tzdata \
    xsel

RUN mkdir -p /run/nginx

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt --no-cache-dir

COPY ingress.conf /etc/nginx/http.d/
RUN rm -f /etc/nginx/http.d/default.conf

COPY GivTCP/ ./GivTCP
COPY WebDashboard/ ./WebDashboard
COPY ingress/ ./ingress
COPY ingress_no_ssl.conf ingress_no_ssl.conf
COPY redis.conf redis.conf
COPY settings.json settings.json
COPY startup.py startup.py

EXPOSE 1883 3000 6379 8099

CMD ["python3", "/app/startup.py"]
