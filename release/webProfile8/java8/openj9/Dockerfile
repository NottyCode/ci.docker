
FROM adoptopenjdk/openjdk8-openj9

COPY docker-server /opt/ol/docker/

LABEL maintainer="Alasdair Nottingham" vendor="Open Liberty" url="https://openliberty.io/" github="https://github.com/OpenLiberty/ci.docker"

# Install Open Liberty
ENV LIBERTY_VERSION=18.0.0.2 \
    LIBERTY_SHA=70d21c807b6625678c2073b961b7b1cefe3d6a87

RUN apt-get update \
    && apt-get install -y --no-install-recommends unzip \
    && curl -L -s -A UA-Open-Liberty-Docker -o /tmp/wlp.zip https://repo1.maven.org/maven2/io/openliberty/openliberty-webProfile8/$LIBERTY_VERSION/openliberty-webProfile8-$LIBERTY_VERSION.zip \
    && echo "$LIBERTY_SHA  /tmp/wlp.zip" > /tmp/wlp.zip.sha1 \
    && sha1sum -c /tmp/wlp.zip.sha1 \
    && unzip -q /tmp/wlp.zip -d /opt/ol \
    && rm /tmp/wlp.zip \
    && rm /tmp/wlp.zip.sha1 \
    && apt-get remove -y unzip \
    && rm -rf /var/lib/apt/lists/*

# Set Path Shortcuts
ENV PATH=/opt/ol/wlp/bin:/opt/ol/docker/:$PATH \
    LOG_DIR=/logs \
    WLP_OUTPUT_DIR=/opt/ol/wlp/output \
    WLP_SKIP_MAXPERMSIZE=true

RUN mkdir /logs \
    && mkdir -p $WLP_OUTPUT_DIR/defaultServer \
    && ln -s $WLP_OUTPUT_DIR/defaultServer /output \
    && ln -s /opt/ol/wlp/usr/servers/defaultServer /config \
    && ln -s /logs $WLP_OUTPUT_DIR/defaultServer/logs

# Configure WebSphere Liberty
RUN /opt/ol/wlp/bin/server create \
    && rm /config/server.env \
    && rm -rf $WLP_OUTPUT_DIR/.classCache /output/workarea \
    && mkdir /config/configDropins \
    && mkdir /config/configDropins/defaults \
    && echo "<server description=\"Default Server\"><httpEndpoint id=\"defaultHttpEndpoint\" host=\"*\" /></server>" > /config/configDropins/defaults/open-default-port.xml \
    && /opt/ol/wlp/bin/server start && /opt/ol/wlp/bin/server stop && rm -rf /output/resources/security/

EXPOSE 9080 9443

ENTRYPOINT ["/opt/ol/docker/docker-server"]
CMD ["/opt/ol/wlp/bin/server", "run", "defaultServer"]
