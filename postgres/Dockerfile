ARG PG_MAJOR=18
FROM postgres:${PG_MAJOR}

LABEL maintainer="suhaib"
LABEL description="PostgreSQL ${PG_MAJOR} with PostGIS and pgvector extensions"

ARG PG_MAJOR=18
ARG PGVECTOR_VERSION=0.8.1

# Install PostGIS and pgvector from source
RUN apt-get update && \
    apt-mark hold locales && \
    apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        git \
        postgresql-server-dev-${PG_MAJOR} \
        postgresql-${PG_MAJOR}-postgis-3 \
        postgresql-${PG_MAJOR}-postgis-3-scripts && \
    cd /tmp && \
    git clone --branch v${PGVECTOR_VERSION} https://github.com/pgvector/pgvector.git && \
    cd pgvector && \
    make clean && \
    make OPTFLAGS="" && \
    make install && \
    cd / && \
    rm -rf /tmp/pgvector && \
    apt-get remove -y build-essential git postgresql-server-dev-${PG_MAJOR} && \
    apt-get autoremove -y && \
    apt-mark unhold locales && \
    rm -rf /var/lib/apt/lists/*

COPY initdb-extensions.sh /docker-entrypoint-initdb.d/10-create-extensions.sh
RUN chmod +x /docker-entrypoint-initdb.d/10-create-extensions.sh

EXPOSE 5432
